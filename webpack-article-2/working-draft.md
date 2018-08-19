# Wrangling with Webpack - pt.2 - On the Subject of Memory

*Working Draft*

**Link to Article 1**

## Introduction

This is the second half of a two-part series on how we use Webpack here at Clio. Last time, I talked briefly about the problems we faced running Webpack on our CI provider before transitioning over to discussing our upgrade path to Webpack v4. The narrative then ended on a bit of a cliff-hanger as I noted that the upgrade did not immediately solve our problems, so I would like to amend that in this article. Welcome to Wrangling with Webpack 2 - :zap: Electric Boogaloo :zap:.

## Can You Repeat the Problem?

To set the stage, I would like to begin this article by reiterating the issue that was introduced in the last article.

We use Buildkite as our CI provider, and AWS EC2 instances to actually run our CI processes. When a developer pushes a new commit up to our Github repo, Buildkite is notified via webhooks and will create a new build for this commit in each of our CI pipelines.

We have multiple pipelines for doing all sorts of stuff, but the focus of this article is on the compile pipeline, which is used to automatically compile our front-end assets. When Buildkite creates a build, it will allocate an AWS EC2 instance and install an "agent" on this machine. The agent is basically a script that will begin running the processes defined in the pipeline on the machine it was installed on:

1. It clones the Github repo and checks out the commit that triggered the build
2. It performs a number of preparatory tasks such as installing Node modules
3. It executes `rake assets:precompile`, which is the Rails command for kicking off compilation in the Rails asset pipeline
4. This compiles all of the assets for legacy Clio and, because we use the [Webpacker](https://github.com/rails/webpacker) gem, also starts Webpack to compile new Clio (Henceforth referred to as Apollo)

This pipeline worked great, until it didn't. Our AWS EC2 instances are of type `c5.large`, which comes with 4GB of memory. Because of background processes, by the time `rake assets:precompile` begins, there is only around 2GB of memory available to be shared among the "actual" compilation processes. As the size of Apollo grew, Webpack started demanding more and more memory. Eventually, as Webpack was starved of the memory it needs, we started experiencing excruciatingly slow builds with intermittent failures.

## The Blackbox Approach

Now that the problem has been clearly identified, let us begin talking about the solutions.

I would like to introduce the first approach we took as the "blackbox" approach, in reference to the blackbox method of testing that many programmers are familiar with. In this approach, we treated our Webpack process as a "black box" that can only be examined and controlled from the outside.

Let me explain.

No matter how complex your Webpack project may be, Webpack is still just a Node program that must conform to all of the normal rules that apply to any other Node program.

This means two things. First, when Webpack is running, you should be able to examine it using the `ps` utility. Second, because it is a Node program, you should be able to configure the Node process that runs it using standard CLI options, such as `--max-old-space-size`.

Before going further, I would like to mention that for this analysis, we temporarily removed all parallelized processes from our Webpack set-up, such as parallel uglification and `fork-ts-checker`. This reduced the complexity of our analysis and you may find it beneficial to do the same. With that said, let's dive into the details.

To begin our analysis, we set up a command in our `package.json`:
```
"build:webpack": "NODE_ENV=production webpack --progress --config ./config/webpack/production.js",
```

With this, when we run `yarn build:webpack` from the root of our project, we should be running a compilation with configurations identical to what Webpacker uses to start Webpack during `rake assets:precompile`.

With Webpack running, we opened up another terminal window and executed:
```
ps ufaxww
```

`ps` is a Unix utility for printing out a snapshot of all active processes. By passing `ufaxww`, the output that will not only show the details (Including memory usage) of all active processes, but also group them together according to parent-child relationships.

For example, here is the output we saw when we ran this command, truncated to focus on Webpack-related processes:

```sh
vagrant  28922  2.0  1.0 1138236 64668 pts/0   Sl+  21:40   0:00 node /usr/local/bin/yarn build:webpack
vagrant  28932  0.0  0.0   4300   812 pts/0    S+   21:40   0:00  \_ /bin/sh -c NODE_ENV=production webpack --progress --config ./config/webpack/production.js
vagrant  28933 88.7  7.1 1550560 435704 pts/0  Rl+  21:40   0:15      \_ node /home/vagrant/clio/webpack/themis/node_modules/.bin/webpack --progress --config ./config/webpack/production.js
```

The sixth column from the left represents the amount of physical memory being consumed by the process on that line. The first two lines are actually generated by `yarn`, which we were using to execute the `build:webpack` command. The last line is the actual Node process that is running Webpack. As you can see, when this snapshot was taken, Webpack was consuming roughly 436MB of memory.

Now that we had a sense of what is actually running when Webpack begins, we got rid of the extraneous output and asked for a summary statistic instead:
```
ps uax | grep webpack | awk '{s+=$6} END {print s}
```

This command will filter down the output from `ps` to only lines that contain the string "webpack". It then adds together the sixth column from each row and prints it out. In effect, running this command gave us a snapshot of how much memory Webpack was using in total. Depending on your situation, you may find it necessary to use a different filter, but for us, `grep webpack` was sufficient.

But a single snapshot is not all that useful. So we made a Node script:

```js
// webpack-memory-csv.js
var process = require("process");
var childProcess = require("child_process");

var repeating = setInterval(
  function() {
    childProcess.exec(
      "ps uax | grep webpack | awk '{s+=$6} END {print s}'",
      function(_err, stdout) {
        process.stdout.write(stdout.trim() + ",\n");
      }
    )
  },
  100
);

const webpack = childProcess.spawn("yarn", ["build:webpack"])
  .on("close", () => {
    clearInterval(repeating);
  });

webpack.stderr.on("data", (data) => {
  process.stderr.write(data);
});
```

We placed this script at the root of our project. When ran, this script starts an interval timer that outputs the total amount of memory used by Webpack every 10th of a second. "Simutaneously," it also starts a new Webpack build by running the same command we put into our `package.json`. And here's the kicker: the output in CSV format. So, if we pipe the output into a file, we get a CSV file that can be used by all of your favorite graphing applications.

We ran the script three times and uploaded the files to [plot.ly](https://plot.ly) to create this graph. :chart_with_upwards_trend:

![Webpack Memory Plot Baseline](assets/plot-baseline.png)
https://plot.ly/~XiaoChenClio/16/

As you can see, with our current configuration, Webpack finishes compilation in around 110 ~ 120s. At the very most, our Webpack compilation appears to use about 1.4 million Kilobytes of memory (Or, 1.4GB).
