
//==============================================================================
//  PARAMETERS      ipGSPS: the GSPS this server was deserialized from
//  RETURN          true when the deserialized annotations are usable for
//                  this GSPS, false when they are stale
//  PRECONDITIONS   valid input GSPS; server content was deserialized from
//                  the GSPS private attribute
//  POSTCONDITIONS  none (read only)
//  NOTES:          RSP-10349 - when a study is transferred between archive
//                  contexts the image UIDs can be coerced: the standard
//                  GSPS reference sequences are rewritten but the
//                  annotation server serialized in the private attribute
//                  still references the original UIDs. Such a server
//                  deserializes successfully, yet none of its annotations
//                  resolve to an image of the current study, so nothing is
//                  displayed.
//                  The check reuses the server's own scope matching: for
//                  each image referenced by the GSPS, ask every
//                  deserialized scoped collection whether it applies to
//                  that image (IncludeScope - same test
//                  UpdateLinksForScopedCollection uses to connect output
//                  collections - complemented by the image-level equality
//                  FindScopedCollection uses). Stale only when not a
//                  single GSPS image matches a single collection, so
//                  healthy studies keep the exact same behaviour.
//==============================================================================
bool
CAliANServerC::InternalServerMatchesGSPSReferences( const CAliDSGSPSC2* ipGSPS )
//==============================================================================
{
    if ( m_ScopedCollectionMap.empty() )
    {
        //--------------------------------------------------
        // deserialized server holds no annotations -
        // nothing to cross check (and nothing that could
        // be displayed); keep the previous behaviour
        //--------------------------------------------------
        return true;
    }

    //--------------------------------------------------
    // the image references of the GSPS itself - the
    // standard reference sequences, which UID coercion
    // rewrites correctly
    // (elements are non owning - ConvertGSPS does not
    // free them either)
    //--------------------------------------------------
    ImageUIDToScopeMap scopesMap;

    if ( !ipGSPS->GetScopes( scopesMap ) || scopesMap.empty() )
    {
        //--------------------------------------------------
        // cannot verify - accept the deserialized server
        // so these paths behave exactly as before
        //--------------------------------------------------
        return true;
    }

    for ( ImageUIDToScopeMap::const_iterator cIter = scopesMap.begin();
          cIter != scopesMap.end();
          ++cIter )
    {
        CAliIPPScopeElementC* pReferenceScope = cIter->second;

        if ( pReferenceScope == 0 )
        {
            continue;
        }

        unique_ptr< CAliIPPScopeCollectionC > pImageScope(
            ConvertScopeElementToScopeCollection( pReferenceScope ) );

        if ( pImageScope.get() == 0 )
        {
            continue;
        }

        //--------------------------------------------------
        // does any deserialized scoped collection apply to
        // this image of the GSPS?
        //--------------------------------------------------
        ScopedCollectionMapIterator it = m_ScopedCollectionMap.begin();

        while ( it != m_ScopedCollectionMap.end() )
        {
            CScopedCollectionC* pScopedCollection = (*it).second;

            if ( pScopedCollection != 0 )
            {
                CAliIPPScopeC* pCollectionScope = pScopedCollection->GetScope();

                if ( pScopedCollection->IncludeScope( pImageScope.get() ) ||
                     ( ( pCollectionScope != 0 ) &&
                       pCollectionScope->IsEqual( pImageScope.get(), 2 ) ) )
                {
                    return true;
                }
            }

            ++it;
        }
    }

    return false;
}


//==============================================================================
// 3) AliANServerC.cpp - full replacement of Deserialize4() (lines 3819-3903)
//==============================================================================

//------------------------------------------------------------------------------
bool CAliANServerC::Deserialize4( const CAliIPGSPS2* cipIPGSPS )
{
    bool bResult = false;

    if ( cipIPGSPS != NULL )
    {
        const CAliDSGSPSC2* pGSPS = cipIPGSPS->GetGSPS();

        CAliANGSPSImporter2 gspsImporter( *pGSPS );

        //
        // check if supported version before proceeding
        //
        bResult = gspsImporter.CheckIfSupported();

        if ( bResult )
        {
            //
            // RSP-10349: an annotation server blob found in a private attribute
            // may be stale - when the study moves between archive contexts the
            // image UIDs are coerced, the standard GSPS references are
            // rewritten, but the serialized server still references the
            // original UIDs. Such a blob deserializes fine yet none of its
            // annotations resolve to an image of this study, so nothing is
            // displayed. Whenever the blob is unusable or inconsistent we now
            // fall back to converting the standard GSPS content instead of
            // failing / silently showing nothing.
            //
            bool bUseDeserializedServer = false;

            //
            // has this GSPS been preprocessed by importer?
            //
            bool bHasImporterPreprocess = gspsImporter.HasAnnotationServer( ALI_GSPS_ANNOTATION_SERVER_READ );

            if ( bHasImporterPreprocess )
            {
                vector< char > importerGSPSBuffer;
                bResult = gspsImporter.GetAnnotationServer( ALI_GSPS_ANNOTATION_SERVER_READ, importerGSPSBuffer );

                if ( bResult )
                {
                    int nSize = static_cast<int>( importerGSPSBuffer.size() );
                    bUseDeserializedServer = Deserialize( &importerGSPSBuffer[ 0 ], &nSize );

                    if ( !bUseDeserializedServer )
                    {
                        REPORT_ERROR( "Failed to deserialize annotation server from GSPS importer preprocess - falling back to GSPS conversion" );
                    }
                }
            }
            else
            {
                //
                // check to see if GSPS contains our own annotation server serialized in a private attribute
                // but, only use it if the GSPS itself is not newer than when the server was originally serialized
                //

                bool bIsNewer = false;
                bResult = gspsImporter.IsGSPSNewerThanOriginalServer( bIsNewer );

                if ( bResult )
                {
                    bool bUseInternalServer = !bIsNewer &&
                        gspsImporter.HasAnnotationServer( ALI_SERIALIZED_ANNOTATIONS_READ );

                    if ( bUseInternalServer )
                    {
                        int nSize = gspsImporter.GetAnnotationServerSize( ALI_SERIALIZED_ANNOTATIONS_READ );

                        vector < char > buffer( nSize );

                        bResult = gspsImporter.GetAnnotationServer( ALI_SERIALIZED_ANNOTATIONS_READ, buffer );

                        if ( bResult )
                        {
                            bUseDeserializedServer = Deserialize( &buffer[ 0 ], &nSize );

                            if ( !bUseDeserializedServer )
                            {
                                REPORT_ERROR( "Failed to deserialize annotation server from GSPS private attribute - falling back to GSPS conversion" );
                            }
                        }
                    }
                }
            }

            if ( bResult )
            {
                //
                // RSP-10349: cross-check the deserialized annotations against
                // the image references of the GSPS itself. A mismatch means
                // the GSPS was modified after the server was serialized
                // (e.g. UID coercion) - discard the stale annotations and
                // re-convert from the standard GSPS content.
                //
                if ( bUseDeserializedServer && !InternalServerMatchesGSPSReferences( pGSPS ) )
                {
                    REPORT_ERROR( "Serialized annotation server does not match GSPS image references (GSPS modified after serialization, e.g. UID coercion) - falling back to GSPS conversion" );

                    DeleteAllElements();

                    bUseDeserializedServer = false;
                }

                //
                // no usable serialized server (absent, newer GSPS, failed to
                // deserialize, or stale) - convert the standard GSPS content
                //
                if ( !bUseDeserializedServer )
                {
                    bResult = ConvertGSPS( pGSPS, gspsImporter );
                }
            }
        }
    }

    return bResult;
}


