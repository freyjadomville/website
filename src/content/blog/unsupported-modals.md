---
title: 'Unsupported Browser Modals Should Not Be Done Via Browser Agent Strings in 2023'
description: 'A summary of why feature detection is viable (and easier!) than using an agent string library'
pubDate: 'Sep 03 2023'
heroImage: '/browser-message.png'
heroAlt: 'A screenshot detailling the footer of my website, in light mode, showing the unsupported browser messages in Microsoft Edge IE Mode'
---

Old browsers. Don't we love them? (sarcasm)

It's often the case these days that if you try to browse the web using Internet Explorer on some older version of Windows or an old Safari version that's tied to an old version of Mac OS, you'll get a messed up web page and a helpful little message that your browser isn't supported for one reason or another. They've been around forever, as a nag to the end user, and as part of a a last gasp effort that allows a developer to hope that the end of one more accursed old browser version is just around the corner.

But let's face it. Implementing this functionality has been a slog since Microsoft first tried to say "Netscape was bad, actually" and that websites were "made for Internet Explorer 6". It's laborious, and it's often unreliable at the best of times. We still need them, but we don't enjoy munging browser agent strings as an industry, so we defer to a potentially brittle library. So much so, that even the web standards folks have said that a [reduction in what's available in browser strings](https://developer.chrome.com/blog/user-agent-reduction-android-model-and-version/) is important and necessary for the web, meaning this method of browser detection gets less reliable by the day.

## There's another way

Thing is... it's always struck me as dumb to cut off one's nose to spite your own face - especially when it comes to makeing good use of developer effort.  My point is thus - browsers have always had the ability to have features tested based on whether they're working correctly. It's how [the famous compat table](https://kangax.github.io/compat-table/es6/) works out how things are behaving in the user's browser. It's how libraries like [Modernizr](https://modernizr.com/) and [core-js](https://github.com/zloirock/core-js) have done things for decades to make the web more accessible. It may look complicated to do all of this work, sure, but these days?

> You've been able to get away with [ES2015 and TLS 1.3 only for 95% of the web](https://browsersl.ist/#q=supports+tls1-3+and+supports+es6+and+supports+es6-module) for a while now, and most build tools are [shipping defaults that are newer than this](https://vitejs.dev/config/build-options.html#build-target)

If you're only using minimal amounts of JavaScript to enhance a page, the content on your blog or marketing site can still be mostly accessible with a warning based on a current feature to try and encourage users to upgrade their browser. Even the `browserslist` project themselves [recommend a default](https://browsersl.ist/#q=last+2+versions%2C+not+dead%2C+%3E+0.2%25) that covers 90% of common users with web traffic, using tools like [babel/preset-env](https://babeljs.io/docs/babel-preset-env).

That means that, assuming you're careful, I'd like to argue that feature detection is the better approach instead of using brittle version string detection.

## The principles of pragmatic feature detection

1) (optional) Enable TLS 1.3 only.
    * The only snag here might be Windows 10 or below, which doesn't support TLS 1.3 on IE web views out of the box, but it can be enabled via `regedit` in most cases.
    * I'm reccommending it because simplifies the cases for IE < 11, and means you don't have to write ES3 in your browser detection code.
    * If you can't get away with this, consider reverse engineering the implementation I wrote in 2020 for [Historic England](https://historicengland.org.uk/) which also uses this approach.
2) Try to find the newest feature you can for the standard you want to support 
     * In my case, I used ES 2020 for this blog site, as it's the [default for Vite](https://vitejs.dev/config/build-options.html#build-target), so ended up with nullish coalescing as the detection point, as per the bottom of this [caniuse page](https://caniuse.com/?feats=mdn-javascript_operators_optional_chaining,mdn-javascript_operators_nullish_coalescing,mdn-javascript_builtins_globalthis,es6-module-dynamic-import,bigint,mdn-javascript_builtins_promise_allsettled,mdn-javascript_builtins_string_matchall,mdn-javascript_statements_export_namespace,mdn-javascript_operators_import_meta).
3) Find detection tools for any CSS features that break your site
    * In my case I had to check in a slightly more roundabout way that tested if `display: flex; gap: 20px;` was supported, because it breaks spacing in some browsers when viewing stuff in more limited viewports

Once you have the ingredients, it's relatively straightforward to construct a polyfill that works in most cases. For the nullish coalescing and flex gap in this example (via Astro's `is:inline` in this specific example, but this is merely supposed to be an inline script):

```html
<script is:inline>
// This could be as simple as checking for
// variables/methods if you're not 
// polyfilling for a language feature.
function browserSupported() {
  try {
    new Function("foo?.bar ?? baz");
    return true;
  } catch (e) {
    return false;
  }
}

// This is only necessary because gap is ambiguously
// defined. If you were doing CSS grid this method
// could be simplified to `CSS.supports('(gap: 20px;)');`
function checkFlexGap() {
  // create flex container with row-gap set
  var flex = document.createElement("div");
  flex.style.display = "flex";
  flex.style.flexDirection = "column";
  flex.style.rowGap = "1px";

  // create two, elements inside it
  flex.appendChild(document.createElement("div"));
  flex.appendChild(document.createElement("div"));

  // append to the DOM (needed to obtain scrollHeight)
  document.body.appendChild(flex);
  // flex container should be 1px high from the row-gap
  var isSupported = flex.scrollHeight === 1; 
  flex.parentNode.removeChild(flex);

  return isSupported;
}
// ... enable relevant classes
</script>
```

With the two functions defined in place, and applying classes to the relevant messages (I have two separate messages to detail varying levels of support and reccommendations), it should be fairly straightforward to appear on the page where you want it, accounting for things like [older Flexbox behaviour](https://css-tricks.com/old-flexbox-and-new-flexbox/) if you need to have something render that's more fancy than a simple message.

## Consider "bundling" your inline scripts regardless

An approach I find useful for this sort of work is to write the ESNext version, as if it's a module script, then use the [TypeScript playground](https://www.typescriptlang.org/play) to give you the version that supports a particular ES standard (possibly even ES5). As long as you choose your method calls carefully, you can get away with writing these scripts in more modern syntax, but without causing old browser issues for your static content that doesn't strictly need JavaScript, especially when it comes to streaming HTML down the wire. The downside is that maintaining these scripts outside of any sort of bundling can get tricky, but the upside is that you get the compatibility you want, for essntially free, if your process is well documented.

## Wait, why are we using `Function` for the nullish coalescing check?

Simply put, `eval` is not secure. If the browser doesn't compile the feature (through `Function`), then the feature isn't supported. It's quite nice as a shortcut for certain things, and if you can be certain of a superset, you minimise the number of browser checks that need to be accounted for.

## Why the parens in the comment above about `CSS.supports`?

Edge legacy versioning issues, plus a couple of other older browsers, require the parentheses, even if they are otherwise optional. Otheriwse, it works just fine for most things, Flexbox gap is just a bit trickier to think about.

## Comments TBD

Once I've got them working there'll be a Mastodon and Cohost post for this blog, where comments can be made, but until then, feel free to ping me on Misskey.
