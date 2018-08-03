# First Draft

## Introduction

Clio's main software offering received a revamp in 2017. Our legacy application, now dubbed Old Clio, is a Rails application that relied on the Rails templating engine. The revamp, codenamed Apollo, is a single page application compiled together using Webpack.

Our Production Engineering team have set up an automated system for building newer versions of Apollo. We use BuildKite as our continuous integration (CI) platform. The system is pretty slick: Whenever a developer checks code into our repository, a new build will be created and ran automatically, compiling together the source code and running tests as per dictated by our set up.

This process has served our developers well, and it has enabled "Single Click Shipping": Commit your code, wait for compilation, and press a single button to ship it. However, as the size of Apollo grew, so too did the amount of time it took for it to be built on CI. Soon enough, we reached our tipping point.

Before going any further, I would like to quickly mention who exactly we are. We are the Front End Infrastructure (FEI) team. Our mission is to create an amazing front-end development environment here at Clio. As a part of this mission, we are responsible for overseeing the usage and maintenance of various front-end technologies.

## Red Alert

In the beginning of May, the FEI team started receiving a stream of alerts on our Slack channel. People were being frustrated by the sheer amount of time it took to compile Apollo, a number of people even reported compilation failures.

Looking at BuildKite, we were greeted by a horrendous sight. Not only were most successful builds taking upwards of 20 minutes to compile, there were also a number of builds that failed after running for similar lengths of time. Developer frustrations aside, this was unacceptable because it hindered Clio's ability to respond effectively to emergency situations. For the FEI team, it was all-hands-on-deck.

After some initial investigation, we determined that the issue was caused by the depletion of RAM on our remote CI agents. In short, our application has grown to a size where Webpack could no longer compile it within the memory limits placed upon it.

So how can we resolve this issue? If money was no object, we could have upgraded to instances with more RAM. However, we felt that our application's current size did not justify such an expense and the issue was most likely the result of poor optimization. As such, we got down to the business of optimizing our build.

## Webpack v4

We decided that the first step in the optimization of our Apollo's compilation was the upgrade of Webpack to v4. There were two reasons for this decision.

