Webapp Langpacks Spec
=====================

This document outlines the design of the language package support for webapps 
and instant webapps.  The design revolves around special webapps 
which can register themselves to provide localization resources for multiple 
languages at once.  In 2.2, we want individual langpacks to support multiple 
apps as well, although this could be deprecated in the future as we move 
towards a mroe modular process of releasing Gaia apps.

There are two independent parts of the language package system:

 - language negotiation, which requires the localization library to be able to 
   query the platform for languages available for the current app, both 
   bundled with the app and installed by the user as langpacks,

 - localization resources IO, which fetches the resources either from the 
   current app or a user-installed langpack.

This proposal relies on new native API methods which provide a way to query the 
platform for any additional languages available for the current app as well as 
to fetch resources provided by other apps for the current app.

(In the following examples we use the Email app to illustrate the design.)


App's HTML
----------

In HTML, language resources are linked like this:

    <link 
      rel="localization"
      href="/locales/email.{locale}.properties">

The information about the available and the default languages is stored in 
`meta` elements (built on buildtime):

    <meta name="appVersion" content="2.2">
    <meta name="defaultLanguage" content="en-US">
    <meta
      name="availableLanguages"
      content="en-US:201411142303, de:201411051234">


mozApps.getAdditionalLanguages()
--------------------------------

When a document is opened the l10n.js library is tasked with negotiating the 
correct language to display the UI in to the user.  It uses 
`navigator.languages` to get the list of user's preferred langs, the 
`mozApps.getAdditionalLanguages` API and the `meta` elements to get the list of 
languages available for the current app, as well as the default language.  The 
result of the negotiation is a list of supported locales.

The information about extra languages available for the current app is stored 
in a Gecko pref and can be accessed via the `mozApps` API.

    mozApps.getAdditionalLanguages(manifestURI);

This method returns a `DOMRequest` which resolves to an object with installed 
additional languages available for the current apps as keys:

    {
      "de": [
        {
          "target": "2.2",
          "revision": 201411051234
        }
      ]
    }

  1. The signature of this method could also accept a second argument to 
     specify the target version:

         mozApps.getAdditionalLanguages(manifestURI, '2.2');

     …which would return:

         {
           "de": [
             {
               "revision": 201411051234
             }
           ]
         }

The information about the bundled and the default languages is stored in `meta` 
elements and can be accessed synchronously.

An example code in l10n.js using this API could look like this:

    navigator.mozApps.getAdditionalLanguages(manifestURI).then(
      function(extraLangs) {
        /*
         * Use <meta name="availableLanguages"> to get the list of 
         * languages bundled with the app. Append extraLangs to it.   
         * Negotiate supported languages using navigator.languages, the 
         * list of all available languages including those coming from 
         * langpacks and the value of <meta name="defaultLanguage">.
         */
      });

When a langpack is installed or removed, an `additionallanguageschange` event is 
dispatched on the `document` with the list of extra languages provided as 
a `languages` event detail.


Resource IO
-----------

Once the l10n.js library has negotiated the list of supported languages, it can 
resolve the URL templates from `link` elements and fetch the resources in the 
correct language.

First, l10n.js resolves the URI of the resource to fetch:

    /locales/email.{locale}.properties

…becomes:

    /locales/email.de.properties

Since l10n.js also knows which languages come from the app and which come from 
langpacks (from the step before), it can decide which method of IO to use:

  - For languages bundled with the app, the resources are fetched using regular 
    XHR requests.

  - For languages provided by langpacks, the resources are fetched via the 
    `mozApps.getLocalizationResource()` native API (see below).

In case of the `/locales/email.de.properties` resource, a regular XHR is used.


Langpack App Manifest
---------------------

Langpacks are webapp with a special 'langpack' role.  They're hidden from the 
Homescreen and can't be launched.  They act as bundles of resources which 
override requests to certain origins.  A single langpack can contain resources 
for multiple apps and multiple languages.  An example of a langpack manifest 
with the origin `my-langpack.gaiamobile.org` looks like this:

    "name": "Gaia Langpack for German and Polish",
    "description": "Support for additional languages: German and Polish",
    "locales": {
      "de": {
        "name": "Sprachpaket für Gaia: Deutsch und Polnisch"
      },
      "pl": {
        "name": "Paczka językowa dla Gai: niemiecki i polski"
      },
    },
    "icons": {
      "128": "/icon.png"
    },
    "role": "langpack",
    "version": "1.0.3",
    "languages-target": {
      "app://*.gaiamobile.org/manifest.webapp": "2.2"
    },
    "languages-provided": {
      "de": {
        "name": "Deutsch",
        "revision": 201411051234,
        "apps": {
          "app://calendar.gaiamobile.org/manifest.webapp": "/de/calendar",
          "app://email.gaiamobile.org/manifest.webapp": "/de/email"
        }
      },
      "pl": {
        "name": "Polski",
        "revision": 201410292345,
        "apps": {
          "app://calendar.gaiamobile.org/manifest.webapp": "/pl/calendar",
          "app://email.gaiamobile.org/manifest.webapp": "/pl/email"
        }
      }
    }

