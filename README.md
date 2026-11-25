# Extension manifest permission surface

Paste a browser extension manifest and a list of URLs, and see per manifest key which URLs are granted and via exactly which match pattern, resolved entirely in your browser.

**Live demo:** https://0xelitesystem.github.io/extension-manifest-permission-surface/

## Use

1. Open the page and paste one `manifest.json` into the left box.
2. Paste concrete candidate URLs, one per line, into the right box.
3. Press Resolve.

For every URL the page prints one row with six columns, one per pattern bearing key. Each cell reads
`GRANTED` with the exact pattern that matched, `no match`, `EXCLUDED` with both the matching pattern
and the exclude pattern that beat it, `EXCLUDES` where an `exclude_matches` list covers the URL,
`INVALID` with the parser's own reason, `unparseable URL`, `UNKNOWN, matcher refused` where the
matcher hit its step budget, or `key not present in this manifest`.

There is no score, no letter grade, no advice paragraph and no network call. One click on "Load the
sample" prints `GRANTED`, `EXCLUDED`, `EXCLUDES`, `no match`, `key not present in this manifest` and
`unparseable URL` in the grant table, and an `INVALID` pattern with the parser's reason in the
pattern inventory under it.

## Why this exists

The same match pattern means different things in different manifest keys, and the difference is not
cosmetic. `https://*/` grants every path under `host_permissions`, because that key discards the
path, and matches only the path `/` under `content_scripts`, because that key does not. A tool that
applies one rule everywhere is wrong in one direction or the other, and both directions are
dangerous: ignoring the path everywhere over reports content script coverage, and glob matching it
everywhere under reports host permission grants.

Three more things go wrong in practice:

- **Suffix versus substring.** `https://*.example.com/*` must reject
  `https://www.example.com.evil.tld/`. Written as a substring test instead of a label boundary test,
  it accepts it. That check is a fixture in the page's self test.
- **The port component.** The grammar has four components, not three. A parser written to three
  rejects `http://*:*/*` with a host wildcard error, and takes every explicit port pattern down with
  it. That is an origin set the manifest really grants, silently missing from the report.
- **Empty column theatre.** An absent key rendered as an empty column reads like a finding. This page
  prints "key not present in this manifest" instead, and only claims that two keys resolve to different
  origin sets when the pasted manifest actually makes them different, computed from the rendered rows.

## Accuracy

The two flags per key, exactly as implemented:

| Manifest key | Path | Schemes accepted |
| --- | --- | --- |
| `host_permissions` | ignored | chrome, http, https, file, ftp, ws, wss, uuid-in-package |
| `optional_host_permissions` | ignored | chrome, http, https, file, ftp, ws, wss, uuid-in-package |
| `content_scripts[].matches` | matched as a glob | chrome, http, https, file, ftp, uuid-in-package |
| `content_scripts[].exclude_matches` | matched as a glob | chrome, http, https, file, ftp, uuid-in-package |
| `externally_connectable.matches` | matched as a glob | any scheme |
| `web_accessible_resources[].matches` | must be exactly `/*` | any scheme |

A wildcard scheme overrides the schemes column: `*` narrows to http or https in every key, including
the two that otherwise accept any scheme, because `SetScheme` intersects the mask the moment it sees
`*`.

**What was verified, and how.** The page carries 127 checks that run on every load, and it refuses
to render a grant table if any of them fails. The fixtures are not invented. They are the parse
result and port tables from Chromium's own `url_pattern_unittest.cc`, the example patterns from the
Chrome match patterns documentation, the per key path split, the suffix lookalike cases, the path
canonicalisation cases, the `web_accessible_resources` path rule, and one positive control: a
deliberately false claim that the checker must report as rejected. All 127 pass, and the control is
rejected. The same 127 checks were also run outside the browser, against the script extracted from
the shipped `index.html`, so the verified code is the code that ships.

