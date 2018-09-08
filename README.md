# `rollup-plugin-postcss-treeshakeable`

Enables treeshaking of modular CSS produced by `postcss`

## Installation

Install the package

```
npm install rollup-plugin-postcss-treeshakeable --save-dev
```

Add it to your `rollup.config.js`

```js
import postcss from "rollup-plugin-postcss";
import postcssTreeshakeable from "rollup-plugin-postcss-treeshakeable";

export default {
  // ...
  plugins: [
    postcss({
      modules: true,
      plugins: []
    }),
    postcssTreeshakeable()
  ]
};
```

The `postcssTreeshakeable` must appear after the `postcss` plugin.

## Disclaimer

This package is experimental. A first test showed that the technique works, but it has not been tested on any big project yet.

## Example

Check out the `example` folder, and especially the [`example/README.md`](./example/README.md) walkthrough to see what this plugin does and follow it to see it in action.

## Gist

This plugins transforms this code (which gets produced by `postcss` when importing a css module):

```js
var css = ".foo { background: red; }";
export default { foo: "foo__classname_1 " };
import styleInject from "<some-folder>/style-inject/dist/style-inject.es.js";
styleInject(css);
```

into this code

```js
import styleInject from "<some-folder>/style-inject/dist/style-inject.es.js";
var cssMap = { foo: "foo__classname_1 " };
var hasRun;
var styles = function styles() {
  if (!hasRun) {
    hasRun = true;
    styleInject(".foo { background: red; }");
  }
  return cssMap;
};

export default styles;
```

🎉 This allows consumers to get rid of CSS of unused modules when treeshaking.

## Long explanation

### Situation

You are writing a cool library which lets people render a `Butler` on screen. The `Butler` is provided as a React component and uses CSS modules to have some styles applied. The library also comes with a bunch of other components.

Whoever uses your library should only bundle the code and styles of the components they are actually using. The unused component code and styles should be excluded from the bundle.

A technique called Treeshaking lets us do this. However, it was so far only able to omit the code of unused components. The styles of unused components would have still been lingering around in your bundle, even though they were never used!

This plugin enables library authors which use PostCSS and bundle with Rollup to create components whose styles can be omitted through treeshaking. This creates smaller bundles and thus makes the application the library consumers are building slimmer.

### What you've been doing

So far you've probably been importing CSS modules like this:

```jsx
import styles from "./Butler.mod.css";

const Butler = props => <div className={styles.alfred}>I am Butler</div>;
```

Given that `Butler.mod.css` has this content:

```css
.alfred {
  background: red;
}
```

The `Butler.mod.css` gets turned into this by postcss:

```js
var css = ".Butler-mod_alfred__3zrhW {\n  background: red;\n}\n";
export default { alfred: "Butler-mod_alfred__3zrhW" };
import styleInject from "<some-folder>/style-inject/dist/style-inject.es.js";
styleInject(css);
```

When you bundle this with with rollup, you end up with the following in your generated bundle:

```js
function styleInject(css, ref) {
  /* redacted */
}

var css = ".Butler-mod_alfred__3zrhW {\n  background: red;\n}\n";
var styles = { alfred: "Butler-mod_alfred__3zrhW" };
styleInject(css);

var Butler = function Butler(props) {
  return React.createElement(
    "div",
    {
      className: styles.alfred
    },
    "I am Butler"
  );
};

export { Butler /* other components */ };
```

### Treeshaking components

If the consumer of your library doesn't import `Butler`, but only imports the other components, then we don't need to keep `Butler` around.

Treeshaking does exactly this and marks the `Butler` export as unused. Once consumers of the library you're building run dead code elimination (for example by running Uglify), `Butler` won't end up in their application bundle anymore.

This is pretty great, however the styles of `Butler` would currently still end up in the produced bundle.

### Treeshaking styles

This plugin transforms the contents of `Butler.mod.css` into a representation which can be removed through treeshaking.

It transforms the contents of modular css files generated by postcss from

```js
var css = ".Butler-mod_alfred__3zrhW {\n  background: red;\n}\n";
var styles = { alfred: "Butler-mod_alfred__3zrhW" };
styleInject(css);
```

to this

```js
var cssMap = { alfred: "Butler-mod_alfred__3zrhW" };
var hasRun;
var styles = function styles() {
  if (!hasRun) {
    hasRun = true;
    styleInject(".Butler-mod_alfred__3zrhW {\n  background: red;\n}\n");
  }
  return cssMap;
};
```

### What you'll be doing now

We are not exporting an object of classnames anymore, but we're rather exporting a function returning that object. The function will add inject the styles the first time it gets called.

The consumers of the CSS modules have to be adapted, as we're now exporting a function instead of an object:

```diff
import styles from "./Butler.mod.css";

-const Butler = props => <div className={styles.alfred}>I am Butler</div>;
+const Butler = props => <div className={styles().alfred}>I am Butler</div>;
```

After applying this plugin and after rewriting the usage of imported CSS (by turning `styles` into `styles()`), your library consumers will benefit from treeshaking it!

### Why it works

Notice that we're not calling `styleInject` directly, but we're rather waiting until the styles are actually being used. At that point, we add the styles to the document.

When treeshaking now removes unused components, it can also remove their unused CSS, as the styles are not referred to anymore after the components have been omitted.

And thus, no more CSS of unused components in the bundles of your library users.

## Caveats

All styles of one modular CSS file get injected the first time any of its styles are used. So, if you want to apply the styles of some files right away, importing them is no longer good enough.

You have to call the function as well:

```js
import someStyles from "./somewhere.css";

// This call injects them into the document and thus applies them.
// It further returns the classnames contained in `somewhere.css`
someStyles();
```
