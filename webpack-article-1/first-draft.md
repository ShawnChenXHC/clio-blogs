# First Draft

This is the first draft for the article that describes the steps that we took
in response the Webpack issues that we faced.

This article is intended to be the first of a two-part series that covers, as
comprehensively as possible, the issues the front-end team here at Clio
experienced with our Webpack compilation process.

In this first part, we will explain:

1. What the issue we faced was
2. Our initial, somewhat naive attempt: Upgrading Webpack
   * Explain our upgrade path, what challenges we faced, what resources helped,
     and how we ensured that things were working
   * Explain the benefits we got from the Webpack upgrade
   * Explain how even with the benefits, it still in
3. Explain the benefits that we got from the Webpack upgrade
4. Explain how we were disappointed when the upgrade didn't yield as much as
   what we had hoped for.
5. Explain how we had to dig deeper, this would be second article.

## Introduction

Clio's main software offering received a revamp in 2017. Our legacy application, now dubbed Old Clio, is a Rails application that relied on the Rails templating engine. The revamp, codenamed Apollo, is a single page application compiled together using Webpack.

Our Production Engineering team have set up an automated system for building newer versions of Apollo. We use BuildKite as our continuous integration (CI) platform. The system is pretty slick: Whenever a developer checks code into our repository, a new build will be created and ran automatically, compiling together the source code and running tests as per dictated by our set up.

This process has served our developers well, and it has enabled "Single Click Shipping": Commit your code, wait for compilation, and press a single button to ship it. However, as the size of Apollo grew, so too did the amount of time it took for it to be built on CI. Soon enough, we reached our tipping point.

Before going any further, I would like to quickly mention who exactly we are. We are the Front End Infrastructure (FEI) team. Our mission is to create an amazing front-end development environment here at Clio. As a part of this mission, we are responsible for overseeing the usage and maintenance of various front-end technologies.

## Red Alert

In the beginning of May, the FEI team started receiving a stream of alerts on our Slack channel. People were being frustrated by the sheer amount of time it took to compile Apollo, a number of people even reported compilation failures.

Looking at BuildKite, we were greeted by a horrendous sight. Not only were most successful builds taking upwards of 20 minutes to compile, there were also a number of builds that failed after running for similar lengths of time.

Developer frustrations aside, this was unacceptable because it hindered Clio's ability to respond effectively to emergency situations. For the FEI team, it was all-hands-on-deck.

After some initial investigation, we hypothesized that the root cause of the issue was memory-related. In rough terms, this is what we hypothesized:

1. The AWS EC2 instance allocated for a build has 4GB of physical memory.
2. In the past, it was possible to run and complete all of the processes inside our compilation pipeline within this memory limit.
3. As the size of Apollo grew, so too did the amount of memory needed by Webpack in order to compile it.
4. As the memory on the instance was exhausted, the running processes began to experience performance issues: This is the cause of Webpack's slowness.
5. Eventually, there was simply not enough resources for Webpack to complete execution, leading to build failures.

So how can we resolve this issue? If money was no object, we could have upgraded to instances with more RAM. However, we felt that our application's current size did not justify such an expense and the issue was most likely the result of poor optimization. As such, we got down to the business of optimizing our build.

## Webpack v4

We decided that the first step in the optimization of our Apollo's compilation was the upgrade of Webpack to v4. There were two reasons for this decision.

