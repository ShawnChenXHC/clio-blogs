# Article Outline

What follows is an outline for the article I wish to write to describe the
Webpack problems we faced a couple of months ago.

In essence, this was the problem:
* We were experiencing slowness on CI for compiling Webpack
* We found Webpack was consuming a lot of memory, and this consumption of
  memory was leading to slowness
* Eventually, Webpack required so much memory in order to run that essentially
  every run was failing because of OOM errors.

The impact of this is simply stated: We couldn't ship out code.

What did we do?

First, we had to get some breathing room for ourselves. With the help of our
Production Engineering team, we were able to temporarily expand the total
amount of memory available to our CI agents and given a week to address the
issue.

Second, we decided to upgrade Webpack to v4. We heard good things about the
performance upgrades Webpack v4 provides, and hoped it would be a silver
bullet for our issue. We encountered a myriad of different issues during the
upgrade, but most of these were the artifacts of Webpack plugins that weren't
compatible with v4.

Reference: https://themis.atlassian.net/browse/CLIO-58985

But it turned out upgrading Webpack wasn't the issue. We had to dig deeper.

Third, we started performing diagnostics, to try to pinpoint exactly what parts
of the Webpack compilation process are causing us the greatest pains. We
performed two forms of diagnostics:

First, we leveraged Node's `--inspect-brk` option and used Chrome dev tools to
monitor Webpack memory consumption in real time as it ran.

This led to the surprising discovery that there was a particular set of CSS
what was being included repeatedly for no good reason. Getting rid of this
gave us notable improvements in memory consumption, time to compile, and size
of bundles created.

References: https://themis.atlassian.net/browse/CLIO-62917 https://themis.atlassian.net/browse/CLIO-63278

However, this still wasn't enough. Our next form of diagnostic was to use
`ignore-loader` to isolate assets by type and see what effects individual types
of files had on our compilation.

References: https://themis.atlassian.net/browse/CLIO-63126

We discovered that, aside from TypeScript, Sass and CoffeeScript files had the
greatest impact on our compile. Even still, because we couldn't simply just
drop these assets, there wasn't much we could do with these statistics.

Ultimately, the action that brought us back under the limit was decoupling
Webpack from the Rails asset pipeline. This way, both processes run one after
the other, instead of at the same time.

References: https://themis.atlassian.net/browse/CLIO-63244

So, lessons learned?
1. How to upgrade to Webpack v4
2. How to use Node and Chrome to see memory consumption
3. How to prevent CSS duplication (When using Sass)
4. How to decouple Webpack from Rails asset pipeline
