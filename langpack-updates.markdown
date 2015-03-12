Langpack Updates
================

This document discusses the mechanics of how (and if) language packages 
should be updated when their target apps are updated.

The proposals listed below are not necessarily mutually exclusive.  
They can complement each other.


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

Langpacks can be updated for two reasons:

  1. to change the revision of the provided languages (and update the 
     localization resources within the langpack).
  2. to change the version of the target apps,

Updates of type #1 can happen multiple times when localizers fix 
translation mistakes.  The langpack is updated to provide a newer, 
better translation for the same version of the target app.

Updates of type #2 can happen when the target app is updated.  
Currently, this is not impemented and is in fact what this document 
aims to describe.  Updates of type #2 can entail updates of type #1.


Langpack versions co-existance
------------------------------

Langpacks targeting different versions of the target app need to be 
able to co-exist in the Marketplace.  E.g. it should be possible for 
the langpack for Settings 2.2 and the langpack for Settings 3.0 to be 
both available for download.  They can be different versions of the 
same app, or two different apps.

Furthermore, when the langpack for Settings 3.0 is released, it should 
not be pushed to users with the langpack for Settings 2.2 until they 
update their Settings app to version 3.0.

Notes:

  - Can the Marketplace expose multiple versions of langpacks depending 
    on the `fxos_version` query argument?

    Example:

        https://marketplace.firefox.com/api/v2/langpacks/?dev=firefoxos&fxos_version=2.2
        https://marketplace.firefox.com/api/v2/langpacks/?dev=firefoxos&fxos_version=3.0

    Can the langpacks listed by these API calls be different versions 
    of the same langack apps, or do they need to be different apps?
 

Proposal 1. Don't do anything
-----------------------------

In this scenario we leave it to the user to install a new langpack 
suitable for their current set of target apps.

When Gaia updates from 2.2 to 3.0, the user needs to go to Settings 
→ Language → Get More Languages and check if new langpack apps are 
available for Gaia 3.0.

Pros:

  - Extremely simple.
  - Langpacks for different target apps need to be seperate apps to 
    avoid pushing updates to them.

Cons:

  - Not user-friendly.

Notes:

  - After an update is applied and we know that the user has langpacks 
    installed, could we offer to open Marketplace for them to look for 
    new langpacks?


Proposal 2.  Push only `PATCHLEVEL` updates
-------------------------------------------

The langpack version uses the `MAJOR.MINOR.PATCHLEVEL` syntax.  The 
target app version uses the `MAJOR.MINOR` syntax.  We could establish 
a convention by which:

  - `MAJOR.MINOR` in the langpack version is only changed when one or 
    more of the target apps changes,
  - `PATCHLEVEL` in the langpack version is only changed when provided 
    languages' revisions change.

`MAJOR` and `MINOR` in the langpack version could correspond to their 
counterparts in the target app versions alhough this wouldn't scale to 
langpacks providing language resource for multiple apps in multiple 
versions.

With this extention, Proposal #1 could be made to work with versions of 
the same langpack app.

Notes:

  - Is it possible to offer multiple versions of one app on the 
    Marketplace or is the newest one always the only one available?


Proposal 3. Listen to Webapps being updated and notify the user
-------------------------------------------------------------

This is similar to Option 1 in that it makes the user responsible for 
installing a new langpack app.  It assists the user by listening to 
Webapps.jsm broadcasts about apps being updated and compares the 
manifest URLs of updated app to the URLs stored in Langpacks.jsm's 
`_data` store.  When an app for which the user has installed a langpack 
is updated we can offer to open the Marketplace listing with langpacks 
for this app and the new version.


Proposal 4. Versions of langpack app correspond to target app versions
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

