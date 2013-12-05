---
layout: page
title: "Champs: Battlegrounds"
comments: true
sharing: true
footer: true
---

[Official Site](http://www.champsbattlegrounds.com)

{% gallery %}
/images/projects/champs/army-management.jpg: Army management
/images/projects/champs/damage.jpg: Damage # animation, bright effects
/images/projects/champs/targeting.jpg: Targetable area
{% endgallery %}

Champs was a project that I was fortunate to receive the opportunity of driving and architecting the technical development of the game logic and client from start to finish. It was exciting turning point for Quark Games, original game design, and the company's first 3D, real-time game. 

In short, it's a real-time multiplayer tactics game where players collect and train armies of "Champs" each with a wide range of unique abilities to defeat their opponents. Each Champ has a fixed pool of energy which constantly regenerates, and determines how often moves are executed. Terrain and positioning are critical, as certain tiles give energy regeneration bonuses.

#### Stats

* Company: Quark Games
* Time: 11 months (09/12 - 08/13)
* Technologies: C#, Unity3D, Websockets
* Platform: iOS, Android
* Role: Client Lead Engineer
* Team size: 2-5 (including myself!)

#### Game logic engine

This started as part of the client and ran on the device, but as the scope of the project expanded we decided to factor out the game logic into a separate component that could be run on both server and client (for offline play).

Player input, and set-up commands are received by the library and processed at a 30ms tick. On the server this meant having a thread per game, with multiplexing connections. Output commands would contain the minimum necessary information: ids for visual outcomes, commands reporting critical game state (position, health, energy). This would give game designers the greatest flexibility to tweaking. Since the client also included this library it could leverage shared code for predictive rule evaluation (i.e. evaluating whether a command was valid).

Configuration for units was placed in a human-readable JSON format. Then loaded to instantiate classes with those config values. I wrote a lot of scripts to grab effect ids and animation timings out of Unity3D prefabs and to serialize them to be able to be read by our game logic engine. This allowed for us to leverage the tools the tech artist was familiar with and not disrupt their workflow

#### UI

A lot of Champs is UI, or I should say a lot of any game is UI. Especially if your client team consists of 3.5 people and you need to implement everything from in-game UI to store, manage army, settings menus, etc. 

#### HP/Energy Bars

Surprisingly problematic, since the naive NGUI implementation gets you into trouble. We had 12 of them on the screen, and the time spent in CPU layout was eating up 20-30% of our frame time. NGUI only uses vertex positions and colors, it's a dynamic mesh. If you implement hp/energy bars' outlines/dividers, alpha pulsing sprites with each line being a quad: you end up with a dense amount of geometry that's constantly being laid-out on CPU and being submitted to the GPU.

The solution is obvious: use shaders! Use an NGUI quad, but override it's material to use your own shader. This turned 26 quads per Champ into 2 (which were static each frame). Pre-generate the dynamic outline/dividers once as a texture, use 1D textures with UV coordinates to draw the bar, blend in the fragment shader. Now our iPad FPS goes from 23 to the vsync'ed 30. 

#### Targetable area tinting

We wanted to highlight which tiles were valid by tinting the rest of the screen. The naive implementation was to tint every material and unit that wasn't in the highlighted area. Besides bloating all of the other shaders and the extra time determining whehter any object in the scene should be tinted, this was so expensive that it brought our frame rate to a crawl.

The solution was to use screen space shaders. GPUs should have a stencil buffer...let's render the tiles we want highlighted into it. Unfortunately, Unity 4.1 actually didn't expose a stencil buffer, and rendering into a render texture was costly. So I exploited the fact that the majority of elements static elements in our scene didn't use alpha blending: encode the tint value (0 or 1) into the alpha channel, and reinterpret it while tweening the tint fade.

#### AI

I can't beat the AI I programmed for Champs, which means I'm either really bad at the game I developed (partially true I admit) or it's a precursor of Robot overlords. One of our talented game designers came up with the design of the AI that would use a heuristic scoring system to weigh potential actions. For moves, this meant evaluating the value of a tile's map position and weighing them against the damage score of a particular ability.

The system is able to compute the "offensive value" and "vulnerability" of any space given current enemy positions (in a tactics game units aren't constantly moving, this heuristic was roughly correct). This took into account for all range rules, terrain bonuses, pairwise damage between units. This was weighted against the damage values relative to the target's health: giving more weight to weaker units, but not enough to make them constantly give chase.

A big part in optimizing the system was providing tools for the designers. I wrote an AI heatmap that would display each of the values computed for tiles, giving the designer a visual indication as to why the AI was making certain decisions, and allowing him to tweak weights on certain values per unit: leading to more aggressive or defensive behaviors.

Eventually we added heuristics for computing damage from triggering environment effects (i.e. pushing an enemy into mines), using remote detonate, planting mines/obstacles. The AI became so good a board went up with the score of people in the office vs AI. Most of them were losses :)

#### Android Workarounds

I have to put this in a category on our own, since the hacks to fix some issues in the very fragmented device space were crazy, but we were able to get the game to ship without graphical glitches!

**Alpha Channel and Stencil Buffer Woes**

Well when we got to preparing Android for ship the alpha hack fell apart. It turns out if you use an alpha enabled (32-bit) display buffer for some devices, pressing the volume controls or when the OS displays an overlay, the game would flicker like crazy. Obviously there was a problem with Unity's render context and the OS's screen compositing.

We were lucky that Unity 4.2 had shipped and exposed stencil buffer support, nice! Now I can do this cleanly and get rid of any alpha hacks. Turns out as well that still, some devices don't expose a stencil buffer (I believe the pattern was that stencil was only supported for devices that exposed the OES_packed_depth_stencil extension). Necessarily, my only option for those devices was to take the performance hit and use a render texture.

**Texture corruption when returning from a notification**

We hit a Unity bug where returning to the UnityPlayerActivity would cause our app to lose it's OpenGL context resulting in full texture corruption. I suspected it was because for some reason it was trying to launch a new instance, or that we weren't passing the right flags back. By luck and guessing I changed the launch Intent's class to the UnityPlayerNativeActivity and I set some flags which logs show was overriden, but that fixed the issue!

#### Build Automation and Tools

One of the things about working on a small team is the need to stay efficient. That means automate as much tasks as necessary, so you can focus on engineering and not on getting builds done. At Quark we focused a lot of efforts on creating a 1 command build script from Unity to Xcode archive + ipa, and then putting them onto a continuous integration "server" (i.e. extra macbook pro sitting on an empty desk ;) ) where QA could pick up builds.

Eventually we got the script to automatically increment version #, push the commit to the release branch, and print out the release notes with links to the github pull requests.



