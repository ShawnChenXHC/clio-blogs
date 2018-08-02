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
