# Wrangling with Webpack - pt.2 - On the Subject of Memory

*Working Draft*

**Link to Article 1**

## Introduction

This is the second half of a two-part series on how we use Webpack here at Clio. Last time, I talked briefly about the problems we faced running Webpack on our CI provider before transitioning over to discussing our upgrade path to Webpack v4. The narrative then ended on a bit of a cliff-hanger as I noted that the upgrade did not immediately solve our problems, so I would like to amend that in this article. Welcome to Wrangling with Webpack 2 - :lightning: Electric Boogaloo :lightning:.

## Can You Repeat the Problem?

To set the stage, I would like to begin this article by reiterating the issue that was introduced in the last article.

We use Buildkite as our CI provider, and AWS EC2 instances to actually run our CI processes. When a developer pushes a new commit up to our Github repo, Buildkite is notified via webhooks and will create a new build for this commit in each of our CI pipelines.

We have multiple pipelines for doing all sorts of stuff, but the focus of this article is on the compile pipeline, which is used to automatically compile our front-end assets. When Buildkite creates a build, it will allocate an AWS EC2 instance and install an "agent" on this machine. The agent is basically a script that will begin running the processes defined in the pipeline on the machine it was installed on:

1. It clones the Github repo and checks out the commit that triggered the build
2. It performs a number of preparatory tasks such as installing Node modules
3. It executes `rake assets:precompile`, which is the Rails command for kicking off compilation in the Rails asset pipeline
4. This compiles all of the assets for legacy Clio and, because we use the [Webpacker](https://github.com/rails/webpacker) gem, also starts Webpack to compile new Clio (Henceforth referred to as Apollo)

This pipeline worked great, until it didn't. Our AWS EC2 instances are of type `c5.large`, which comes with 4GB of memory. Because of background processes, by the time `rake assets:precompile` begins, there is only around 2GB of memory available to be shared among the "actual" compilation processes. As the size of Apollo grew, Webpack started demanding more and more memory. Eventually, as Webpack was starved of the memory it needs, we started experiencing excruciatingly slow builds with intermittent failures.

### Temporary Breathing Room
