# An AAA engine for GTA San Andreas in the browser — and rewriting Three.js

![Main Cover](./assets/3/0-main-cover.jpg)

This is the next chapter in the story of how I'm building an AAA-grade engine to run old games — specifically GTA San Andreas and its mods — in the browser.

## Intro

If you can't be bothered to read — here are the important links right away:

- [Demo](https://opensa.cc)
- [Video](https://www.youtube.com/watch?v=eA1gVWzANRU)
- [Repo](https://github.com/AlexSergey/opensa)


- [Previous article: I ran GTA San Andreas on my own engine in the browser](./1-i-ran-gta-san-andreas-on-my-own-engine-in-the-browser-solo-with-claude-in-3-weeks.md)
- [Previous article: How I built a perfect GTA San Andreas map with my own engine](./2-how-i-built-a-perfect-gta-san-andreas-map-with-my-own-engine.md)

> **What this is — and what it isn't.** OpenSA is an **experiment** and a learning project. The goal is the
> _engine_: a browser-based runtime built to be **compatible with the RenderWare formats** that GTA San
> Andreas uses — **not** to clone the game or redistribute it. It ships **no game assets**; you bring your
> own legitimate copy (or a community mod). Think of it as an alternative way to _run_ a game you already
> own, in the browser. GTA San Andreas, its assets and trademarks belong to Rockstar Games / Take-Two — this
> is an unofficial, non-commercial fan project.

---

In the previous episodes I covered building the first prototype, and then fixing the problems with the map.

Now for the fun part — modern graphics.

RenderWare itself gives you effectively nothing in the way of dynamic shadows and lighting: all of that is baked into the world objects as static prelight and as painted-on shadows, which are static too.

My plan was to pile on effects and find out what a browser is actually capable of, and how far I could push it.

Before starting, I wrote a couple of test scenarios where the camera flies over the heaviest parts of the map, so I could measure FPS. I decided to take a measurement at every step of the work.

The first run, before any of the changes, wasn't exactly encouraging: my map improvements — the high-poly vegetation above all — cost a serious amount of FPS. And it dropped unevenly: 43 FPS out in the countryside, 30 in San Fierro and Las Venturas, and only 18–19 in Los Santos, the densest city of the three. In other words, the heavy city was already on the floor before I'd even touched the graphics. Well then — let's see where this goes.

I started by picking a color management system, choosing between three options: AgX, ACES and Neutral.

![AgX](./assets/3/1-agx.jpg)
![ACES](./assets/3/2-aces.jpg)
![Neutral](./assets/3/3-neutral.jpg)

I settled on ACES as the middle ground between Neutral's HDR look and AgX's rather washed-out image.

Then a physically-based sky and proper fog.

![City](./assets/3/4-city.jpg)

The picture kept getting better and better…

![Cars](./assets/3/5-cars.jpg)

![City 2](./assets/3/6-city.jpg)

![Night](./assets/3/7-night.jpg)

![Night 2](./assets/3/8-night2.jpg)

![Shadows](./assets/3/9-shadows.jpg)

![Sun](./assets/3/10-sun.jpg)

And at last I got real headlights.

![Car Lights](./assets/3/11-car-lights.jpg)

![Car Lights 2](./assets/3/12-car-lights2.jpg)

But the FPS kept falling and falling. On one of the passes, after tuning cascaded shadows and lighting, I took a measurement.

![FPS](./assets/3/13-fps.jpg)

FPS had slid to 12–18. That was no good at all — because what we have here is essentially an empty city, and it still has to be filled with cars. And on that front I have big plans to simulate city life, which needs a lot of headroom.

There were also a number of problems still to solve. The shadows up close, for instance, looked truly awful.

![Shadows Close Up](./assets/3/14-shadows.jpg)
![Shadows Close Up 2](./assets/3/15-shadows2.jpg)

## Approaches to 3D rendering in the browser

Put simply, there are two ways to render.

WebGL is a fairly old technology. You get a lot of draw calls, and since the map is made of a huge number of objects, preparing them happens on the CPU and clogs the main thread — hence the stutters and the lost frames.

WebGPU is the new technology, the one that lets you use everything the device's GPU can do: you can take the load off the CPU and free up the main thread.

My current version was built entirely on WebGL, so a move to WebGPU it was. Except there's a problem — Three.js doesn't fully support WebGPU. I didn't know that at the time, though, so more on it later.

## WebGPU

Before rewriting anything, I decided to build a separate scene that emulates streaming: 15k objects, with 2k of them swapped out on the fly over a few seconds.

The Three.js result was disheartening.

![Test Scene](./assets/3/16-test-scene.jpg)

Looking at the FPS in a scene like that is pointless: 15k boxes stand one behind another, the GPU redraws the same spot over and over, and the number means nothing. What you have to look at is something else — how much time the CPU spends assembling a single frame and handing it to the GPU. The lower, the better.

And here WebGPU in Three.js gave 27 milliseconds against 10 on the old WebGL. Which is to say the new technology came out two and a half times SLOWER than the old one. A move like that buys you nothing!

I tried Babylon. On plain WebGL it was even worse — three times slower than Three.js. But in WebGPU mode it knows one clever trick: record the whole scene once and then just replay that recording. With it, frame assembly sped up by almost a hundred times. I did not see that coming.

Streaming ruined it, though. Babylon's recording is one per world: add or remove a single object and the whole thing has to be re-recorded, and that costs 50 milliseconds. My map, meanwhile, loads and unloads objects constantly as you drive. So I'd be getting a visible hitch on every one of those loads. Babylon is an excellent solution for static scenes, but it sags under dynamic ones.

Fable and I decided to write a patch for Three.js. And the patch worked: frame assembly got five times faster, the map started loading quickly, the camera flew, the car drove.

![Block](./assets/3/17-block.jpg)

![Low FPS](./assets/3/18-low-fps.jpg)

And the FPS didn't go up. At all. A frame still took a third of a second; the game ran at 3–4 FPS.

Then came rounds of digging, and here's what turned up: the time wasn't in the CPU. Everything we could reach from outside the framework, we cleaned up — frame assembly ended up about fifteen times faster, memory was squeezed down. And the frame still went off to the GPU and vanished there for hundreds of milliseconds. What's more, cutting the resolution by a factor of four didn't change that time at all — meaning the problem wasn't that the GPU had too much work. It was sitting inside Three.js itself, in its layer down to the graphics driver. And there's no reaching in there from outside — you have to own that layer.

After a few more rounds of coding, Fable handed me this.

![Stuck](./assets/3/19-stuck.jpg)

The whole project was at risk. So what now?

## Building my own Three.js

There was only one option left — write my own Three.js, one that's WebGPU-first and built from the start for heavy streaming.

I'm not a fan of turns like this and I try to avoid rewriting a core, because things like that very rarely pay off for real. So I decided to run a proof first.

I created a separate application — I called it "the Lab" — where Fable and I reproduced, step by step, what the production version could do.

![First Render](./assets/3/20-first.jpg)

We started by rendering a pile of simple cubes, and cubes with an alpha channel. Result: 120 FPS. Not bad — keep going.

Rendering one of the cities. 120 FPS:

![Map](./assets/3/21-map.jpg)

And at every step I took benchmark measurements, which I'll come back to below.

The map with basic lighting. 120 FPS.

![Map Lights](./assets/3/22-map-lights.jpg)

The map with night lighting — the same 120:

![Map Night](./assets/3/23-map-night.jpg)

## The great rework

In the previous article, on improving the map, I described an approach where I implemented a set of tools for processing models and textures, fixing various bugs along the way.

Its architecture was done unix-style: each tool takes a game in and gives a patched game back.

![Pack Architecture](./assets/3/24-pack-architecture.jpg)

And I thought: what if — since we're replacing the engine wholesale anyway — I made my own format for models as well as textures?

What that could buy us:

- We can bake shadows into the LOD objects and only compute shadows for the HD sector (same goes for lighting).
- We can pack textures into arrays so the GPU takes them in batches, and cut the number of draw calls that way. I also tried the classic option — dumping everything into one big atlas — and it came out 7% worse: in GTA, textures tile across a surface by repeating, and in an atlas there's nothing to repeat.
- We can build entirely new gameplay mechanics that the original game never allowed for.

So, off we went to build another tool.

This is what the baked shadows look like.

![Baked Shadows](./assets/3/25-shadows.jpg)

There's no getting away from bugs, of course.

![Bug](./assets/3/26-bug.jpg)

![Bug 2](./assets/3/27-bug2.jpg)

![Bug 3](./assets/3/28-bug3.jpg)

I won't keep you in suspense: many rounds later I had a tool that not only improves the graphics at zero runtime cost, but also makes the models lighter, the textures more efficient, and so on.

A new problem showed up — since this is a new format, we have to introduce it to the animations, the cars and everything else too. Some time later we had:

![Car](./assets/3/29-car.jpg)

Interim results:

![Results](./assets/3/30-results.jpg)

After numbers that optimistic, a decision was made — **REWRITE THREE.JS**!!!

## In the game

The first run in the actual game was a mess, but — the world on my own engine, a historic screenshot!

![In Game](./assets/3/31-in-game.jpg)

Plenty of bugs, naturally.

![Bugs](./assets/3/32-bugs.jpg)
![Bugs 2](./assets/3/33-bugs2.jpg)
![Bugs 3](./assets/3/34-bugs3.jpg)
![Bugs 4](./assets/3/35-bugs4.jpg)

The new engine made it possible to do beautiful clouds.

![Clouds](./assets/3/36-clouds.jpg)

And water.

![Water](./assets/3/37-water.jpg)

Realistic lighting would have been impossible without transforming the map. Here's what it looks like on a map without the processing I described in the previous article:

![Lights Bug](./assets/3/38-lights-bug.jpg)
![Normals Bug](./assets/3/39-normals-bug.jpg)

And here's what it looks like with it:

![Lights Fix](./assets/3/40-lights-fix.jpg)

## How I work

A body of work this size is impossible without constantly keeping an eye on both quality and benchmarks. I measured non-stop and reacted to every FPS drop. In most cases they turned out to be bugs.

The graphics still aren't fully where I want them, but I'll keep working on that.

Another observation: Fable gets things wrong! Badly, sometimes. There was one occasion — two problems back to back: the engine was designed for a single car, and a car was designed for a single paint set. Meaning the world would only ever have one make of car driving around, in one color. That wasn't a bug, it was an architectural decision.

Working with Fable generally reminds me of working with a genuinely strong developer — Fable will argue and defend its work, but the final call is still mine.

The "Lab" approach is the real experience of large companies doing large-scale refactors: you prove the hypothesis on small fragments, then do the replacement in small steps, measuring each one. At every stage you can go back to the old version, because nothing is deleted until the migration is complete — so you can compare, and see what's missing and how it used to be done. You hide the old features behind flags one by one and replace them with the new ones. And at the end, you simply deprecate the outdated code.

## Results

In the space of two weeks I took performance from 16–18 FPS to 100–120. A frame is assembled 3–7 times faster, and the engine issues 5–12 times fewer commands to the GPU. All of it through a lot of grind, and in a state that isn't finished or fully dialled in yet — but it's a huge step forward, and it gives me a healthy performance buffer for implementing the thing that matters most: the city life system.

My next step is to build a properly living city, with characters and cars, real draw distance and the rest of it. Something that looks like a metropolis.

After that I'll come back to the graphics, because a lot of things are still in flight and I need to take my measurements on a genuinely loaded city to understand where the limits are.

Thanks, everyone, for reading. To be continued.

Links once more:

- [Demo](https://opensa.cc)
- [Repo](https://github.com/AlexSergey/opensa)
