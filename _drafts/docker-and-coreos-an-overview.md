---
layout: post
title:  "Docker and CoreOS, an overview in pictures"
date:   2015-05-23 13:10
tags:   coreos docker aii
---

The world of containerised deployment is a little opaque at first.
Over the last month or two, I've been trying to introduce myself to it.
It wasn't easy at first, and information is pretty scattered, but I feel I've learned enough that I can pass some knowledge on to others who are also interested and don't know where to start.

I haven't had a ton of experience at ops _without_ containers, but since I've been using Docker as a super easy way to spin up development environments, I figured it was a natural choice to stick with them right on through to deployment.
However, I wanted to do it _right_, and the CoreOS philosophy appealed to me the most.
It seems like a technology that will grow with me - from a single server to host my blog on, all the way up to a cloud cluster running a complex application.
CoreOS comes with its own ecosystem and workflow, making it an even taller order to learn on top of Docker.

This post is intended to be the first of several describing my adventures in infrastructure.
I'l be writing these as I learn things, so expect them to be filled with mistakes, misunderstandings, missteps and herrings of all colours.
Today, we'll go through a broad overview of the various components involved in deploying applications with Docker and CoreOS.

## Motivations

Why are you here?
Why am _I_ here?
Essentially, I want to learn how to develop and deploy web applications reliably, conveniently, and at scale.
I assume you do, too.
I also assume that, like me, you don't know tons about traditional deployment or containerised deployment.

With all that in mind, there are four broad topics I'll cover in this post:

 * Linux containers: like process isolation on steroids
 * Docker: a tool for creating and managing linux containers
 * CoreOS: a minimal Linux distribution designed to run containerised applications
 * Fleetd and Etcd: utilities that ship with CoreOS

Hopefully this will put you in a good position to read more detailed posts about all of these topics with some high-level understanding of the whole system and the various pieces involved.
Throughout this post, I'll link to the best other pieces of writing I've discovered on various subjects - the ones that really helped me, and the ones I had trouble with until I had some basic understanding of other parts of the system.

## Anatomy of a cluster

Here's a diagram of a simple two-machine 'cluster' using CoreOS and Docker.
Don't worry if it doesn't make any sense right now - by the end of this post, it should!