The control was proved to be real, by breaking the parser on purpose and counting. Move the host
wildcard check up in `parsePattern` so that it runs on the raw host and port text, before the port
separator is found, which is exactly the three component parser this page exists to argue against,
and 34 of the 127 checks go red, including `http://*:*/` failing with an invalid host wildcard
error. A checker that has never produced a failure has not been tested.

**Sources read at build time**, in the Chromium main branch, on 2026-09-08:

- `extensions/common/url_pattern.cc`: `port_("*")` at :159, :165 and :173; the port separator search
  inside `host_and_port` at :269 to :300, which runs before the host is validated; `SetPath` at :407,
  which stores the path verbatim with no expansion of a bare `/`; `IsValidPortForScheme` at :98 to
  :113; `MatchesPortPattern` at :907, which is `return port_ == "*" || port_ == port;`
- `extensions/common/url_pattern_unittest.cc`: the `ParseInvalid` and `Ports` tables, and the
  explicit port match cases
- `extensions/common/manifest_handlers/permissions_parser.cc` :150 to :152, which forces the path to
  `/*` after a successful parse, under the comment "The path component is not used for host
  permissions, so we force it to match all paths."
- `extensions/common/manifest_handlers/externally_connectable.cc` :106 to :108, which constructs the
  pattern with `SCHEME_ALL` under the comment "Safe to use SCHEME_ALL here; externally_connectable
  gives a page -> extension communication path, not the other way."
- `extensions/common/manifest_handlers/web_accessible_resources_info.cc`: the `SCHEME_ALL`
  construction at :120, the `pattern.path() != "/*"` rejection at :121 to :124 that refuses the
  whole manifest, the manifest version 2 legacy bypass at :199 to :201, and the
  `entry.matches.MatchesURL(initiator_url)` gate at :231 inside `IsResourceWebAccessibleImpl`
- `extensions/common/user_script.cc` :69 to :73 and :109 to :118 for the content script scheme mask,
  and `extensions/common/extension.cc` :217 to :221 for the host permission scheme mask
- `url/url_canon_stdurl.cc` :122 to :148 for `DefaultPortForScheme`
- `url/url_util.cc` :50 to :64, `content/common/url_schemes.cc` :61 to :66 and
  `chrome/common/chrome_content_client.cc` :186 to :204 for the standard scheme registry
- Chrome for Developers, Match patterns, at
  https://developer.chrome.com/docs/extensions/develop/concepts/match-patterns
- Chrome for Developers, the `web_accessible_resources` manifest reference

Line numbers move as the branch moves. The function names do not, and the page says the date the
files were read.

**A correction that came out of the verification.** `*://example.com:8443/*` is not a valid match
pattern. `IsValidPortForScheme` rejects any non wildcard port whose scheme has no default port, and
`DefaultPortForScheme` knows only http, https, ftp, ws and wss. The wildcard scheme `*` is not among
them, so Chromium returns an invalid port error. Write `https://example.com:8443/*` instead. This is
a fixture in the self test rather than a claim in prose.

**The URL side is canonicalised, the pattern side is not.** Chromium matches against
`GURL::PathForRequest()`, so by the time the glob runs, dot segments have been resolved, a backslash
has become a path separator under a special scheme, the bytes a URL may not carry literally have
been percent encoded, and tab, carriage return and line feed have been deleted from the spec rather
than encoded. This page does the same four things to the URL and leaves the pattern
verbatim, exactly as `URLPattern::SetPath` does, which is why `https://example.com/b*` grants
`https://example.com/a/../b` here as it does in Chrome. Matching the raw text instead prints a
confident "no match" for a URL the manifest really grants. Those cases are fixtures in the self test.

