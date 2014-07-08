---
layout: post
title:  "Monster Mash: entry #2"
date:   2014-07-08 13:01
tags:   torque jams
---

Welcome one, welcome all to my third and final [Monster Mash][] jam log.
(Here's [the first one][] and the [second][].)
Since I got busy during the week, this will basically summarise everything I did between Wednesday and the end of the jam last night.
I'll also write a bit of a post-mortem about the project and how I felt it went.

I have to report, unfortunately, that I didn't actually make it into the jam, because a midnight I was fiddling with getting the game's [itch.io page][] up and working.
However, I _did_ make a game, and I had a lot of fun doing it.
Which is kind of the point of doing a jam, so I consider this mission a success!

So, what did I get up to in the latter half of the jam?

 * Added AI!
   Obviously between Wednesday and Monday I wrote basically the whole AI system for tourists (the ones you eat) and rangers (the ones who are dangerous), but initially I just started out by making tourists respond to your movement.
   The idea was that if you swim around fast and close enough, they'll notice you.
   You can also make noise deliberately - and of course, attacks causea lot of noise.
   I brought in the event-queue code I'd written for [vlrtt][] to simplify this process, so the backbone of AI code looks like this:

   ```
   eventQueue(Monster);
   event(Monster, Swim);

   function Tourist::onAdd(%this, %obj) {
      subscribe(%obj, Monster, Swim);
   }

   function Tourist::onMonsterSwim(%this, %obj, %swimPos) {
      // %swimPos is the location the monster was at when it made a noise.
   }
   ```

   So now every 500ms, I make the monster emit a swim event by calling `postEvent(Monster, Swim, %obj.getPosition())` and the tourists hear it.
   They have to do some processing, of course, to make sure they don't hear things too far away.
   Here's a video from my early AI tests:

<iframe width="560" height="315" src="//www.youtube.com/embed/TpmzfEb0ENg?list=UUvuAVYC7MeN989AmKI0sYHQ" frameborder="0" allowfullscreen></iframe>

 * Controller input!
   I spent far too long on this.
   I just couldn't work out why there was no controller input in `t3d-bones` despite the controller working flawlessly in the Empty template.
   I eventually discovered that you have to assign `$enableDirectInput = true` somewhere in your code _as well_ as calling `activateDirectInput()`.
   Who'd have thought.

 * A help dialog.
   It's barebones, with just enough room to display the controls and a brief instructional message.
   It can also be hidden with a keybind.
   Yay.

 * More AI!
   I conscripted the state machine I wrote ages ago, again for `vlrtt`, to give the tourists and rangers some simple scripted behaviour.
   They revolve around a simple three-state loop: `relaxed`, `scared` and `fleeing`.
   They build up through this process when they hear monstery noises, and they go downwards with time.
   Of course, rangers have states `alert` and `pursuing` instead of `scared` and `fleeing`, but the state flow is similar.
   For example:

   ```
   new ScriptObject(TouristSMTemplate) {
      transition[relaxed, monsterNoise] = scared;
      transition[scared, monsterNoise] = getHelp;
      transition[scared, timeOut] = relaxed;

      transition[_, attackNear] = getHelp;
      transition[_, attackFar] = scared;
   };
   
   ```

   There's also some fun behaviors built around these states.
   When tourists flee, they do so towards the nearest ranger, and then bring the ranger back to the location they heard the monster.
   They'll then all be worried for a little while, and then go back to whatever they were doing.
   And if someone becomes scared nearby, they'll turn to them and ask what's up.
   Kind of unhelpful, but a fun touch.

![Tourist thinks the water is safe](http://imgur.com/r7Ryv2m.png)

 * Eating.
   You can now actually eat people.
   This went in on Sunday, funnily enough.
   I think I should have maybe done it much earlier.
   On the other hand, it was never going to be a very complicated part of gameplay.
   Basically, if you attack and a person is close enough, they'll delete themselves.
   I also prevent you from attacking if you're under the boardwalk, which I thought was a nice touch.
   Originally that would be accompanied by a head-bumping sound, but I didn't get around to adding sound effects.

 * Being photgraphed!
   Yes, on Monday I finally hacked in the ranger's ability to make you lose the game.
   This was a change from my original intent to make the rangers fire tranquilisers at you.
   They're now not park rangers but photographers, and if you attack when they're nearby and alert, they'll snap a photograph and cause the game to end.
   I don't have much to say about this, actually.
   Other than, again, I probably should have done this a week earlier.
   As it is, I didn't get any time to playtest or balance the range at which photographers capture you, or the circumstances under which they should or shouldn't do so.

 * More AI...
   Since the way the tourists and rangers responded to you was the most important part of the gameplay, I spent a while trying to make the interactions rich.
   Unfortunately I didn't quite succeed, I don't think.
   This was mostly due to the limited navmesh support in Torque currently.
   For example, I had no way of checking whether movement target points were on the navmesh, resulting in lots of trial-and-error code like this:

   ```
   while(!%obj.setPathDestination(chooseRangerSpot(%obj.spot))) {}
   ```

   where `chooseRangerSpot` will choose a random position near the destination each time it's called.
   So you _eventually_ get a valid path.
   If you assume, as I do, that there's a valid location _somewhere_ near the target...

 * Particles! (Lukas would be proud.)
   I really spent far too long on Friday making all the various partlcle effects happen.
   There's a small bubbly one when you make a noise to scare people, and a big splashy one when you actually attack.
   And a really terrible trail when you move, so you have some visual indication of whether you can be heard.
   I was quite proud of how they turned out in the end, basically being the first particle effects I've written by hand.
   Actually, that's a complete lie.
   But they're the first effects I've written for a long time.

![Bubble and attack particles](http://imgur.com/A6SpeWP.png)

 * An end-game GUI and cycle.
   This was something that actually really excited me - figuring out the process of how to _end_ the game completely, deleting all the objects, and showing a GUI, then _recreating_ the entire thing to start over again.
   It actually turned out to be fairly easy, but I ended up having to use a `commandToServer` to restart the game, although it wasn't necessary to end it.
   It had something to do with calling `GameConnection::onEnterGame`:

   ```
   // After game has ended:
   function resetGame(%val) {
      if(%val) {
         commandToServer('reset');
      }
   }
   
   function serverCmdReset(%client) {
      onStart();
      %client.onEnterGame();
   }
   ```

   Anyway, now I'm worried that I understand the client/server interaction in the engine _even less_ thanks to `t3d-bones`'s minimal setup.
   Maybe that'll be a topic for a future [safari][]...

So that's that!
The jam is over and I have a finished product of sorts.
But there was a lot I planned and didn't get to complete.
Like:

 * Boats floating on the lake.
 * Counters showing the number of people you've eaten, the number remaining, and the number that escaped.
 * A way to actually attract people towards your location.
   At the moment, tourists who are scared just freeze.
   This doesn't really help you.
   I just didn't have any way to find the closest point on a navmesh to the monster's position, so I ended up not bothering.
 * Better AI in general.
   Like fleeing completely after being scared too much, and tourists wandering around between different resting spots.
 * Sound!
 * An intro sequence, with the camera lurking under the water's surface, then rising up into the starting position.
   A luxury, I know.

And much more besides.
Now that the jam's over, I've had time to bother creating a [GitHub repo][] with my code in it.
Feel free to poke around if you'd like to see a poorly-commented, extremely hacky example of a simple single-player game.
How do I think it went?
I guess I'm happy with how it turned out overall.
Torque certainly held up, and surprised me occasionally with its ease of use.
Like how easy it is to use a gamepad, once you've figured out which undocumented global variable you need to enable.

Writing this amount of code again confirmed to me how much I dislike TorqueScript as a development language.
I ended up making heavy use of schedules in some of the code, which meant duplicating calls to `schedule` and writing temporary functions outside of where they were needed.
It would have been much nicer if I could have used something like Javascript's `setInterval` with an anonymous function.
I was also bitten several times by the lack of name safety - if I made a typo in a local variable name, I'd get `""` instead of an error.
Not _too_ difficult to debug, but debugging I really didn't have time or patience for.

It had been a while since I tried to use the COLLADA art pipeline, and this time I found it much improved for some reason.
Possibly because I wasn't trying to make textures, simply using solid colours everywhere.
I even managed to assign different materials to different faces on the same object!
Tricky stuff.
Also, I believe Blender's COLLADA support has improved since I last used it, so that may have helped.

So overall I was quite happy with the engine, though I did discover a couple of bugs along the way.
Which is good and bad.
I suspect the engine is rarely used without the padding of the template scaffolds, so some of these issues aren't immediately apparent.
I also think that the engine is simply not used by many people, and those who do use it haven't tended to share the massive amount of niggling bugs they've had to fix.
Which makes sense, since T3D's development was closed up until recently.

However, I also found that the engine is just very difficult to work with in script only.
For example, I was writing my own spring physics simulator in order to make the camera work bask in my second post.
Spring physics, honestly, is exactly the kind of thing I think a good game engine should handle - I should be able to attach a couple of springs between the camera and player, and it should jut work.
I also found some inconsistency in the API and implementation of existing features.
For example, the camera's tracking mode does not alter the camera's transform, so you can't actually tell where it's looking at any given moment without redoing the maths yourself.

So, I think we have some work to do before I would call T3D completely user-friendly - just fising lots of these annoying issues and providing a sane and sensible API with good documentation.
I look forward to the challenge.

[Monster Mash]: http://itch.io/jams/monster-mash
[the first one]: ../../2014-06-29/monster-mash-entry-0/ 
[second]: ../../2014-07-01/monster-mash-entry-1/ 
[itch.io page]: http://eightyeight.itch.io/nessie-sim-2014
[vlrtt]: https://github.com/eightyeight/vlrtt
[safari]: ../../safaris/
[GitHub repo]: https://github.com/eightyeight/nessiesim14