When Webpack v4 was released, its headline was [performance](https://medium.com/webpack/webpack-4-released-today-6cdb994702d4). As other folks upgraded their applications to the latest versions of Webpack, many started reporting significant performance improvements. We hoped, if we were lucky, that the upgrade could be the silver that we needed to get ourselves back in the green. A little hopeful, yes, but why would we not want to take advantage of the hardwork the Webpack team has put into optimization?

The second reason is a little bit more plain. The FEI team already had the upgrade to Webpack v4 in its pipeline. Being a major version bump, it is possible that any optmizations we find/implement whilst on Webpack v3 will be moot once we upgraded. Therefore, we felt it better to upgrade now, see what it gives us, and improve things from there.

Apollo was built with a blend of many different front-end technologies. CoffeeScript, TypeScript, AngularJS and Sass are the main ingredients in this blend, complimented by healthy dosages of many other libraries and tools. As a result, our Webpack configuration is rather heavy: We have at least 3 different build environments and utilize more than 20 loaders and 9 plugins.

With such a "mature" Webpack v3 setup, the upgrade to Webpack v4 was not straightforward. The rest of this article will provide a description of the strategy we used to upgrade to the latest version, with an emphasis on the issues we encountered and how we went about resolving them.

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

After the upgrade, our application was completely broken. This is where the actual work begins. For the most part, the errors that we got were the result of outdated Webpack loaders/plugins, but we also found it necessary to adjust some configurations to meet with the new specifications of Webpack v4. In no particular order, the following is a list of recommendations and interesting pieces of knowledge that we think will be useful for anyone attempting to upgrade their own Webpack project.

### Set Your Mode

### One Step At a Time

Initially, our upgrade plan was to systematically go through every single
loader and plugin that we used and upgrade each and every one of them
regardless of whether or not they had been broken (We didn't bother checking.)
We also sought to perform a complete review of our Webpack configurations and
check them against Webpack v4's new specifications.

This turned out to be a really bad idea. After eagerly upgrading a bunch of
loaders and plugins, we ran into an issue and Webpack wouldn't compile. Since
we upgraded everything at once, it was really hard for us to find the root
cause. Thus, we recommend an "upgrade-if-necessary" strategy, with periodic
tests along the way.

Our "test-Google-debug" can be described as:
1. Run the Webpack compilation.
   * We run this in production mode to capture as many loaders/plugins issues
     as possible
2. Google the errors that we got
3. Upgrade/Reconfigure as necessary
4. Go back to step 1

### CommonsChunkPlugin

After this upgrade was complete, we decided to test the waters of our progress
by running a test compilation. This gave us the following error:

```
Error: webpack.optimize.CommonsChunkPlugin has been removed, please use config.optimization.splitChunks instead.
```

Like many other Webpack projects, we used the `CommonsChunkPlugin` in order to
generate separated chunks for third-party modules and the manifest file.
Because these files change less often than our application code, separating
them out gives our clients the ability to long-cache them, reducing the
initial load time of our app.

In Webpack v4, the `CommonsChunkPlugin` is deprecated. This is what we had in
our config:
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

Looking at the documentation for `config.optimization.splitChunks`, we removed
those items in our plugins array and added the following configuration:

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

### ExtractTextWebpackPlugin

Back to the top of the cycle, we ran Webpack compile again. Error again:

```
Error: Chunk.entrypoints: Use Chunks.groupsIterable and filter by instanceof Entrypoint instead
    at Chunk.get (/home/vagrant/clio/themis/node_modules/webpack/lib/Chunk.js:712:9)
    at /home/vagrant/clio/themis/node_modules/extract-text-webpack-plugin/dist/index.js:176:48
```

Once again, the error itself is not as important as the source of the error. In
this case, the culprit is `extract-text-webpack-plugin`. This is another one of
those "documented breakages." In prior versions of Webpack, the
`extract-text-webpack-plugin` was commonly used to lift CSS out of compiled JS
bundles and into dedicated CSS asset files. We, too, used it for this purpose.
Since Webpack v4, however, it has been superseded by `mini-css-extract-plugin`.

**? Should you talk about mini-css-extract-plugin ?**

Here, you actually have two options. In the long-term, you probably want to use
`mini-css-extract-plugin`. But, if you want to "park" that and do that in a
later fix, you can actually upgrade `extract-text-webpack-plugin` to one of its
beta releases. This is what we did:
```
yarn upgrade extract-text-webpack-plugin@v4.0.0-beta.0
```

### UglifyJSPlugin

Back to the top of the cycle, we ran Webpack compile again. Error again:

```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory
```

This seems to have occurred as UglifyJS was running.

So, in previous versions of Webpack, you would use and configure
`UglifyJSPlugin` by including it into the `plugins` array of the configuration
object. We did just the same:

```js
plugins: [
  new webpack.optimize.UglifyJsPlugin({
    // ... configuration
  }),
]
```

In Webpack v4, however, the configuration of `UglifyJSPlugin` is accomplished
through `optimize.minimizer`. In addition, `UglifyJSPlugin` is included and ran
on default options when the `mode` is `"production"`. Therefore, we deleted it
from our `plugins` array. Once this was done, lo and behold, the error was gone.

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

## Here come the tests

Karma is the chosen test runner for New Clio. We use `karma-webpack` to
better integrate our test suite with the rest of our application. When we wish
to run our tests, Karma uses Webpack as a preprocessor. All of our tests, along
with all of our application code, are processed and bundled into one JS file.
Once this is done, the tests begin running automatically.

This integration has given us a lot of benefits: Our test code is written just
like our source code since it gets passed through the same set of loaders, and
our test environment closely mirrors production since the Webpack configuration
for both shares the same base configs.

When we upgraded Webpack to v4, we also had to make sure that this integration
was still operational. So we ran Karma and got this error:

```js
ERROR [config]: Invalid config file!
  Error: webpack.optimize.CommonsChunkPlugin has been removed, please use config.optimization.splitChunks instead.
```

What happened? Well, like I've mentioned, our shared Webpack configuration file
used to include the `CommonsChunkPlugin`. However, `karma-webpack` has a known
[compatibility issue](https://github.com/webpack-contrib/karma-webpack/issues/22)
with this plugin, so when we generated the test configuration from the shared
file, we had this iterator in place to strip out `CommonsChunkPlugin`:

```js
testConfig.plugins = sharedConfig.plugins
  .filter((item) => {
    return !(item instanceof webpack.optimize.CommonsChunkPlugin);
  });
```

Unfortunately, with the upgrade to Webpack v4, the mere mentioning of the
`CommonsChunkPlugin` is now grounds for error. Therefore, once we removed this
`filter()`, the error was gone.

Next, do not forget to make sure that your test configuration settings are
actually being consumed by Karma. We ran into an issue with this because our
`karma.conf.js` looked like this:

```js
const webpackConfig = require("config/webpack/test.js");
module.exports = (config) => {
  config.set({
    // ... other configurations
    webpack: {
      module: webpackConfig.module,
      output: webpackConfig.output,
      resolve: webpackConfig.resolve,
      plugins: webpackConfig.plugins,
    },
  });
}
```

When Karma actually calls upon Webpack, the options it uses will be only those
specified within the Karma options file. This meant things that weren't defined
explicitly will revert to Webpack's defaults, for example, `mode` will be set
to the default of `"production"`.

### Tidying Up

Lastly, we made some further adjustments to make our Webpack process better for
everyone. Essentially, we are taking advantage of the benefits that Webpack
provides.
