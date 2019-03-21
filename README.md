## DropCSS

A simple, thorough and fast unused-CSS cleaner _(MIT Licensed)_

---
### Introduction

DropCSS is an unused CSS cleaner; it takes your HTML and CSS as input and returns only the used CSS as output. The core is simply some minimal glue between these awesome low-level tools:

- [Fast HTML Parser](https://github.com/taoqf/node-html-parser)
- [CSSTree](https://github.com/csstree/csstree)
- [css-select](https://github.com/fb55/css-select)

The entire logic for DropCSS is this [~60 line file](https://github.com/leeoniya/dropcss/blob/master/src/dropcss.js).

It is recommended to also run your CSS through an optimizer like [clean-css](https://github.com/jakubpawlowicz/clean-css) to group, merge and remove redundant rules. Whether this is done before or after DropCSS is up to you, but since `clean-css` can also perform CSS minification, it probably makes sense to run DropCSS first to avoid bouncing [and re-parsing] the output back and forth (optimize & minify -> drop) vs (optimize -> drop -> minify), though this will depend on the actual inputs; profiling is your friend.

---
### Install

```
npm install -D dropcss
```

---
### API

```js
const dropcss = require('dropcss');

let html = `
    <html>
        <head></head>
        <body>
            <p>Hello World!</p>
        </body>
    </html>
`;

let css = `
    .card {
      padding: 8px;
    }

    p:hover a:first-child {
      color: red;
    }
`;

const whitelist = /\b(?:#foo|\.bar)\b/;

let dropped = new Set();

// returns a string
let cleanCSS = dropcss({
    html,
    css,
    shouldDrop: (sel) => {
        if (whitelist.test(sel))
            return false;
        else {
            dropped.add(sel);
            return true;
        }
    },
});

console.log(cleanCSS);

console.log(dropped);
```

`shouldDrop` is called for every CSS selector that could not be matched in the `html`.
Return `false` to retain it or `true` to drop it.
Additionally, this callback can be used to log all removed selectors.

---
### Features

DropCSS stands on the shoulders of giants.

- Due to the exhaustive selector support of `css-select`, it properly supports testing for and removing of practically all conceivable selectors: https://github.com/fb55/css-select#supported-selectors.
- CSSTree easily parses media queries, so they are transparently processed and removed like all other blocks with no special handling required.
- DropCSS avoids removing any transient pseudo-class and pseudo-element selectors which cannot be deterministically checked from the parsed HTML.

---
### Performance

#### Input

**test.html**

- 18.8 KB minified
- 502 dom nodes via `document.querySelectorAll("*").length`

**styles.min.css**

- 27.67 KB minified
- contents: [bootstrap reboot.css](https://github.com/twbs/bootstrap/blob/master/dist/css/bootstrap-reboot.min.css), a custom flex grid, global & page-specific styles. (the grid accounts for ~85% of this starting weight, lots of media queries & repetition)

#### Output

<table>
    <thead>
        <tr>
            <th></th>
            <th>lib size w/deps</th>
            <th>output size</th>
            <th>time elapsed</th>
            <th>unused bytes (test.html coverage)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th><strong>DropCSS</strong></th>
            <td>
                2.16 MB<br>
                251 Files, 51 Folders
            </td>
            <td>6.67 KB</td>
            <td>195ms</td>
            <td>575 / 8.5%</td>
        </tr>
        <tr>
            <th><a href="https://github.com/uncss/uncss">UnCSS</a></th>
            <td>
                13.7 MB<br>
                2,831 Files, 301 Folders
            </td>
            <td>6.72 KB</td>
            <td>424ms</td>
            <td>730 / 10.6%</td>
        </tr>
        <tr>
            <th><a href="https://github.com/FullHuman/purgecss">Purgecss</a></th>
            <td>
                2.53 MB<br>
                513 Files, 110 Folders
            </td>
            <td>8.01 KB</td>
            <td>79ms</td>
            <td>1,898 / 23.1%</td>
        </tr>
        <tr>
            <th><a href="https://github.com/purifycss/purifycss">PurifyCSS</a></th>
            <td>
                3.45 MB<br>
                791 Files, 207 Folders
            </td>
            <td>15.4 KB</td>
            <td>186ms</td>
            <td>9,532 / 60.2%</td>
        </tr>
    </tbody>
</table>

**Notes**

- About 400 "unused bytes" are due to an explicit/shared whitelist, not an inability of the tools to detect/remove that CSS.
- About 175 "unused bytes" are due to vendor-prefixed css properties (-moz, -ms) that are not active in Chrome (used to test coverage), but are properly retained.
- Purgecss is fast but has no support for removing attribute selectors [Issue #110](https://github.com/FullHuman/purgecss/issues/110), and maybe other stuff.

---
### Caveats

- Not tested against malformed HTML (the underlying Fast HTML Parser claims to support common cases but not all)
- Not tested against malformed CSS (the underlying CSSTree parser claims to be "Tolerant to errors by design")
- There is no processing or execution of `<script>` tags; your HTML must be fully formed (or SSR'd). You should generate and append any additional HTML that you'd want to be considered by DropCSS. If you need JS execution, consider using the larger, slower but still good output, `UnCSS`. Alternatively, [Puppeteer can now output coverage reports](https://www.philkrie.me/2018/07/04/extracting-coverage.html), and there might be tools that utilize this coverage data to clean your CSS, too. DropCSS aims to be minimal, simple and effective.

---
### TODO

- See if any perf can be gained. Run a profile and maybe cache result of querySelector for identical selectors encountered across multiple @media blocks.
- (Internal) figure out how to properly prune empty @media query block nodes from the AST instead of doing a regex replace on the output.

---
### Similar Projects

- [UnCSS](https://github.com/uncss/uncss)
- [Purgecss](https://github.com/FullHuman/purgecss)
- [PurifyCSS](https://github.com/purifycss/purifycss)