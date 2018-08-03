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




Our Production Engineering Team (PEng) has set up an automated system for
building newer versions of Clio. We use BuildKite as our continuous integration
(CI) platform. This process can roughly be described as follows:

1. On BuildKite, we have configured a pipeline called `App-Compile`
2. When developers merge their commits into the master branch of our
   application repo, BuildKite will automatically create a new build within the
   `App-Compile` pipeline.
3. An AWS EC2 instance is allocated for this build, and BuildKite will begin to
   use it to run the commands that were specified.
4. BuildKite checks out the commit that triggered the build and performs a
   number of preparatory operations.
5. When ready, it runs  
   ```
   bundle exec rake assets:clean assets:precompile
   ```
   based on the code in this commit, compiling the assets for both Old and New
   Clio.
6. When the compilation is finished, the assets are compressed and uploaded;
   thus completing the build process.



 1. The AWS EC2 instance allocated for a build has 4GB of physical memory.
 2. In the past, it was possible to run and complete all of the processes inside our compilation pipeline within this memory limit.
 3. As the size of Apollo grew, so too did the amount of memory needed by Webpack in order to compile it.
 4. As the memory on the instance was exhausted, the running processes began to experience performance issues: This is the cause of Webpack's slowness.
 5. Eventually, there was simply not enough resources for Webpack to complete execution, leading to build failures.

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
