---
layout: post
title:  "Monster Mash: entry #0"
date:   2014-06-29 23:24
tags:   torque jams
---

The first day of my [Monster Mash][] attempt has ended with some fortunate progress.
Though I'm technically cheating (the jam starts tomorrow), I absolve myself by pointing out that I'll be having a fairly busy week with deveral days off anyway.
Here's what I got done today, mostly in gaps between getting on and off aeroplanes:

 * Drafted the single level the game will take place in, first on paper then in Blender.
   It includes a lake, two islands, a boardwalk, and lots of pine trees.
   _So many_ pine trees.
 * Learned how to import stuff into Torque from Blender.
   I feel this isn't something to be all that proud of, because people manage it all the time.
   I've always had trouble with it, however.
   But now my meshes are going into the game as they should.
   And while I haven't tried to work out texturing yet, I'm feeling much more confident about the whole process.
 * Learned how to use `WaterPlane` settings.
   Seriously, there are a lot of them.
   I mostly futzed with the array of ripple textures to get a really good ooze-like look, like the deadly sludge from Portal.
   Still to be done in this area: make the water less transparent.
   At the moment, it's completely see-through, which I don't want.
 * Re-learned how to get the game to capture mouse input directly (spoilers: it's not about your `ActionMap`, you have to set `noCursor` on your `GameTSControl`).
 * Fiddled with separate player/camera input schemes.
   I discovered separating the player and camera is easy enough - you just call `stFirstPerson(false)` on the connection and it starts to view through the camera.
   It gets trickier when you want to to any sort of control of both of them.
   My first step now is to make the movement relative to the camera, rather than absolute.
   I'm sure there's a resource out there for that...

And here's a pic of the level so far!

![Current level progress](http://i.imgur.com/YuYYyNf.png)

Thoughts:

 * I'm enjoying being able to actually create stuff in Blender and put it in the game.
   Not something I've done for a long time :P.
 * I probably should be doing less manual Blender-ing of stuff like trees, and making use of the shape replicator, since that's what it's designed for.
   I didn't do that originally just because I was excited after having finished the boardwalks using instanced geometry, so I ust continued on doing the trees in the same fashion.
   Hopefully now the level is relatively finalised I won't have to worry too much about it.
   Also, I can't benefit from using LODs this way.
	As far as I can tell.
 * I messed up my export process by deciding part-way through that I should really only have part of the level geometry be collidable - i.e. not the trees or the details of the boardwalks.
   I solved this quickly by just exporting different parts of the level as two different COllada meshes.
   This wouldn't be a problem if I were designing the level in T3D using replicators and placing the boardwalk instances manually, since each of them could have their own collision meshes.

[Monster Mash]: http://itch.io/jams/monster-mash
