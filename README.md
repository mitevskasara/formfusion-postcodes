# @formfusion/postcodes

Set of validation rules for worldwide postal codes.

A zero-dependency lookup table of **244 country-specific entries** — 194 regex patterns plus 50 `null` placeholders where no rule is known — for validating postal and ZIP codes. Every pattern works directly as an HTML [`pattern`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/pattern) attribute value, so you can use it with plain HTML, React, FormFusion, or `new RegExp()`.

## Why

Postal code formats have very little in common. The UK is alphanumeric (`SW1A 1AA`), the US is five digits plus an optional ZIP+4, Canada is `K1A 0B1`, Japan is `123-4567`, Brazil is `12345-678`, India is six digits, the Netherlands is four. Every app with an address field ends up re-implementing and re-maintaining this table.

This package ships it as one flat object so you don't have to.

## Installation

```bash
npm install @formfusion/postcodes
```

```bash
yarn add @formfusion/postcodes
```

## Usage

The package exports a single default object mapping lowercase ISO 3166-1 alpha-2 country codes to regex **strings**, or to `null` where no rule is provided.

### ES modules

```js
import postcodes from '@formfusion/postcodes';

console.log(postcodes.jp); // "^(\\d{3,3}(-)\\d{4,4})$"
console.log(postcodes.ae); // null - no rule for the United Arab Emirates
```

### CommonJS

```js
const postcodes = require('@formfusion/postcodes').default;

new RegExp(postcodes.jp).test('100-0001'); // true
new RegExp(postcodes.jp).test('1000001'); // false - missing hyphen
new RegExp(postcodes.jp).test('sw1a 1aa'); // false - wrong format
```

### Plain HTML

The patterns are valid `pattern` attribute values, so they work without any JavaScript:

```html
<label for="postcode">Postal code (Japan)</label>
<input id="postcode" name="postcode" type="text" pattern="^(\d{3,3}(-)\d{4,4})$" required />
```

### With FormFusion

FormFusion passes unknown `type` values straight through to the input's `pattern` attribute, so you can hand it a pattern directly:

```jsx
import React from 'react';
import { Form, Input } from 'formfusion';
import 'formfusion/style.css';
import postcodes from '@formfusion/postcodes';

const MyForm = () => (
  <Form onSubmit={(data) => console.log('Submitted', data)}>
    <Input id="postcode" name="postcode" type={postcodes.de} label="Postal code" required />
    <button type="submit">Submit</button>
  </Form>
);
```

Patterns compose with FormFusion's `rules` combinators if you need to accept more than one country:

```jsx
import { Input, rules } from 'formfusion';
import postcodes from '@formfusion/postcodes';

// Accept either a German or an Austrian postal code
<Input name="postcode" type={rules.existIn([postcodes.de, postcodes.at])} />
```

### Dynamic country selection

```jsx
const [country, setCountry] = useState('de');

<Select name="country" value={country} onChange={setCountry}>
  {Object.keys(postcodes).map((code) => (
    <option key={code} value={code}>
      {code.toUpperCase()}
    </option>
  ))}
</Select>

{postcodes[country] ? (
  <Input name="postcode" type={postcodes[country]} label="Postal code" required />
) : (
  <Input name="postcode" type="text" label="Postal code" required />
)}
```

### Standalone validation

```js
import postcodes from '@formfusion/postcodes';

export function isValidPostcode(value, country) {
  const pattern = postcodes[String(country).toLowerCase()];

  if (pattern === undefined) return false; // unknown country
  if (pattern === null) return true; // no rule provided, cannot reject
  return new RegExp(pattern).test(value);
}

isValidPostcode('100-0001', 'JP'); // true
isValidPostcode('1000001', 'JP'); // false
isValidPostcode('anything at all', 'AE'); // true - AE is a null placeholder
```

### TypeScript

Typings are hand-written in `index.d.ts` and mirror the lowercase keys via a mapped type. The 50 `null` placeholders are deliberately absent from the declarations. Because the declaration uses `export =`, you need `esModuleInterop` or `allowSyntheticDefaultImports`.

```ts
import postcodes from '@formfusion/postcodes';

const jp: string = postcodes.jp;
// @ts-expect-error - unknown country
const xx: string = postcodes.xx;
```

## API

The export is a plain object with no functions or classes:

```ts
{ [countryCode: string]: string | null }
```

Country codes are **lowercase** (`de`, `gb`, `jp`). Lookups are case-sensitive, so normalize user input first.

