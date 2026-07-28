# RSP-10349 — Root Cause & Solution

## Root Cause

`CAliANServerC::Deserialize4()` (AliANServerC.cpp:3820) prefers a **pre-serialized annotation server cached in the GSPS private attribute (3711,10b1)** over converting the standard GSPS content.

When a study moves between archive contexts (GATI → CHIR → HULL prefetch), the image UIDs are **coerced**. The standard GSPS reference sequences are rewritten correctly, but the cached private blob still references the **original** UIDs.

That stale blob deserializes without error, so `bResult == true` and the function returns — never reaching the `ConvertGSPS()` fallback. The annotations load successfully but resolve to images that do not exist in the current study, so **nothing is displayed**.

The one existing staleness guard, `IsGSPSNewerThanOriginalServer()`, does not catch this: UID coercion does not update the GSPS presentation creation date/time, so `bIsNewer == false` and the stale cache passes the check.

This is confirmed by the customer workaround in the ticket — deleting tag (3711,10b1) forces the `ConvertGSPS()` path and the annotations render correctly.

## Solution

Two changes in `Deserialize4()`:

**1. Validate the deserialized cache against the GSPS, fall back when it doesn't match.**
Add `InternalServerMatchesGSPSReferences()`: for each image referenced by the GSPS (`pGSPS->GetScopes()`), ask every deserialized scoped collection whether it applies to that image, reusing the server's own scope matching — `CScopedCollectionC::IncludeScope()` (the same test `UpdateLinksForScopedCollection()` uses) with `CAliIPPScopeC::IsEqual(scope, 2)` as an image-level fallback. If **not a single** GSPS image matches **any** collection, the cache is stale: call `DeleteAllElements()` and re-run `ConvertGSPS()`.

**2. Fall back instead of hard-failing when the blob won't deserialize.**
Both `Deserialize()` failure paths currently just `REPORT_ERROR` and return false, losing all annotations. They now fall through to `ConvertGSPS()` as well.

Guards keep healthy studies on the existing path unchanged: an empty scoped-collection map, a failed or empty `GetScopes()`, or any partial match all accept the cache. A false "stale" verdict is benign — `ConvertGSPS()` rebuilds the annotations from the GSPS graphic content, which is the source of truth.

| Scenario | Old | New |
|---|---|---|
| No blob / GSPS newer than blob | ConvertGSPS | unchanged |
| Blob present, references consistent | use blob | unchanged |
| **Blob present, references stale** | **blob used → blank** | **fallback to ConvertGSPS** |
| **Blob fails to deserialize** | **hard fail → blank** | **fallback to ConvertGSPS** |

**Also recommended:** any component that coerces UIDs (archive morph, prefetch) should strip or rewrite (3711,10b1) — a cache inconsistent with its own document is a data-integrity problem in itself. And `Deserialize3()` (line 3483, the v1 importer path) has the identical structure and needs the same treatment if v1-era GSPS objects still reach modified-UID scenarios at customer sites.

Full implementation with all APIs verified against the codebase is in the `RSP-10349-final-code.cpp` file I sent in the previous message.