**What this tool does not do.** It does not read a packaged extension, only the manifest text you
paste. It does not check Chrome Web Store policy. It does not tell you whether a permission is
justified. It does not model runtime grants such as `activeTab`, user granted optional permissions,
runtime `chrome.scripting` registrations, or declarative net request rules. It does not model the
install warning path, so a pattern Chrome accepts at parse time and drops later still reads as valid
here. It does not percent decode paths before matching, does not apply internationalised domain name
conversion, and does not canonicalise shorthand IP forms such as `127.1`, all three of which Chromium
does. Those limits are printed on the page as well as here.

## Privacy

Everything runs in your browser. Nothing is uploaded, stored or transmitted. There is no network call
of any kind in this page: no fetch, no background request, no external font, no analytics and no
telemetry. Nothing is written to browser storage either, so closing the tab leaves nothing behind.
The manifest you paste never leaves the machine you paste it on, which is the point, because a
manifest is a description of an unshipped product as often as it is a description of a shipped one.

## Run locally

```
git clone https://github.com/0xelitesystem/extension-manifest-permission-surface.git
cd extension-manifest-permission-surface
```

Open `index.html` in any browser. Opening it straight off the local disk works, with no server and
no build step, because the page uses nothing that requires a secure context.

If you prefer to serve it:

```
python -m http.server 8000
```

then open http://localhost:8000/

## Build

There is no build. The project is one self contained `index.html` with inline CSS and inline
JavaScript, no dependencies, no package manager and no bundler. Edit the file and reload the page.

## Accessibility

- Contrast was verified by arithmetic, not by eye, for every text token against every background
  token in both the light and the dark theme, whether or not the page currently paints that pair.
  The worst text pair is 5.06:1 in light and 7.77:1 in dark, against a WCAG AA minimum of 4.5:1.
  Control boundaries clear the 3:1 non text minimum in both themes. The computed table is in a
  comment at the top of `index.html`.
- No result is carried by colour alone. Every cell prints a word, `GRANTED`, `no match`, `EXCLUDED`,
  `EXCLUDES`, `INVALID`, `unparseable URL`, `UNKNOWN, matcher refused` or `key not present in this
  manifest`, with a leading glyph and a distinct font weight. The state colours differ by hue, not by
  lightness: in greyscale `GRANTED` and `no match` sit 1.44:1 apart in the light theme and 1.20:1 in
  the dark one, and a wider gap is not reachable, because every light theme text token has to stay at
  or below a relative luminance of 0.1306 to clear 4.5:1 on `--surface-2`. So the word, the glyph and
  the weight are the channels that carry the state, and the colour is decoration on top of them. The
  arithmetic is in the comment block at the top of `index.html`.
- Real tables with a `<caption>`, `<th scope="col">` on the key columns and `<th scope="row">` on
  each candidate URL.
- Every textarea has a visible `<label>`, not a placeholder standing in for one.
- The self test banner is a polite live region, so a screen reader hears the pass or fail without
  focus being stolen on load. A refusal is announced as an alert.
- Focus rings are visible in both themes and are never removed. Everything is keyboard reachable.
  There are no drag targets and no hover only affordances.
- `prefers-color-scheme` is honoured.
- Long patterns and long URLs wrap inside a horizontally scrollable container. The page body never
  scrolls sideways.

## Third-party notices

No third-party code is reproduced in this repository. There are no dependencies, no vendored files
and no copied source. The behaviour of the match pattern parser and matcher was reimplemented from
the published specification and from reading the public Chromium source and its public unit test
tables, which are cited above by file and line so the reimplementation can be checked. Short
quotations from Chrome for Developers documentation pages appear on the page, each marked as a
quotation and attributed, and were diffed against the pages as fetched on 2026-09-08.

This project is independent and unaffiliated with Google, the Chromium project, or any browser
vendor. No endorsement is implied. Product and documentation names are used only to identify the
format being parsed.

## Related

- https://github.com/0xelitesystem/csp-header-toggle-builder
- https://github.com/0xelitesystem/cors-basics-reference
- https://github.com/0xelitesystem/url-parser-and-query-editor

## License

MIT. Copyright (c) 2026 RBL Industries LLC.
