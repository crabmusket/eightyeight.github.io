---
layout: post
title:  "Containerising a Laravel application"
date:   2015-11-14 17:50
tags:   docker laravel php
---

I recently joined a startup as cofounder, and took over responsibility for our technology.
The prototype of our software was a Laravel application which was deployed behind Apache on AWS.
After I got familiar with Laravel, and suffered through a few deployment cycles involving `git pull`ing to the production server, I dediced to make an executive decision and reroll our infrastructure using containers.

There were two particular experiences that prompted this decision.
While I always had ambitions for us to scale up using containerised infrastructure, via CoreOS or whatever other solution we decided was appropriate,  these two infrastructure problems we overcame were something we never wanted to repeat.

First, we decided to deploy an ancillary service using nodejs on the same server, and expose it via a subdomain.
The experience of confuguring Apache to correctly reroute traffic on the subdomain, as well as keeping our existing Apache tweaks intact (particulary, redirecting HTTP traffic to HTTPS) took a very long night of tiol and pain.
Neither of us are experienced Apache users, but we struggled through and eventually got it working.

Second, we added to our software a PDF report generator, using [this excellent package][barryvdh] to do so.
The first issue we ran into was actually trying to get `wkhtmltopdf` binaries onto the production server.
Because we deployed to Amazon Linux, but both developed on Ubuntu, we were caught out when the package we were using to provide binaries couldn't find them in production.
When it came time to deploy, we then had difficulty sequencing Composer commands to install the binaries and facades atomically, and had to do a strange git commit shuffle to first install the packages, then add the facades that depended on them.

I'm fully aware that both of these issues boiled down to unfamiliarity with the tools we were using, specifically: Apache, Composer, and Amazon Linux.
The containerised app was a way to sidestep these issues.
For example, deploying in a container meant that the application would be completely isolated from changes in OS in development and production: as far as Laravel is concerned, everywhere looks like Debian.

The process we eventually worked out to stuff this application into a container may be suboptimal, and I'm sure there are some edge cases we're missing, or long-cuts we're taking that could be avoided.
However, it's a step up on our old process, and for that I'm thankful.
It's also a step towards a vastly more scalable infrastructure, which is almost a reward in its own right for a coder that takes pride in their work.

Now you know the backstory, let's get into the fun stuff.

[barryvdh]: 

## Due diligence and requirements

I read up on Laravel and containers before attempting our own version, to see if there was an easy guide we could follow.
The most in-depth article I could find [recommends](http://www.dylanlindgren.com/docker-for-the-laravel-framework/) splitting the workload between no fewer than three separate long-lived containers, plus two more on-demand containers for running Composer and Artisan respectively.
I didn't want to go that far; I can't judge whether the approach is superior for scaling, but it definitely seemed like it would make development more of a headache.

At the other extreme, [this project](https://github.com/mtmacdonald/docker-laravel) seems to follow in Homestead's footsteps of sticking everything inside the container, including MySQL and beanstalkd just in case they're needed.
While this would have been a much simpler way to mgirate to a container immediately, I'm definitely a fan of small containers with a single responsibility.
While I'm not strict enough to really care about running Composer in the same container as Apache, I definitely didn't want to run MySQL inside a container.

I'm also a fan of the immutable container ideal, which means we would never run Composer or Artisan tasks in production anyway.
They would be reserved for running in development, and the build process would then create a container by running all the necessary incantations to set up the app's dependencies from scratch.

Additionally after reading [the 12-factor app][12factor], and in a significant departure from the usual Laravel way of doing things, I decided we would also need a way to configure details like database hosts at runtime (i.e. through environment variables), rather than in the application source.

With all this in mind, I decided that we'd just forge our own path and learn along the way.
Here are the concrete requirements I settled on for our solution:

 * Run Apache inside the container
 * Make Composer and Artisan available inside the container for use in development
 * Allow mounting the current directory into the container for live editing in development
 * Allow configuration via environment variables

[12factor]: 
