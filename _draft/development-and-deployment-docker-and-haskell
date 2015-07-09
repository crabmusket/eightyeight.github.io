---
layout: post
title:  "Development and deployment: Docker and Haskell"
date:   2015-05-23 13:10
tags:   coreos docker aii
---

This post was prompted by a [recent discussion](https://www.reddit.com/r/haskell/comments/3bmzn8/how_can_i_improve_build_time_especially_on_docker/) about using Docker for Haskell development.
The question was: how do I develop quickly using Docker, and avoid waiting for the container to be rebuilt all the time?
My answer is - for a compiled language like Haskell - that you should be separating your development and deployment.

Now, this does sound odd, given that Docker was supposed to be the great equaliser between development and production environments.
Docker sells itself on the idea that when we deploy a container, we're deploying exactly what we wrote and tested on.
There are two disclaimers I'd like to make about this before I continue:

 * For compiled applications, environment matters less than applications that depend on a language runtime.
   Once I have a binary, I can be a lot more lax about the environment I deploy it to.
 * You should be testing the containers that you deploy to production, even if you don't necessarily develop in them.
   There's no reason to deploy a container with your development tools in it.
   But once you've build your 'production' container, you should definitely test it locally and in staging before shipping it.

With those points in mind, here's my process for using Docker for developing and deploying Haskell.

# Development

I'm pretty frugal in my development environment, which makes this workflow simpler.
When I want to start a new Haskell project, I spin up one of the standard containers available on the Docker Hub:

    $ docker run -ti --rm -v $(pwd):/code haskell:7.10 bash
    # cd code
    # cabal update
    # cabal sandbox init
    # cabal init

And so on.
Notice that I use the `--rm` flag; this removes the container immediately after I close the bash session I'm using.
I like to make sure all my containers are stateless, and never leave containers about that I'm not actively working in.
Cabal sandboxes let me do this, by keeping my entire build inside the mounted `/code` volume.
This way, next time I spin up a fresh container, I don't have to reinstall.

Note that I _do_ have to run `cabal update` again, if I want to install more libraries in a fresh container.
This is a slight annoyance, but I usually do it in that foggy minute right after I re-open my source files and try to get my head bask where I left it.

# What about `yesod devel`?

Ah yes.
Some projects might require specific binaries installed, like the `yesod` set of tools, or even your favourite editor, `ghc-mod`, etc.
I do most of my Haskelling without any such tools, but if I _were_ going to use them, I'd make my own container to do so, which I could use in place of the vanilla container:

    $ docker run -ti --rm -v $(pwd):/code eightyeight/haskell:7.10-yesod bash

This container would be built from a Dockerfile in its own directory elsewhere, not from this current project's Dockerfile.
I'll admit this is a little bit of a weak solution - for instance, it'd be better to ensure that the version of `yesod devel` was the correct one specified in your project's Cabal file.
My _tentative_ recommendation for this is to try to install all binaries in the Cabal sandbox, and set your path so that you can run them as usual.
I've never tried it for Haskell, so I can't make any specific recommendations there, but it's how I work with Node projects.

# A simpler way?

At this point if you're familiar with Docker at all, you might have realised that the answer to needing to rebuild the container constantly is usually to stop changing your Dockerfile all the time.
After all, Docker is really great at using cached layers when your files haven't changed.
If you simply order the commands in your Dockerfile correctly, you should be able to avoid pretty much all the pain!
