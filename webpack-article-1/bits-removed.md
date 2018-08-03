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
