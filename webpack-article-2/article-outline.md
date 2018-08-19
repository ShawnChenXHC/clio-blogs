# Article Outline

I may not have the time and capacity to complete this article, but I would like to make a start on it none the less. Partly (Mostly) because I need this for my school credits.

In the first article, I briefly described the problems that we were experiencing on our CI and talked about what we suspected was the cause of the issue. After, I quickly transitioned away from this topic and focused instead on giving a practical set of tips and and recommendations to help people upgrade their Webpack projects from v3 to v4.

I promised that I will be diving more into the actual problem that we faced in this latter article, and that is exactly what I plan to do.

The following is an overview of the sections that I think will be in this article:

1. Problem analysis
   * This is a more in-depth description of our CI set-up and the problem that we faced.
   * I want to describe:
     * Exactly what our CI system does when it runs our app's compile
     * What our threshold is, what we wanted Webpack to limit itself to, and what it is currently at
     * What Webpack does during compilation, the processes that were ran
2. The Blackbox approach
   * In this approach, we treat our Webpack build as a black box and tried to measure it from the outside
   * Tools used:
     * `ps`, `top`
     * Node script that tracks memory usage over time
   * Key takeaways:
     * You can place "soft limits" on the memory with Node options, to do so, you cannot use the binary provided by Webpack, you have to call Node yourself
     * These "soft limits" are created by the Node option `--max-old-space-size`
     * 1gb is probably a pretty good limit to have for most people, but best to adjust according to project
 3. The Whitebox approach
    * In this approach, we peek right into Webpack and see exactly what our build is made up of
    * Tools used:
      * Script from before
      * Node + Chrome dev tools debugging goodies
      * Isolation
    * Key takeaways:
      * Memory spikes hard in plugins stage
      * Be careful with your SCSS
      * Be weary of parallel processes
4. It's a Rails thing
   * This is a more niche piece of advice: If your application is a Rails application that uses the Webpacker gem to integrate Webpack with the Rails asset pipeline, you may find it useful to split Webpack out of the asset pipeline and compile it independently
5. Conclusions

Timeline of events:

1. OOM error forces FEI to divert attention
2. Agent is upgraded to provide additional breathing room
3. NodeJS is upgraded to 8.9.4 as pre-requisite for Webpack v4
4. Webpack is updated to v4
5. Investigation is performed, learnings about profiling Webpack which leads to small improvements such as the reduction of compile stylesheets. But unfortunately, we couldn't find a direct way to decrease the amount of memory used
6. Found that Rails and Webpack ran at the same time, both consuming memory, when they could in fact be ran synchronously; decoupled the two and found that we were within threshold again
7. Agent is downgraded again, immediate OOM errors but these were "soft," found that they were caused by parallel UglifyJS, disabled that, error resolved
8. CI now running smoothly on lower capacities
9. About 10 days later, issue returns (Maybe don't mention this)
