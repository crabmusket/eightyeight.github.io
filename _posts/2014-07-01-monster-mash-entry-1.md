---
layout: post
title:  "Monster Mash: entry #1"
date:   2014-07-01 13:46
tags:   torque jams
---

Welcome one, welcome all to my second [Monster Mash][] jam log.
(Here's [the first one][].)
I've got two days to summarise, since I didn't do an awful lot of work yesterday.
But, I:

 * Made a third-person camera controller with relative movement to the camera, all in script!
   This is an ugly hack and one I don't want to repeat.
   The main challenge was the maths involved in converting the local direction we want to travel (e.g., 'away from the camera') into absolute coordinates to feed to the player move input.
   And then realising that I needed to update that variable regularly, because the direction the player would be moving often changes constantly (when moving sideways relative to the camera).
   At the moment, here's the guts of it:

   ```
   function updateMovement() {
      %axes = getCameraAxes();
      %front = getField(%axes, 0);
      %right = getField(%axes, 1);

      %moveDir = VectorAdd(VectorScale(%right, $right - $left),
                           VectorScale(%front, $forward - $backward));
      $mvRightAction = getWord(%moveDir, 0);
      $mvForwardAction = getWord(%moveDir, 1);
   }
   ```

   It's fairly simple after all that effort!
   Note that I had to write my own `getCameraAxes` function because the camera transform doesn't take object tracking into account.
   The global variables `$forward` and friends represent the states of the WASD keys.
   Still to be done: controller input!

 * I also made a spring physics system to keep the camera following the player.
   Currently it just uses springs and no dampers, so it kind of sucks, but it works.
   It's also all in scripts, and it's also something I never want to do again.
   That said, I'm thinking of generalising the system so that the islands and shorelines can act as springs as well, keeping the camera aligned more usefully (since at the moment, there's no way to manually rotate the camera, which sucks a bit).
   Again, here is is in full, just so you can share how painful vector maths is in TorqueScript:

   ```
   function updateCamera() {
      // Diff vector in x, y only, since we don't care about camera height.
      %diff = getWords(VectorSub(TheMonster.getPosition(), TheCamera.getPosition()), 0, 1) SPC 0;
      %dist = VectorLen(%diff);
      %diff = VectorNormalize(%diff);

      // Attractive/repulsive force from monster.
      if(%dist < 18) {
         TheCamera.camForce = VectorScale(%diff, -1 * (18 - %dist));
      } else if(%dist > 22) {
         TheCamera.camForce = VectorScale(%diff, %dist - 22);
      }

      // Drag.
      TheCamera.camForce = VectorAdd(TheCamera.camForce, VectorScale(TheCamera.camVel, -1));

      // Physics!
      %position = VectorAdd(TheCamera.getPosition(), VectorScale(TheCamera.camVel, 1 / $MovementHz));
      TheCamera.setTransform(%position SPC getWords(TheCamera.getTransform(), 3, 6));
      TheCamera.camVel = VectorAdd(TheCamera.camVel, VectorScale(TheCamera.camForce, 1/50));
   }
   ```

And today, I:

 * Fiddled a lot with movement physics.
   I discovered a [big hooting bug][] that was screwing up character movement underwater.
   It wasn't major, just meant that the chap would always accelerate to full speed instantly.
   Which, you know, I could deal with, but I just wanted a bit of smoothness to the movement.
 * Added collision to the boardwalks, in order to set up a navmesh, which you can see below.
   Apparently, my efforts intended to make the trees and boardwalks un-collide-able didn't work.
   Which I should probably look into, as all those meshes should be hurting performance...

![Navmesh on island](http://imgur.com/KmodSQF.png)

 * Made the water not-see-through, thanks to Azaezel pointing out the underwater fog settings. Hoorah!
 * Made the shadows less rough, thanks to Steve for sharing his sun settings.
 * Another bright suggestion from the #GarageGames IRC channel was to add more (by which I think they meant _any_) goats.
   I'm very tempted... but it will probably have to wait until after the jam.
 * Also, thanks to that bunch, the game has a name: "Loch Ness Monster Simulator 2014".
   It has a ring to it.
 * Added colours!
   After struggling yesterday and settling for a single-colour palette, I played around with the stock colours Lukas [added][] from T2D, and found some that I liked.
   This was much easier than fiddling with RGB codes!
   Here's what it looks like now:

![Lakeside with tourists](http://imgur.com/LYCx6y7.png)

 * Yes, those are tourists you can see chilling by the lake.
   And an umbrella, which I thought was cute.
   They're basically just blob-shapes, but I added arms so you can tell which direction they're facing.
   I think the park rangers will basically just be this mesh, but with hats.
   They'll tranquilise you with... I dunno, mind powers or something.
   Anyway, they have no AI yet, but that's on the agenda for tomorrow...

Till next time!

[Monster Mash]: http://itch.io/jams/monster-mash
[the first one]: ../../2014-06-29/monster-mash-entry-0/ 
[big hooting bug]: http://www.garagegames.com/community/forums/viewthread/138093
[added]: https://github.com/GarageGames/Torque3D/pull/613
