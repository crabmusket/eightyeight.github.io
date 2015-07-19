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
 * Fleet, Etcd, and Confd: utilities that ship with CoreOS

Hopefully this will put you in a good position to read more detailed posts about all of these topics with some high-level understanding of the whole system and the various pieces involved.
Throughout this post, I'll link to the best other pieces of writing I've discovered on various subjects - the ones that really helped me, and the ones I had trouble with until I had some basic understanding of other parts of the system.

## Anatomy of a cluster

Here's a diagram of a simple two-machine cluster using CoreOS and Docker.
Don't worry if it doesn't make any sense right now - by the end of this post, it should!

![Two-machine cluster](https://cloud.githubusercontent.com/assets/904269/7782873/983250ba-016f-11e5-88c5-299dcd82b306.jpg)

The big rectangles are physically separate computers - of which two represent cloud machines running CoreOS, and one represents my local computer.
The little blue rectangles are services running on each machine.
Finally, the yellow rectangles are containers, each of them running some sort of process, whether it be an application, database, etc.

The lines represent some form of communication - networked or otherwise.
(I haven't been very precise with these lines - there should be a lot more of them, but I've just tried to give you the general gist of the communication that goes on!)
We'll refer back to this diagram at the end of this post, and hopefully it will make a lot more sense!

## Containers, from afar

This is going to be a brief section, because frankly, I still don't understand containers very intimately.
Basically, you can think of a container as a lightweight virtual machine.
It's kind of like a chroot.
But these things can be _really_ light.
Like, so light they include [nothing but your application executable][docker-golang-scratch].

The idea of a container is that they contain everything needed to run your application, with their only dependency being a kernel.
That makes them super-portable.
You can take a container built on any machine, and run it on any other machine.
(With caveats, of course - for example, you can't run containers natively on Windows currently, because they're a Linux technology.
They're not _that_ magical.)

I hope that made sense, because it just gets more complex from here.

## Docker, as viewed from the stratosphere

Docker is a tool for creating, managing and running containers.
To be perfectly honest, I don't know how difficult this stuff is without Docker, because it's all I have experience with.
Essentially, `docker` is the command that you'll use when you want to interact with your containers.

In Docker, you get used to managing both images and containers.

 * **Images** are exactly what they sound like - images of a filesystem that you want to run in a container.
   For example, you could have a Debian image that contains a full copy of a Debian server.
   Or you could have an Nginx image that contains just the files necessary to run the Nginx server.
 * **Containers** are the actual running instances of images. 

The Docker command-line client gives you all sorts of commands for interacting with both images and containers.
It also, critically, lets you compile images, using a Dockerfile and your application source code.
A Dockerfile tells Docker what things to stuff into the image, and therefore what resources your container will have available to it at runtime.

Docker is clever, though, at reusing images - for example, if I have three different applications, which all need to run in a Debian environment, those three containers will all refer to the Debian base image, and branch off them like a tree.
This means that I only have to store one copy of the Debian image, and then can store three small layers, one with the modifications on top of Debian for each of my applications.

If you're familiar with git, it starts to feel exactly like the graph of commits in a repository - and the analogy is helped by the way Docker gives each image a uique hash ID.
Docker also has infrastructure built around this, like the Docker registry, which you can use to upload and download images.

There are additional tools, like Docker Compose, which helps you manage running several containers that need to work together.
However, I've tried to keep away from this, and use Docker only for container and image management.
CoreOS will provide the coordination layers atop this, as we're about to find out.

## CoreOS, on the surface

Despite what I've titled this section, there's very little to tell you about CoreOS 'on the surface', so we're inevitably going to have to dig down a little.
At its simplest level, CoreOS is an operating system like Debian or Ubuntu.
It's a very minimal OS, with almost nothing built-in.
It's designed to do nothing but run containers and some supporting services.

## The above, in pictures

I feel like now is the time, before we get into the heavy stuff, to have a brief interlude to recap some of the stuff we've just covered.
I will use pictures to do this.
Imagine these shipping containers are your applications.

![Shipping containers](http://www.lighthome.com.au/Images/How%20do%20I/Shipping%20Containers/shipping-containers.jpg)

Each container is some service - an API server, a load balancer, a database, a log aggregator, and so on. These containers are created using Docker, which is like, I don't know, whoever packs the stuff into shipping containers.

![Stevedores](http://tlisc.org.au/wp-content/uploads/2014/06/Stevedore-Team-Leader1.jpg)

But containers can't do anything in isolation. CoreOS is where we run containers: like this vessel, each CoreOS server runs a bunch of containers.

![Containers on a ship](http://static.progressivemediagroup.com/uploads/imagelibrary/The%20worlds%20biggest%20cargo%20container%20ships.jpg)

Note that you probably shouldn't run *this* many containers on a server, depending on what they are. But remember that for a serious application, you'll probably have lots of servers - each its own vessel, running a bunch of containers on it.

![A fleet of container ships](https://ferrebeekeeper.files.wordpress.com/2012/10/3417930513_392816a872.jpg)

That's your cluster.

## CoreOS's supporting cast

CoreOS, the vessel on which our containers are shipped, comes with its own set of attendant daemons and tools (like the crew of the ship!).
CoreOS's philosophy is to be modular instead of monolithic, and in theory any of these components could be swapped out - however, they're good defaults, and they ship with CoreOS by default, so let's consider them essential for now.

Here's where my ship-themed visual aids start to break down, but bear with me a little.

### Fleet: loading and unloading

We need a way to get all these containers on our ships.
Fleet, which we run on the command-line as `fleetctl`, is a program that lets us distribute 'services' across our CoreOS machines.
It uses `systemd`, a Linux utility for managing long-running process.

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