The `version` field is the version of the langpack app, for the purposes of 
pushing updates via the Marketplace.

Two special fields are unique to the `langpack` role: `languages-target` and 
`languages-provided`.

The former, `languages-target`, describes the target for the langpack.  For 
now, only a special `app://*.gaiamobile.org/manifest.webapp` wildcard would be 
supported denoting Gaia system-wide langpacks and the version of the system.  
This field is used by the Marketplace to build the list of langpacks that are 
relevant to the user's OS.

The latter field, `languages-provided` field defines languages and their 
revisions for app origins.  This information is stored in device's chrome's 
IndexedDB and used for resource IO and language negotiation.  The value of the 
`revision` field must be an integer to allow comparison between language 
revisions with the `<` and `>` operators in JavaScript.

  1. Can we skip the protocol part of the manifest URI in the keys of the 
     `languages_provided` object?

  2. Should we explicitly list all l10n resources in the manifest instead of 
     defining the entry point dir for them?


Langpack Installation
---------------------

Chrome's IndexedDB is used to store the information about the new additional 
languages and the contents of the additional resources.  When a new langpack is 
installed, the information about the new additional languages for each app is 
saved in the database:

    languages,app://email.gaiamobile.org/manifest.webapp: {
      "de": {
        "target": "2.2",
        "revision": 201411051234
      },
      "pl": {
        "target": "2.2",
        "revision": 201410292345
      },
    }

For each app, each resource form the langpack (from the corresponding 
`basepath` directory) is also saved in the DB, keyed by the app it belongs to, 
language code, app version and the resource path.

    resource,app://email.gaiamobile.org/manifest.webapp,de,2.2,locales/email.de.properties: "foo=Foo\nbar=Bar"

The `additionallanguageschange` event is dispatched on the `document` with the 
list of all languages now available in the chrome.

  1. This requires a way to get the list of all files in the langpacks 
     implicitly.  See question #2 in the previous section about providing the 
     list explicitly in langpack's manifest.

  2. Is it possible to run the installation procedure during buildtime or the 
     first launch of the system in order to install any pre-built langpacks?  
     This would allow us to dirtibute all locales, even the bundled ones, as 
     pre-installed langpacks, which would in turn make them receive updates 
     from the Marketplace.

  3. On System Update, should the database be cleared (on the assumption that 
     old langpacks are not good to show with newer apps)?  If so, maybe we 
     don't need to track the version information at all?

  4. Are installed langpacks stored on the device as zips even after their 
     contents are inserted to the chrome's database?  Are they instantly 
     deleted?  Does the `mozApps` object know about installed langpacks as it 
     does about all other apps?  If not, how can the Marketplace push updates 
     to the installed langpacks?
    

Language Negotiation and Resource IO, with Langpacks
----------------------------------------------------

`mozApps.getAdditionalLanguages()` for a given app now returns the newer 
revision of `de`, as well as adds `pl` to the list of available languages.  The 
l10n.js library can negotiate the UI language using user's preferred languages 
stored in `navigator.languages`.

Once the negotiation has completed, specific resources can be fetched.  Again, 
`{locale}` is replaced by l10n.js with the result of the negotiation.

    /locales/email.{locale}.properties

…becomes:

    /locales/email.pl.properties

Since 'pl' is a langpack language (one of the languages returned by 
`mozApps.getAvailableLanguages()`), l10n.js will use the native API to get the 
resource:

    mozApps.getLocalizationResource(
      'app://email.gaiamobile.org/manifest.webapp',
      'pl',
      '2.2',
      '/locales/email.pl.properties');


mozApps.getLocalizationResource()
---------------------------------

This API provides a way for userland code to request resources provided for 
a specific app by some other app.

    mozApps.getLocalizationResource(
      manifestURI, languageCode, appVersion, resourcePath);

It is not a method of the `App` object to allow instant webapps (which are not 
installed) to benefit from langpacks.

The `getLocalizationResource` API queries the chrome IndexedDB for 

    'resource,' + manifestURI + ',' + languageCode + ',' + appVersion + ',' + resourcePath

