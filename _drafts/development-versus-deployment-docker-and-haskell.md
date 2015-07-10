---
layout: post
title:  "Development versus deployment: Docker and Haskell"
date:   2015-07-10 12:54
tags:   docker haskell
---

This post was prompted by a [recent discussion](https://www.reddit.com/r/haskell/comments/3bmzn8/how_can_i_improve_build_time_especially_on_docker/) about using Docker for Haskell development.
The question was: how do I develop quickly using Docker, and avoid waiting for the container to be rebuilt all the time?
My answer is that you should develop in one container, and deploy an _entirely different_ container.

Now, this does sound odd, given that Docker was supposed to be the great equaliser between development and production environments.
Docker sells itself on the idea that when we deploy a container, we're deploying to the exact environment that we developed and tested in.
There are two comments I'd like to make about this:

 * For compiled applications, environment matters less than applications that depend on a language runtime.
 * You _should_ be testing the containers that you deploy to production, even if they're not the containers you did development in.

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

## What about `yesod devel`?

Ah yes.
Some projects might require specific binaries installed, like the `yesod` set of tools, or even your favourite editor, `ghc-mod`, etc.
I do most of my Haskelling without any such tools, and I edit in my host, not inside the container.
But if I _were_ going to use them, I'd make my own container to do so, which I could use in place of the vanilla container:

    $ docker run -ti --rm -v $(pwd):/code eightyeight/hypothetical-yesod-container:7.10 bash

This container would be built from a Dockerfile in its own directory elsewhere, not from this current project's Dockerfile.

I'll admit this is a little bit of a weak solution - for instance, it'd be better to ensure that the version of `yesod devel` was the correct one as specified in your project's Cabal file.
My _tentative_ recommendation for this is to try to install all binaries in the Cabal sandbox, and set your path so that you can run them as usual.

I've never tried this in a Haskell project, so I can't exactly sell this as a complete solution, but this is the approach I take with Node projects.
I avoid installing (e.g.) `gulp` globally, and instead create `npm` scripts that run `gulp` from the local `node_modules` folder.

## Just write your Dockerfile properly!

At this point if you're familiar with Docker at all, you might have realised that the answer to needing to rebuild the container constantly is usually to stop changing your Dockerfile all the time.
After all, Docker is really great at using cached layers when your files haven't changed.
If you simply order the commands in your Dockerfile correctly, you should be able to avoid pretty much all the pain!

The [official Haskell image](https://registry.hub.docker.com/_/haskell/) readme shows you how to do this.
You first add your Cabal file to the project, which creates a layer:

    ADD example.cabal /code/snap-example.cabal

You then install all dependencies, creating another layer:

    RUN cd /code && cabal install --only-dependencies -j4

This will only re-run after you change your Cabal file.
Note that obviously Docker can't actually analyse the contents of your Cabal file and determine whether you've changed your dependencies, but it'll notice that the file is different, and that will invalidate the cache and re-run the install command.

Then, finally, you add the rest of your application code, and compile it:

    ADD . /code
    RUN cd /code && cabal install

This structure maximises your Docker cache usage, by ensuring that you only re-create the final layer when you change your code.
However, even though this does provide an advantage, I still think we should go further when building applications for deployment.

# Deployment

As I've started to deploy Docker containers others have built, I'm starting to run into many cases of containers, particularly ones packing Go applications, that are based on the `golang` official container.
I've seen this with Haskell containers, too, and the advice in the previous section (straight from the official readme) certainly suggests that the first line of your application's Dockerfile should be `FROM haskell`.

I contend that this is bad container etiquette.
When I go to install your application, I've now just downloaded, in the case of a Haskell application, 700MB.
This includes an operating system, GHC, its attendant infrastructure, and also your application's source code.
But your application is actually just a single binary that's probably less than 70MB.

I firmly believe that for deployment containers, you should not be starting with `FROM haskell`, but from `haskell-scratch`.
[haskell-scratch](https://github.com/snoyberg/haskell-scratch) is a cute little project that lets you create a minimal Docker container with only the shared libraries needed to run Haskell binaries compiled with GHC.
This means that you can start with a Docker container that's essentially empty and add nothing but your executable to it.

The workflow is slightly more complex.
To start using `haskell-scratch`, you'll need to compile it yourself locally, as there's no Docker Hub image.
However, this is as simple as cloning [the repository](https://github.com/snoyberg/haskell-scratch) to your computer and running `make`.

This will create two Docker images in your local Docker database: `haskell-scratch:integer-simple` and `haskell-scratch:integer-gmp`, containing linkages to the two different big-int libraries GHC can use.
Deploying an application using this library is as simple as this Dockerfile:

    FROM haskell-scratch:integer-simple
    COPY dist/build/example/example /bin/example
    ENTRYPOINT ["/bin/example"]

In fact, I did this exact thing in my [small server example](https://github.com/eightyeight/srvr).
I needed a tiny lightweight demo server which I could configure from the command-line, and I didn't want to wait for all of GHC to download.

To build your application, you must first compile the executable from your development Docker container (the one that _is_ based on `haskell`), then run `docker build` outside that container to copy the binary into the deployment image.

I really think this is an advantageous workflow.
I'd love to hear the opinions of more experienced Haskellers and Docker-ers.
I haven't thought about how [stack](https://github.com/commercialhaskell/stack) will impact this workflow; for me, my current setup is fine for small projects, but I have no doubt that this approach will start to break down on larger projects, multiple binaries, etc.

# Prior art

I should acknowledge two main influences on this post.
First and foremost, the FP complete article [a Haskell web server in a 5MB Docker container](https://www.fpcomplete.com/blog/2015/05/haskell-web-server-in-5mb), which introduced the `haskell-scratch` project.
And secondly, [this great article](http://blog.xebia.com/2014/07/04/create-the-smallest-possible-docker-container/) about using `scratch`, the _actually empty_ Docker container, to deploy Go applications.