When Webpack v4 was released, its headline was [performance](https://medium.com/webpack/webpack-4-released-today-6cdb994702d4). As other folks upgraded their applications to the latest versions of Webpack, many started reporting significant performance improvements. We hoped, if we were lucky, that the upgrade could be the silver bullet we needed to get ourselves back in the green. A little hopeful, yes, but why shouldn't we take advantage of the latest and greatest?

The second reason is a little bit more plain. The FEI team already had the upgrade to Webpack v4 in its pipeline. Being a major version bump, it is possible that any optmizations we find/implement whilst on Webpack v3 will be moot once we upgraded. Therefore, we felt it better to upgrade now, see what it gives us, and improve things from there.

Apollo was built with a blend of many different front-end technologies. CoffeeScript, TypeScript, AngularJS and Sass are the main ingredients in this blend, complimented by healthy dosages of many other libraries and tools. As a result, our Webpack configuration is rather heavy: We have at least 3 different build environments and utilize more than 20 loaders and 9 plugins.

With such a "mature" Webpack v3 set-up, the upgrade to Webpack v4 was not straightforward. The rest of this article will provide a description of the strategy we used to upgrade to the latest version, with an emphasis on the issues we encountered and how we went about resolving them.

## The Upgrade

The actual upgrading of Webpack itself is simple. We use `yarn` to manage our Node modules, so, to upgrade to Webpack v4, we simply ran:

```
yarn upgrade webpack@v4.8.3
```

You'll notice that we specified the exact version. We do this with all of our Node modules to avoid any unintentional upgrades. Why `v4.8.3`? No particular reason, anything `v4.X.X` would have been fine, `v4.8.3` just happened to be the latest version available when we performed the upgrade.

Coupled with this upgrade, we also immediately upgraded `webpack-dev-server` to `v3.X.X` and added `webpack-cli`, which used to be a part of `webpack` but has been separated into another package in v4:
```
yarn upgrade webpack-dev-server@v3.1.4
yarn add -D webpack-cli@v2.1.4
```

## Compile God Damn It

After the upgrade, our application was completely broken. This is where the actual work began. For the most part, the errors that we got were the result of outdated Webpack loaders/plugins, but we also found it necessary to adjust some configurations to meet with the new specifications of Webpack v4. In no particular order, the following is a list of recommendations and interesting pieces of knowledge that we think will be useful for anyone attempting to upgrade their own Webpack project.

### One Step At a Time

First piece of advice: Do the upgrade one step at a time. Initially, our upgrade plan was to perform a systematic review of our entire Webpack set-up and preemptively resolve as many issues as we possible could. This involved, among other things, a complete review of all of our loaders and plugins to see if they were compatible with the new version of Webpack, if they needed an upgrade, or if they were deprecated since v4's release.

We chose this strategy initially because we liked the idea of "doing everything right the first time." In practice, however, this actually led to us "doing nothing right the first time." The issue that we ran into is that, once we finished doing all the things we thought were necessary, and Webpack failed to compile our application, we were at a loss for what went wrong. The change set was simply too large, and we couldn't diagnose the issue.

So, as familiar as this advice is to many, it is nevertheless worth repeating here in the context of Webpack: Make each step in your upgrade as small as possible. Webpack actually lends itself nicely to this process, and we found the error outputs we got usually gave us a good idea of what the next step should be. In particular, we recommend the following iterative process for upgrading:

1. Run your Webpack compilation.
2. Google the errors that you get.
3. Upgrade/Reconfigure as necessary to resolve this one error.
4. Repeat 1 ~ 3 until Webpack compiles and everything works properly.

### CommonsChunkPlugin

One of the errors we got during our upgrade was this:

```
Error: webpack.optimize.CommonsChunkPlugin has been removed, please use config.optimization.splitChunks instead.
```

Like many other Webpack projects, we used the `CommonsChunkPlugin` to generated separate chunks for our third-party modules and the Webpack manifest file. These files tend to change less often than our bundled application file, so by separating them out, our clients' browsers can cache them separately and experience less-costly cache busts.

To accomplish this, we had the following set-up in our Webpack configuration file:

```js
plugins: [
  new webpack.optimize.CommonsChunkPlugin({
    name: 'apollo-vendor',
    minChunks: function (module) {
      return module.context && module.context.indexOf('node_modules') !== -1;
    }
  }),
  new webpack.optimize.CommonsChunkPlugin({
    name: 'apollo-manifest'
  })
],
```

In Webpack v4, the `CommonsChunkPlugin` was deprecated. Instead, Webpack has introduced a new configuration property, `optimization`, that handles various output optimization settings. Luckily, our usage of the `CommonsChunkPlugin` was fairly standard, and we were able to find a direct translation of our old configuration:

```js
optimization: {
  splitChunks: {
    cacheGroups: {
      commons: {
        test: /[\\/]node_modules[\\/]/,
        name: "apollo-vendor",
        chunks: "all"
      }
    }
  },
  runtimeChunk: {
    name: "apollo-manifest"
  }
}
```

So by removing all instances of the `CommonsChunkPlugin` from the `plugins` array and adding the above to our configuration object, we were able to resolve this error and achieve the same outcomes as before.

### ExtractTextWebpackPlugin

Another error that we ran into looked like this:

```
Error: Chunk.entrypoints: Use Chunks.groupsIterable and filter by instanceof Entrypoint instead
    at Chunk.get (/home/vagrant/clio/themis/node_modules/webpack/lib/Chunk.js:712:9)
    at /home/vagrant/clio/themis/node_modules/extract-text-webpack-plugin/dist/index.js:176:48
```

This time, the error itself is not as important as its source. In this case, the culprit is `extract-text-webpack-plugin`. This is another one of those "documented breakages." In prior versions of Webpack, the `extract-text-webpack-plugin` was commonly used to lift CSS out of compiled JS bundles and into dedicated CSS asset files. We, too, used it for this purpose. The Webpack team now recommends using `mini-css-extract-plugin` for this role, but `extract-text-webpack-plugin` is still around as it has other use cases as well.

To deal with this error, you actually have two options. In the long-term, you probably want to use `mini-css-extract-plugin`. But, if you want to "park" that and do it in a later fix, you can actually upgrade `extract-text-webpack-plugin` to one of its beta releases. This is what we did:

```
yarn upgrade extract-text-webpack-plugin@v4.0.0-beta.0
```

Without changing any other plugins/configurations, this was sufficient for resolving this issue.

### UglifyJSPlugin

The last "special issue" we ran into whilst upgrading is this one:

```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory
```

Out of context, this error could mean literally anything, hence it was a bit harder for us to pin this one down. The hint to the solution was in the timing: We noticed that this error always arose as UglifyJS was running.

In previous versions of Webpack, you would use the `UglifyJSPlugin` by including it in the `plugins` array of your configuration object:

```js
plugins: [
  new webpack.optimize.UglifyJsPlugin({
    // ... This object configures the behaviour of the plugin
  }),
]
```

In Webpack v4, this syntax is no longer supported. In its place, the `optimization.minimizer` property now takes on the role of telling Webpack how it should optimize the size of output bundles:

```js
// Remember to require the plugin
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');

module.exports = {
  // ... Other configuration properties
  optimization: {
    minimizer: [ // This array tells Webpack which minimizers it should use
      new UglifyJsPlugin({
        // ... Specify options of UglifyJS here
      })
    ]
  },
}
```

In our production Webpack set-up, we found that we were still using the old `plugins` array syntax. Knowing this was definitely wrong, we removed it and tried our compilation again, just to see what happens. To our surprise, this resolved the issue!

It is also worth noting that, thanks to Webpack's `mode` defaults, you don't have to specify `optimization.minimizer` in order to get uglification. By default, if your `mode` is set to `"production"`, Webpack will perform uglification on the output. Hence, by simply removing the `UglifyJsPlugin` from our `plugins` array and making sure our `mode` is set properly, we were able to get past this error and still have optimized output.

### Upgrading Other Plugins

Back to the top of the cycle, we ran Webpack compile again. Error again:

```
/home/vagrant/clio/themis/node_modules/fork-ts-checker-webpack-plugin/lib/index.js:143
    _this.compiler.applyPluginsAsync('fork-ts-checker-service-before-start', function () {
                           ^

TypeError: _this.compiler.applyPluginsAsync is not a function
```

This time, the error itself is not as important as where the error came from.
Looking at the stack trace, we were able to see that the error originated from
`fork-ts-checker-webpack-plugin`.

What is happening here is that the `fork-ts-checker-webpack-plugin` is on an
older version and is trying to use a Webpack function that has been removed in
v4. Looking up the `fork-ts-checker-webpack-plugin` on GitHub and checking out
its release history, we were able to find that `v0.4.0` is the first version
for which Webpack v4 is supported. We thus upgrade this package:
```
yarn upgrade fork-ts-checker-webpack-plugin@v0.4.1
```

### Upgrading Loaders

Back to the top of the cycle, we ran Webpack compile again. Error again:

```
ERROR in ./client-src/themisui-vendor.scss
Module build failed: ModuleBuildError: Module build failed: TypeError: Cannot read property 'context' of undefined
    at getLoaderConfig (/home/vagrant/clio/themis/node_modules/fast-sass-loader/lib/index.js:72:29)
```

This time, it looks like we were able to get past the plugins initialization
stage of Webpack. The error output was generated after Webpack started reading
files and resolving dependencies. Our particular error came from
`fast-sass-loader`.

Similar to what we did for `fork-ts-checker-plugin`, we searched online and was
able to find that `fast-sass-loader` only started supporting Webpack v4 in
`v1.4.1`. So we did an upgrade:
```
yarn upgrade fast-sass-loader@v1.4.5
```

Going back to the top of the cycle, we quickly discovered that we had to do the
same for a bunch of other loaders as well. In all cases, similar errors
and, looking online, we verified that the loader was actually out of date by
checking out the release history. When we found a newer version that supported
Webpack v4, we would upgrade. For posterity, below is a complete list of the
loaders that we touched as a part of the upgrade:

```
// These had to be upgraded for Webpack to compile
fast-sass-loader@v1.4.0 -> v1.4.5
ts-loader@v3.2.0 -> v4.3.0
file-loader@v0.11.1 -> v1.1.11

// Temporarily disabled as it does not yet support v4
tslint-loader

// This was removed because we didn't need it
json-loader // Built into Webpack
rails-erb-loader // We don't use it anymore
style-loader // We don't use it anymore

// Why?
css-loader@v0.28.4 -> v0.28.11
postcss-loader@v2.0.6 -> v2.1.5
istanbul-instrument-loader@v3.0.0 -> v3.0.1
```

There is one special case that needs to be mentioned: `tslint-loader`. This
loader has also been broken by Webpack v4, and it appears that the authors have
acknowledged this. They have a fix merged into their master branch but have not
yet attached a version to this change:
https://github.com/wbuchwalter/tslint-loader/pull/95
As a result, we have currently disabled `tslint-loader`.

After this, Webpack was able to compile! Jubilated, we decided to fire up our
Rails server and see if our bundles actually ran properly. Incredibly, we found
our app working as it should!

What about `webpack-dev-server`? Our development configuration has also
specified options for `webpack-dev-server` and we have upgraded it as well. So
it's worth checking that it isn't broken. To our surprise and great relief,
when we fired up `webpack-dev-server` and got it hooked up with our Rails
application, it worked flawlessly without any changes to our configuration.
Furthermore, we also noticed that recompilation on v4 was significantly faster
than in v3, so that was a really nice little cherry on top.

**INSERT CELEBRATION GIF**

## Tidying Up

Lastly, we made some further adjustments to make our Webpack process better for everyone. Essentially, we are taking advantage of the benefits that Webpack provides.

### Set Your Mode

First piece of advice: Make sure that you are setting the `mode` explicitly in all of your Webpack configurations. The new `mode` property is one of the main features of Webpack v4: It enables zero-configuration Webpack projects. If left unspecified, Webpack will default `mode` to `"production"`. This will cause Webpack to use a set of "sensible defaults" when compiling your application.

When we upgraded to v4, we forgot to set the `mode` to `"development"` in our development and test environments. This led to ridiculously long recompilation times as Webpack was performing all sorts of unnecessary output optimization every time we changed the source code.