There are 244 keys: 194 patterns and 50 `null` entries (`ae`, `hk`, `qa`, `ye`, `zw` and 45 others) where no rule is known.

A handful of patterns expect their own prefix as part of the value — `ad` matches `AD100`, `bb` matches `BB12345`, `ht` matches `HT1234` — while most match the bare postal code.

Enumerate the available codes at runtime with `Object.keys(postcodes)`, and the ones that actually have a rule with `Object.keys(postcodes).filter((code) => postcodes[code])`.

## Caveats

Read these before relying on the patterns.

**50 entries are `null`.** `new RegExp(null)` compiles to the regex `/null/`, so a placeholder accepts the literal string `"null"` and rejects every real postcode. Guard on `null` before use.

**Unknown keys are `undefined`, and `new RegExp(undefined)` matches everything.** `postcodes.xx` produces the regex `(?:)`, which passes any input. Guard on `undefined` too.

**Five patterns are not fully anchored.** `nl` is just `\d{4}`, so `abc1234xyz` passes. `cl` is `[0-9]{3}0000` with no anchors at all. `gg` and `je` have no end anchor (`^GY[0-9]?[0-9]`), and `im` has a branch with no anchors at all. If you need a strict match, wrap the pattern yourself.

**`$` binds only the last alternative in some patterns.** `ca` is `^[A-Z][0-9][A-Z]|[A-Z][0-9][A-Z] [0-9][A-Z][0-9]$`, and because of alternation precedence the first branch is unanchored — `K1Axyz` passes. The same applies to `bh` and `cr`.

**Several patterns require literal spaces.** `af`, `fk`, `gr`, `kn`, `mf`, `nf`, `pn` and `sh` only match a value that begins and/or ends with a space, so trimmed input fails. Which side needs the space differs per alternative.

**`eg` uses `\A` and `\Z`.** Those are Python/PCRE anchors. JavaScript reads them as the literal letters `A` and `Z`, so `eg` matches `12345Z` and rejects a plain `12345`.

**Character classes contain literal `,` and `|`.** `bd` is `^([1,3,5,7]\d{3}|...)$` — inside `[...]` the comma is a member of the class, so `,234` passes. The same is true of `|` in `de`, `cz` and others.

**Uppercase is required wherever the pattern uses `[A-Z]`.** `gb`, `ca`, `mt` and most of the alphanumeric formats reject lowercase input. Uppercase before validating.

## Development

```bash
git clone https://github.com/mitevskasara/formfusion-postcodes.git
cd formfusion-postcodes
npm install
npm run build
```

### How it works

All source lives in [`src/index.js`](src/index.js) as a single object of uppercase country codes. The last step lowercases every key before exporting, so `AT` becomes `at`.

[`esbuild.js`](esbuild.js) bundles that into a minified CommonJS `index.js` at the repo root, targeting Node 14. Consumers get the built file, so **changes are not live until you rebuild and commit `index.js`**:

```bash
npm run build
```

### Commit convention

This repo follows [Conventional Commits](https://www.conventionalcommits.org/), and `CHANGELOG.md` is generated from those subjects:

```
Feat: add Croatian postal pattern
Fix: anchor the Canadian pattern
```

### Adding a country

1. Add the entry to `src/index.js`, using an uppercase country code. Use `null` if no rule is known.
2. Add the uppercase code to the `PostalCodes` type in `index.d.ts`, unless the entry is a `null` placeholder.
3. Run `npm run build` and commit the regenerated `index.js`.

## Related packages

Part of the FormFusion family of extracted validation rule sets:

- [`@formfusion/iban`](https://www.npmjs.com/package/@formfusion/iban)
- [`@formfusion/licence-plates`](https://www.npmjs.com/package/@formfusion/licence-plates)
- [`@formfusion/passports`](https://www.npmjs.com/package/@formfusion/passports)
- [`@formfusion/phones`](https://www.npmjs.com/package/@formfusion/phones)
- [`@formfusion/tin`](https://www.npmjs.com/package/@formfusion/tin)
- [`@formfusion/vat`](https://www.npmjs.com/package/@formfusion/vat)
- [`formfusion`](https://www.npmjs.com/package/formfusion) — the core library

## Issues

Report bugs and feature requests at https://github.com/mitevskasara/formfusion-postcodes/issues.

## License

BSD-2-Clause. Copyright (c) 2023, Mitevska Sara.