![Two-machine cluster](https://cloud.githubusercontent.com/assets/904269/7782873/983250ba-016f-11e5-88c5-299dcd82b306.jpg)

The big rectangles are physically separate computers - of which two represent cloud machines running CoreOS, and one represents my local computer.
The little blue rectangles are services running on each machine.
Finally, the yellow rectangles are containers, each of them running some sort of process, whether it be an application, database, etc.

The lines represent some form of communication - networked or otherwise.
(I haven't been very precise with these lines - there should be a lot more of them, but I've just tried to give you the general gist of the communication that goes on!)
We'll refer back to this diagram at the end of this post, and hopefully it will make a lot more sense!

## Containers

You can think of a container as a lightweight virtual machine.
_Really_ light.
In terms of what's in the container, it might be anywhere from a whole OS (like a traditional VM) to [nothing but your application executable][docker-golang-scratch].
At runtime, they work essentially like any other process, except they're sandboxed so that they can't touch your regular OS files, only the files that are inside the container.

![Shipping containers](http://www.lighthome.com.au/Images/How%20do%20I/Shipping%20Containers/shipping-containers.jpg)

The only dependency the container has is on your kernel, which makes them super-portable.
You can take a container built on any machine, and run it on pretty much any other machine.
(With caveats, of course - for example, you can't run containers natively on Windows currently, because they're a Linux technology.
They're not _that_ magical.)

[docker-golang-scratch]: http://blog.xebia.com/2014/07/04/create-the-smallest-possible-docker-container/

## Docker

Docker is a tool for creating images and running them in containers.
Docker images are created using a Dockerfile, and they capture the state of a filesystem, like the 'disk images' that are stored when you run a VM.
Unlike VM disk images, Docker tracks each change you make to an image when you build it almost like commits in a git repository, giving each 'layer' a unique ID.
This has a couple of implications:

 * When you edit the contents of an image (for example, your application code), only layers _after_ those contents are added must be rebuilt.
   This saves quite a bit of time when you're creating images!
 * Layers can be shared between images - for example, a Debian or Ubuntu base image can be cached and reused easily by other images.

Docker is basically like the movers who pack items into a container.

![Packing a container](http://www.hiloliving.com/Move/packingupthematsoncontainer.JPG)

The Docker command-line client gives you all sorts of commands for interacting with both images and containers.
There is also a rapidly-growing ecysostem around Docker itself, both third-party tools and a growing suite of Docker-branded utilities like Docker Compose, which helps you manage running several containers that together comprise a single 'application' (say, a web app container and a database container).

However, I've tried to keep my use of the Docker infrastructure to a minimum, and use Docker only for container and image management.
CoreOS's ecosystem will provide the coordination layers atop this, as I'm about to detail.

## CoreOS

CoreOS is a very minimal Linux distribution designed to run on cluster servers.
One of the most surprising parts is it has no package manager like Ubuntu or Fedora; you're expected to 'install' and run your applications exclusively using containers.
At the moment, this means using Docker, but the CoreOS team is also developing their own container runtime, rkt.

![Containers on a ship](http://static.progressivemediagroup.com/uploads/imagelibrary/The%20worlds%20biggest%20cargo%20container%20ships.jpg)

So CoreOS, in the visual metaphor we're developing here, is the boat all the containers are shipped on.
It provides the kernel, and your containers provide the rest.

## CoreOS components

CoreOS comes with its own set of attendant daemons and tools.
CoreOS's philosophy is to be modular instead of monolithic, and in theory any of these components could be swapped out - however, they're good defaults, and they're installed on CoreOS by default, so let's consider them essential for now.

Here's where my ship-themed visual aids start to break down, but bear with me a little.

### Fleetd: loading and unloading

We need a way to get all these containers on our ships.
Fleet, which we run on the command-line as `fleetctl`, is a program that lets us distribute 'services' across our CoreOS machines.
`fleetd` is the daemon that runs on the machines themselves.
It uses `systemd`, a Linux utility for managing long-running process, to manage the various 

The Fleet workflow looks something like this:

 * Create your application code
 * Create a `systemd` _unit file_ describing how to run it (i.e. using a Docker command)
 * Send the unit file to one of your CoreOS machines
 * Run the unit file with `fleetctl`

Fleet takes care of choosing which particular actual machine to run each service on.
It has some high-level options that let you achieve common configuration goals - for example, you can specify that only one instance of the API server should run on each physical machine, and Fleet will obey that constraint when choosing a machine to run a new instance on.
Or, you could specify that they must all run in separate geographic locations.
Or, they must only run on machines that are also running a database server.
Or only on machines with so much RAM.

A critical step of this process to note is that, if you want to run multiple copies of some service (for example, two API servers), you have to create two unit files.
Fleet introduces a simple templating language to make this easier, so you don't have to copy-and-paste your unit files.

With all this in mind: I give Fleet the role of crane, the machinery that gets your containers onto your ships.

![Crane with container](http://www.rigsofrods.com/attachment.php?attachmentid=197508&d=1303744861)

I want to take this chance to apologise all the people whose images I'm deep-linking to.
With any luck, this blog won't be popular enough that they'll notice.

### etcd: communication

`etcd` is a 'distributed key-value store'.
The 'key-value store' part should make sense - it means you can store a value associated with a particular key.
The 'distributed' part is what makes it interesting.
When you have multiple machines in a cluster, they need a way to communicate between themselves.

For example, once a machine joins the cluster and starts up an API server, it should let everyone else know there's a new API server at so-and-so address.
This is accomplished by setting a key in `etcd`, which is then automatically replicated to all the other machines in the cluster.

To continue the visual metaphor, `etcd` is the radio, the communication system that allows ships' captains to talk to each other.

![On the bridge of a vessel](http://www.travelwithachallenge.com/Images/Travel_Article_Library/Freighter-Travel/Container-Ship-Bridge.jpg)

You may also hear people talk about `confd` in the same breath as `etcd`.

### confd: responding to communication

In the scenario I mentioned above, a new API server coming online, you might want to take some action after this happens - for example, reconfiguring your load balancer to send traffic to the new API server instance.
`confd` is a principled way of watching `etcd` keys for changes.
To use it,

 * Create a template of your application's configuration file
 * Set `confd` to watch for changes to particular `etcd` keys
 * `confd` will regenerate your config file and restart your application when the keys change

I don't have a picture for `confd` :(.

Something to note is that `confd` actually works with many different bakends, not just `etcd`.
For example, you could set it up to watch Redis or ZooKeeper.

## Confused yet?

There's a lot to take in, especially when you're just reading about it.
Next time, I'll go over a high-level architecture of how I'm going to deploy my own little one-machine cluster, and hopefully this will all become a bit more concrete.


![A fleet of container ships](https://ferrebeekeeper.files.wordpress.com/2012/10/3417930513_392816a872.jpg)
