Langpack Updates
================

This document discusses the mechanics of how (and if) language packages 
should be updated when their target apps are updated.


Problem Statement
-----------------

Langpacks are webapps which provide l10n resources for specific 
versions of other apps (henceforth called *target apps*).  When 
a target app is updated the l10n resources provided by any installed 
langpack for the previous version become outdated.


Langpack versioning
-------------------

Langpacks use three versioning schemes, each serving a different 
purpose:

 - langpack version

   This is the version of the webapp that is the langpack.  It's used 
   when a new version of the langpack is available and the update is 
   pushed to devices.

   Defined in the top-level `version` field of the langpack's manifest 
   as a string.

   Example:

        "version": "1.0.1"

 - target apps' versions

   These are the versions of apps that the langpack provides the l10n 
   resources for.  E.g. a langpack can target Settings 2.2.

   Defined in `languages-target.${appId}` as a string.

   Example:

        "languages-target": {
          "app://*.gaiamobile.org/manifest.webapp": "2.2"
        }

 - revision of the language resources

   This is the revision of the language resources included in the 
   langpack.  It is used for language negotiation by l10n.js.  E.g. the 
   target app may already contain a localization which is newer in 
   revision than the one provided by a langpack.

   Defined in `languages-provided.${lang}.revision` as an integer.

   Example:

        "languages-provided": {
          "de": {
            "name": "Deutsch",
            "revision": 201411051234,
            "apps": {
              "app://calendar.gaiamobile.org/manifest.webapp": "/de/calendar",
              "app://email.gaiamobile.org/manifest.webapp": "/de/email"
            }
          }
        }
