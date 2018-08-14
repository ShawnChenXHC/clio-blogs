# First Draft

## Introduction

Clio's main software offering received a revamp in 2017. Our legacy application, dubbed Themis, is a Ruby on Rails application that uses ERB templates to serve up the views. The revamp, codenamed Apollo, is a single page application compiled together using Webpack and backed by our [public API](https://app.clio.com/api/v4/documentation).

Our Production Engineering team set up an automated system for building newer versions of our app. We use BuildKite as our continuous integration (CI) platform. The system is pretty slick: whenever a developer checks code into our repository, a new build is created and run automatically, compiling the source code for both Themis and Apollo.

This process has served our developers well, and it has enabled "Single Click Shipping": Commit your code, wait for CI, and press a single button to ship it.  However, as Apollo grew in size, so too did the amount time it took for CI to build our app. Soon after April, we reached our tipping point.

## Red Alert

In early May 2018, the Front End Infrastructure (FEI) team started receiving a stream of alerts in our Slack channel. Our team members were frustrated by the amount of time it was taking for CI to finish compiling, and a number of people even reported compilation failures.

Looking at BuildKite, we were greeted with a horrendous sight. Not only were most successful builds taking upwards of 20 minutes to compile, some were also failing after running for similar lengths of time. Developer frustrations aside, this was unacceptable because it hindered Clio's ability to respond effectively in emergency situations. For the FEI team, it was all-hands-on-deck.

After some initial investigation, we determined that the issue was caused by the depletion of RAM on our remote CI agents. In short, our CI agents have a limited amount of memory that is shared by a bunch of different processes. As Apollo grew in size, Webpack started to consume more and more memory. When it approached and exceeded the memory limit, it started to slow down and eventually fail. Webpack does not account for the entire compilation time, but it is the process that has grown the quickest and pushed us over the edge.

So how could we resolve this issue? If money was no object, we could have upgraded to instances with more RAM. However, we felt that our application's current complexity did not justify such an expense and the issue was most likely the result of poor optimization. As such, we got down to the business of optimizing our build.

## Webpack v4

We decided that the first step in the optimization of Apollo's compilation was to upgrade Webpack to v4. There were two reasons for this decision.

