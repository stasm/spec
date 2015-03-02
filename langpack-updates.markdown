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


Updating langpacks
------------------

Updating langpacks works in the same manner as updating regular webapps 
does.  `App.checkForUpdate()` polls the URL of the mini-manifest, then 
polls the URL in the `package_path` field in the mini-manifest.
 

Option 1. Don't do anything
---------------------------

In this scenario we leave it to the user to install a new langpack 
suitable for their current set of target apps.

When Gaia updates from 2.2 to 3.0, the user needs to go to Settings 
→ Language → Get More Languages and check if new langpack apps are 
available for Gaia 3.0.

Pros:

  - Extremely simple.

Cons:

  - Not user-friendly.

Notes:

  - After an update is applied and we know that the user has langpacks 
    installed, could we offer to open Marketplace for them to look for 
    new langpacks?


Option 2. Versions of langpack app correspond to target app versions
--------------------------------------------------------------------

In this model, we could make major+minor versions of a langpack app 
(2.2.*, 3.0.*) correspond to the versions of the target apps:


    "version": "2.2.123",
    "languages-target": {
      "app://*.gaiamobile.org/manifest.webapp": "2.2"
    }

    "version": "3.0.456",
    "languages-target": {
      "app://*.gaiamobile.org/manifest.webapp": "3.0"
    }

Cons:

  - Doesn't scale to multiple target apps included in a single 
    langpack:

        "languages-target": {
          "app://system.gaiamobile.org/manifest.webapp": "2.2",
          "app://settings.gaiamobile.org/manifest.webapp": "3.0"
        }

Notes:

  - This requires a change to the `App.checkForUpdate()` API so that 
    the platform doesn't install a newer version of a langpack if the 
    target app hasn't been yet updated.  E.g. a langpack providing 
    resources of Settings 2.2 shouldn't be updated to version 3.0.0 if 
    the Settings app is still in version 2.2.

  - Can the Marketplace expose multiple versions of langpacks depending on the 
    fxos_version query argument?

    Example:

        https://marketplace.firefox.com/api/v2/langpacks/?dev=firefoxos&fxos_version=2.2
        https://marketplace.firefox.com/api/v2/langpacks/?dev=firefoxos&fxos_version=3.0

    The langpacks listed by these API calls would be different versions 
    of the same langack apps.