…and returns the contents stored in the DB:

    resource,app://email.gaiamobile.org/manifest.webapp,de,2.2,locales/email.de.properties: "foo=Foo\nbar=Bar"

  1. In which form do we store resource contents in the DB?  Raw string with 
     the contents of the .properties file?  Parsed JSON?  ZIP of the langpack?


Localizing App Names
--------------------

Each langpack can provide a special file next to the target app's locales/ 
directory with localization resources:  `manifest.{locale}.properties`.  This 
file has a `name` and `description` strings and is used to provide localizations 
of the app names on the Homescreen, on the Rocket bar, in the Haida Cards view, 
in Task Manager and in the Settings > App Permissions panel.

For Homescreen, the code in [Gaia Grid][] is responsible for returning the 
localized name of the app.  For other places, a helper class called the 
[ManifestHelper][] is responsible for returning the localized app names. 

Both can use the `mozApps.getLocalizationResource()` API to fetch the 
`manifest.{locale}.properties` file if the translation has not been found in the 
app's manifest.

    mozApps.getLocalizationResource(
      'app://email.gaiamobile.org/manifest.webapp',
      'pl',
      '2.2',
      '/manifest.pl.properties');

  1. How can this code know whether to fetch from the app's 
     `manifest.webapp` or to use `mozApps.getLocalizationResource()`?  It could 
     first try the former and resort to the latter.

  2. This assumes `mozApps.getLocalizationResource()` requests can span 
     different domain.  How can we avoid this?

[Gaia grid]: https://github.com/mozilla-b2g/gaia/blob/a6c295a7a6bddf5bcd42d970725300c4c30760b8/shared/elements/gaia_grid/js/items/mozapp.js#L169-L180
[ManifestHelper]: https://github.com/mozilla-b2g/gaia/blob/a6c295a7a6bddf5bcd42d970725300c4c30760b8/shared/js/manifest_helper.js


Langpack Uninstallation
-----------------------

TBD.


App Upgrade & Uninstallation
----------------------------

TBD.


Requirements
------------

Platform Team:

  - Write support for Langpacks API
    - `mozApps.sgetAdditionalLanguages`
    - `mozApps.getLocalizationResource`
  - Write custom logic for language pack installation.
  - Write logic for storing and retrieving resources in chrome’s IndexedDB.
  - Write logic for firing `additionallanguageschange` event.


Marketplace Team:

  - Add support for ‘language pack’ addon type
  - Create a landing page for the 'language pack' addon type
  - Provide REST API for retrieving list of available language packs


UX Team:

  - Design interaction flow for discovering, installing and selecting 
    additional languages
    - The minimal idea is to add a link in the Settings > Languages panel to 
      the Marketplace's landing page for langpacks.
    - Make sure the user returns to the Settings > Language panel after 
      installing a langpack.  Should the system prompt the user to enable the 
      newly installed locale right away?
    - Update the list of installed languages in the Language panel.
  - Open questions:
    - Installing keyboard layouts and IMEs?
    - Installing dictionary files?


Gaia Team:

  - Settings: Extend the Language Panel to support the flow designed by UX.
    - Add the link to Marketplace in the Settings > Languages panel.
    - React to the `additionallanguageschange` event in 
      `shared/js/language_list.js`.
  - Homescreen:  Add support for retrieving localized names of applications 
    from langpacks via `mozApps.getLocalizationResource`.
  - Shared:  Use `mozApps.getLocalizationResource` in `shared/js/manifest_helper.js` in 
    order to provide localized app names for the Cards view, the Task Manager, 
    the Rocketbar and the Settings > App Permissions panel.
  - Input method: keyboard layout should fall back to `GAIA_DEFAULT_LOCALE` if 
    no better is available for the newly installed langpack.


L10n Team:

  - Runtime L10n.js changes:
    - Use `mozApps.getAdditionalLanguages()` in the bootstrapping code when 
      looking for the information about the available languages.  
    - Remove the codepath which fetches the manifest to get this information;  
      from now on we will require `meta` elements to provide it inlined into 
      the HTML.
    - Create logic which negotiates languages and their revision given the list 
      of languages bundled in the app and the list returned by 
      `mozApps.getAdditionalLanguages()`.
    - Mark which negotiated languages come from the app and which come from 
      langpacks.
    - Use regular XHR or `mozAppgetResource()` accordingly.
  - Buildtime L10n.js changes:
    - Inline information about languages and their revisions in meta elements.  
      (Right now we only inline language codes without revisions).


Rel Eng / Localization infrastructure:

  - We need a way to build langpacks from specific l10n changesets (singed off 
    on by the localizers and approved by the l10n team) and upload them to the 
    Marketplace.  Spec TBD.