When Webpack v4 was released, its headline was [performance](https://medium.com/webpack/webpack-4-released-today-6cdb994702d4). As other folks upgraded their applications to the latest versions of Webpack, many started reporting significant performance improvements. We hoped, if we were lucky, that the upgrade could be the silver bullet we needed to get ourselves back in the green.

The second reason is more simple: we already had the Webpack v4 upgrade on our roadmap. Being a major version bump, it was possible that any optimizations we implemented while on Webpack v3 would be moot once the (eventual) upgrade was completed. Therefore, we agreed it was worth it to upgrade now, see what it gives us, and improve things from there.

Apollo was built with a blend of many different front-end technologies. CoffeeScript (for a few legacy features), TypeScript, AngularJS and Sass are the main ingredients in this blend, complemented with healthy dosages of many other libraries and tools. As a result, our Webpack configuration is rather heavy: we have at least 3 different build environments and use more than 20 loaders and 9 plugins.

With such a "mature" Webpack v3 set-up, the upgrade to Webpack v4 was not straightforward. The rest of this article will provide a description of the strategy we used to upgrade to the latest version, with an emphasis on the issues we encountered and how we went about resolving them.

## The Upgrade

The actual upgrading of Webpack itself is simple. We use [yarn](https://yarnpkg.com) to manage our Node modules, so, to upgrade to Webpack v4, we simply ran:

```
yarn upgrade webpack@v4.8.3
```

You'll notice that we specified the exact version. We do this with all of our Node modules to avoid any unintentional upgrades. Why `v4.8.3`? No particular reason, anything `v4.X.X` would have been fine, `v4.8.3` just happened to be the latest version available when we performed the upgrade.

Coupled with this upgrade, we also immediately upgraded `webpack-dev-server` to `v3.X.X` and added `webpack-cli`, which used to be a part of `webpack` but has been separated into another package in v4:
```
yarn upgrade webpack-dev-server@v3.1.4
yarn add -D webpack-cli@v2.1.4
```

## Are We Compiling Yet?

After the upgrade, our application was completely broken. This is where the actual work began. For the most part, the errors that we got were the result of outdated Webpack loaders/plugins, but we also found it necessary to adjust some configurations to meet with newer specifications. The following is a list of recommendations and interesting pieces of knowledge that we think will be useful for anyone attempting to upgrade their own Webpack project.

### One Step At a Time

First piece of advice: Do the upgrade one step at a time. Initially, our upgrade plan was to perform a systematic review of our entire Webpack set-up and preemptively resolve as many issues as we possible could. This involved, among other things, a complete review of all of our loaders and plugins to see if they were compatible with the new version of Webpack, if they needed an upgrade, or if they had been deprecated since v4's release.

We chose this strategy initially because we liked the idea of "doing everything right the first time." In practice, however, this actually led to us "doing nothing right the first time." The issue that we ran into is that, once we finished doing all the things we thought were necessary, and our application failed to compile, we were at a loss for what went wrong. The change set was simply too large, and we couldn't diagnose the issue.

So, as familiar as this advice is to many, it is nevertheless worth repeating here in the context of Webpack: Make each step in your upgrade as small as possible. Webpack actually lends itself nicely to this process, and we found the error outputs we got usually gave us a good idea of what the next step should be. In particular, we recommend the following iterative process for upgrading:

1. Run your Webpack compilation.
2. Investigate the cause of the errors.
3. Upgrade/reconfigure as necessary to resolve this one error.
4. Repeat 1 ~ 3 until Webpack compiles and everything works properly.

### Upgrading Plugins/Loaders

It is likely that a number of your plugins and loaders are broken as a result of the upgrade to Webpack v4. Webpack v4 has made a number of breaking changes to its API, and if any of your plugins/loaders are relying on an older interface that has been changed, they will cause errors during the Webpack compilation process.

A typical plugin error looks like this:

```bash
node_modules/fork-ts-checker-webpack-plugin/lib/index.js:143
    _this.compiler.applyPluginsAsync('fork-ts-checker-service-before-start', function () {
                           ^

TypeError: _this.compiler.applyPluginsAsync is not a function
```

And a typical loader error looks like this:

```bash
ERROR in ./client-src/themisui-vendor.scss
Module build failed: ModuleBuildError: Module build failed: TypeError: Cannot read property 'context' of undefined
    at getLoaderConfig (node_modules/fast-sass-loader/lib/index.js:72:29)
```

Webpack has really good stack traces when these types of errors occur, and they will usually tell you which loader/plugin the error is coming from. In the examples above, you can see that the culprits were `fork-ts-checker-webpack-plugin` and `fast-sass-loader` respectively.

Luckily, Webpack gave the community plenty of notice before pushing out the breaking changes in v4, so most loaders/plugins have released updates that are compliant with v4.

Our strategy was to search for the loader/plugin on Github, look at their release history, and try to find a release that explicitly states support for Webpack v4. If such a release existed, we would upgrade our package to the latest version.

If you run into a package that does not appear to be compatible with Webpack v4, and their release history does not have a compatibility update, check the Github issues page for the package! You may have come across a package that has not yet been made compatible with the newest version of Webpack.

Aside from issues that were resolved by simply upgrading a package, we also ran into a number of errors that were a little more unique. The following sections will describe three errors that we think a lot of people will also run into during their upgrade.

### CommonsChunkPlugin

One of the errors we got during our upgrade was this:

```bash
Error: webpack.optimize.CommonsChunkPlugin has been removed, please use config.optimization.splitChunks instead.
```

Like many other Webpack projects, we used the `CommonsChunkPlugin` to generated separate chunks for our third-party modules and the Webpack manifest file. These files tend to change less often than our bundled application file, so by separating them out, our clients' browsers can cache them separately and experience less-costly cache busts.

To accomplish this, we had the following set-up in our Webpack configuration file:

```javascript
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

In Webpack v4, the [CommonsChunkPlugin](https://webpack.js.org/plugins/commons-chunk-plugin/) was deprecated. Instead, Webpack has introduced a new configuration property, `optimization`, that handles various output optimization settings. Luckily, our usage of the `CommonsChunkPlugin` was fairly standard, and we were able to find a direct translation of our old configuration:

```javascript
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

```bash
Error: Chunk.entrypoints: Use Chunks.groupsIterable and filter by instanceof Entrypoint instead
    at Chunk.get (node_modules/webpack/lib/Chunk.js:712:9)
    at node_modules/extract-text-webpack-plugin/dist/index.js:176:48
```

This time, the error itself was not as important as its source. In this case, the culprit is `extract-text-webpack-plugin`. This is another one of those "documented breakages." In prior versions of Webpack, the `extract-text-webpack-plugin` was commonly used to lift CSS out of compiled JS bundles and into dedicated CSS asset files. We, too, used it for this purpose. The Webpack team now recommends using `mini-css-extract-plugin` for this role, but `extract-text-webpack-plugin` is still around as it has other use cases as well.

To deal with this error, you actually have two options. In the long-term, you probably want to use `mini-css-extract-plugin`. But, if you want to "park" that and do it in a later fix, you can actually upgrade `extract-text-webpack-plugin` to one of its beta releases. This is what we did:

```bash
yarn upgrade extract-text-webpack-plugin@v4.0.0-beta.0
```

Without changing any other plugins/configurations, this was sufficient for resolving this issue.

### UglifyJSPlugin

The last "special issue" we ran into while upgrading is this one:

```bash
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory
```

Out of context, this error could mean literally anything, so it was a bit harder for us to pin this one down. The hint to the solution was in the timing: we noticed that this error always arose as UglifyJS was running.

In previous versions of Webpack, you would use the `UglifyJSPlugin` by including it in the `plugins` array of your configuration object:

```javascript
plugins: [
  new webpack.optimize.UglifyJsPlugin({
    // ... Specify options for UglifyJs here
  }),
]
```

In Webpack v4, this syntax is no longer supported. In its place, the `optimization.minimizer` property now takes on the role of telling Webpack how it should optimize the size of output bundles:

```javascript
// Remember to require the plugin
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');

module.exports = {
  // ... Other configuration properties
  optimization: {
    minimizer: [ // This array tells Webpack which minimizers it should use
      new UglifyJsPlugin({
        // ... Specify options for UglifyJs here
      })
    ]
  },
}
```

In our production Webpack set-up, we found that we were still using the old syntax to include and configure `UglifyJsPlugin`. Thus, we removed this item from our `plugins` array and put our configurations under `optimization.minimizer` instead. This change, as it turns out, was enough to resolve our issue.

As a side note, thanks to the new `mode` options in v4, Webpack now performs uglification by default in `production` mode. How is this different from Webpack v3? Well, in v3, if you do not explicitly include `UglifyJsPlugin` in the `plugins` array, no uglification will occur on the output. However, in v4, if you are running in `production` mode, uglification will occur even if `optimization.minimizer` is not specified.

## Conclusion

After upgrading a bunch of loaders and plugins, and reconfiguring a small number of options, we finally managed to get Webpack to compile and our application running again.

**INSERT CELEBRATION GIF**

Ecstatic, we were eager to start benchmarking our compilation process and comparing it against our previous set-up with Webpack v3. And this is when joy turned into slight disappointment.

Don't get us wrong, we did see **significant** performance improvements, just as the Webpack team promised. When not limited by memory, Webpack v4 cuts down the compile time for Apollo by around 30 ~ 40%, which is :fire:. We also saw noticeable improvements in `--watch` recompilation times, giving us shorter code-compile-debug cycles.

However, we did not see a significant improvement in Webpack's memory usage. After performing some basic tests, we found that it was still running over the memory limit we have on our CI agents. This meant it was back to the drawing board for us.

Tune in to the next installment to see how we dealt with this memory issue.
