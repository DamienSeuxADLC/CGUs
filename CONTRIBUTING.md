First off, thanks for taking the time to contribute! 🎉👍

# Adding a service

## Declaring a new service

Before all you have to choose the [service name and the service ID](#service-name-and-service-id).

Then, in the folder `services`, create a JSON file with the id of the service you want to add, with the following structure:

```json
{
  "name": "<the public name of the service>",
  "documents": {
    "<document type>": {
      "fetch": "<the URL where the document can be found>",
      "filter": "<an array of service specific filter function names>",
      "remove": "<a CSS selector, a range selector or an array of selectors that target the noise parts of the document that has to be removed. Useful to remove parts that are inside the selected parts>",
      "select": "<a CSS selector, a range selector or an array of selectors that target the meaningful parts of the document, excluding elements such as headers, footers and navigation>",
    }
  }
}
```

You can find examples in the `services` folder.

### Service name and service ID

- The service name is presented to end users. It should reflect as closely as possible the official service name, so that it can be identified easily. This name is currently exposed as the `name` property in the service declaration.  [See Practical guidelines to find the service name](https://github.com/ambanum/CGUs/wiki/Practical-guidelines#practical-guidelines-to-find-the-service-name).

- The service ID is used internally and exposed for analysis. It should be easy to handle with scripts and other tools.
In addition to the following constraints, see also the [practical guidelines to derive the ID from the service name](https://github.com/ambanum/CGUs/wiki/Practical-guidelines#practical-guidelines-to-derive-the-id-from-the-service-name).

#### Constraints for the ID

As long as this ID is stored in the filesystem:

- No `/` for UNIX.
- No `\` for Windows.
- No `:` for APFS and HFS.
- No case-sensitive duplicates to support case-insensitive filesystems.
- No more than 255 characters to support transfer over [FAT32](https://en.wikipedia.org/wiki/File_Allocation_Table#FAT32).

UTF, spaces and capitals are all supported, even on case-insensitive filesystems.

#### However

- UTF in filenames can be [a (fixable) problem with Git and HFS+](https://stackoverflow.com/questions/5581857/git-and-the-umlaut-problem-on-mac-os-x).
- UTF in filenames is by default quoted in Git, leading for example `été.txt` to be displayed as `"\303\251t\303\251.txt"`.
- Most online services align their brand name with their domain name. Even though UTF is now officially supported in domain names, support is limited and most services, even non-Western, have an official ASCII transliteration used at least in their domain name (e.g. “qq” by Tencent, “rzd.ru” for “РЖД”, “yahoo” for “Yahoo!”).
- We currently use GitHub as a GUI, so the service ID is presented to the user instead of the service name. The name is used in email notifications.

#### Thus

1. The service name should be the one used by the service itself, no matter the alphabet.
  - _Example: `туту.ру`_.
2. We don't support non-ASCII characters in service IDs, at least as long as the database is Git and the filesystem, in order to minimise risk. Service IDs are derived from the service name through normalization into ASCII.
  - _Example: `туту.ру` → `tutu.ru`_.
  - _Example: `historielærer.dk` → `historielaerer.dk`_.
  - _Example: `RTÉ` → `RTE`_.
3. We support punctuation, except characters that have meaning at filesystem level (`:`, `/`, `\`). These are replaced with a dash (`-`).
  - _Example: `Yahoo!` → `Yahoo!`_.
  - _Example: `Last.fm` → `Last.fm`_.
  - _Example: `re:start` → `re-start`_.
  - _Example: `we://` → `we---`_.
4. We support capitals. Casing is expected to reflect the official service name casing.
  - _Example: `hi5` → `hi5`_.
  - _Example: `DeviantArt` → `DeviantArt`_.
  - _Example: `LINE` → `LINE`_.
5. We support spaces. Spaces are expected to reflect the official service name spacing.
  - _Example: `App Store` → `App Store`_.
  - _Example: `DeviantArt` → `DeviantArt`_.
6. We prefix the service name by the provider name when self-references are ambiguous, separated by a space. For example, Facebook refers to their Self-serve Ads service simply as “Ads”, which we cannot use in a shared database. We thus call the service “Facebook Ads”.
  - _Example: `Ads` (by Facebook) → `Facebook Ads`_.
  - _Example: `Analytics` (by Google) → `Google Analytics`_.
  - _Example: `Firebase` (by Google) → `Firebase`_.
  - _Example: `App Store` (by Apple) → `App Store`_.

##### Combined examples

- `C'est qui le patron ?!`
- `Google Play`
- `Last.fm`
- `DeviantArt`
- `hi5`
- `Pepper&Carrot`

### Document type

You can find the list of allowed values for the `<document type>` key in [`/src/app/types.json`](./src/app/types.json).

The types might not always match the exact name given by the service provider. For example, some providers might call their document “Terms and Conditions” or “Terms of Use” instead of “Terms of Service”. The document type does not have to match the exact name, it only has to match the _commitment_ that is taken.

> This model enables comparing documents across providers as long as they cover similar commitments, notwithstanding their given name.

In order to guide usage and disambiguate synonyms, we characterise each document type along three dimensions of the commitment that is being taken in it:

1. The `writer` of the document;
2. the targeted `audience`;
3. the `object` of the commitment.

A document type thus looks like:

```json
{
  …
  "Privacy Policy": {
    "commitment": {
      "writer": "service provider",
      "audience": "end user",
      "object": "personal data"
    }
  },
  …
}
```

#### Defining a new document type

If the document you want to add matches no existing document type, you can create a new one in the same pull request in which you declare the service that would use it.

If you're in doubt, please list the potential synonyms you have considered.

### Filters

Some documents will need additional filtering beyond simple element deletion and selection to remove noise (changes in textual content that are not meaningful to the terms of services).

These filters are declared as JavaScript functions that modify the downloaded web page through the [DOM API](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model).

Filters take the document and the document declaration as parameters and are:

- **in-place**: they modify the document structure and content directly;
- **idempotent**: they should return the same document structure and content even if run repeatedly on their own result.

Filters are loaded automatically from files named after the service they operate on. For example, filters for the Meetup service, which is declared in `services/Meetup.json`, are loaded from `services/Meetup.filters.js`.

The generic function signature for a filter is:

```js
export [async] function filterName(document [, documentDeclaration])
```

Each filter is exposed as a named function export that takes a `document` parameter and behaves like the `document` object in a browser DOM. These functions can be `async`, but they will still run sequentially.

> The `document` parameter is actually a [JSDOM](https://github.com/jsdom/jsdom) document instance.

The second parameter contains the whole document declaration. It can be useful if you need to access the defined document URL or selector inside your filter.

For example, you can scope the selection in your filter with the `selector` defined in the document declaration:

```js
export function removeImages(document, { select: selector }) {
  const images = document.querySelectorAll(`${selector} img`);
  images.forEach(el => el.remove());
}

```

You can find examples of filters in [`/services/*.filters.json`](./services) files.

You can also learn more about [usual noise](https://github.com/ambanum/CGUs/wiki/Usual-noise).

### Selectors

A selector allows to target a part of the document. It can be of two types, a [**css selector**](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors) or a **range selector**.

A range selector is defined with a _start_ and an _end_ css selector. It is also necessary to define if the range starts before or after the element targeted by the _start_ css selector and to define if it ends
before or after the element targeted by the _end_ css selector. To do that, four keys are available to properly describe a range selector: `startBefore`, `startAfter`, `endBefore` and `endAfter`.

Thus, a range selector looks like this:

```json
{
  "start[Before|After]": "<css selector>",
  "end[Before|After]": "<css selector>"
}
```

For example, the following selector will select the content between the element targeted by the css selector `#privacy-eea`, including it, and the element targeted by the css selector `#privacy-row`, excluding it:

```json
{
  "startBefore": "#privacy-eea",
  "endBefore": "#privacy-row"
}
```

### Selecting jurisdiction and language

For now, when multiple versions coexist, terms are only tracked in their English version and for the European (EEA) jurisdiction.


## Test your declaration

Test the declaration by running the following command with the service ID:

```
npm run validate $service_id
```

> The service ID is the case sensitive name of the service declaration file without the extension. For example, for `Twitter.json`, the service ID is `Twitter`.

Since this operation fetches documents and could be long, you can also validate the declaration structure only:

```
npm run validate:schema $service_id
```

If you change filters or selectors and want to refilter documents from snapshots and regenerate versions only:

```
npm run refilter [$service_id]
```
