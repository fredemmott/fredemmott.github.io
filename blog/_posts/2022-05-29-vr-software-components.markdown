---
layout: post
title:  "An Overview of VR Software Components"
date: 2022-05-29 11:54:00 -0500
tags: VR
---

The VR software stack has many user-visible components, and increasing interest
in optional, third-party components, which can add more functionality or improve
performance. Here's how the most common ones fit together and relate.

<!--more-->

## Overview

![Major software components of the VR pipeline for several vendors](/assets/images/2022-05-vr/overview.svg)

Some parts of this may be missing: most people don't use an API translation layer, and some games
do not use an Off The Shelf (OTS) game engine. Additionally, most users only have the drivers for one headset
installed.

There's a few important caveats for this graph:

* Some vendor-specific information is not publicly available, such as how Pitool interacts with the
  other APIs, or how different vendors' OpenVR drivers interact with the vendor-specific stack. The links I've
  shown are my best guess - please [get in touch] if you have a public source of authoritative information.
* I've simplified a few things here to keep the chart manageable; for example, the SteamVR runtime and compositor
  are separate components, and only some OpenVR drivers use the SteamVR compositor.

In practice, most people will be using a small subset of this graph; for example, when I play
[DCS World], the stack looks like this:

![DCS -> Oculus API -> Oculus Runtime -> Oculus Headset](/assets/images/2022-05-vr/dcs-oculus.svg)

## Optimization

First, some non-VR-specific advice:

1. If it ain't broke, don't fix it: the way things work by default is likely the most well-tested path; it's easy
  to break things, and can be hard or time consuming to get back to a previous good state. Unless you enjoy
  tinkering, if you find things playable and enjoyable, it's probably best to leave things alone. That said...
2. Make a full system backup first, or be prepared to reinstall.
3. If you try something (a new component, or a new setting), test it, and:
   - if it doesn't work, undo it immediately
   - otherwise, keep a note of it so you can undo it later

### Optimizing for Performance

Find the most direct path top-to-bottom, backtracking as little as possible. If you need to use OpenVR, this
often means using something in the "API Translation" layer, avoiding SteamVR and the
OpenVR drivers - unless you have a SteamVR-based headset like the [Valve Index], in which case, directly using
OpenVR+SteamVR is usually the best way to go.

### Optimizing for reliability

If the game has direct support for your headset, use that - otherwise,
start with OpenVR+SteamVR, avoiding the API Translation layer. If you have problems, try the tools in the API
Translation layer.

These approaches *usually* get you the best results; some people are happy calling things 'done' here,
while others consider them a starting point for experimentation.

## The Layers

### Application/Game

A 'game engine' is a reusable component that aims to solve common problems that most games need to solve -
[Unreal] and [Unity] are the most common for VR; these problems include:

* turning a map and a player position into instructions for your graphics card
* game physics
* changing the game world based on different kinds of user input - e.g gamepads, keyboard and mouse, or VR
  headsets/controllers

Some engines are 'off the shelf' and available to anyone who wants it - or is willing to pay for it - but some
are written for a specific developer, or a specific game. If it's just for one game, the resuabiltiy is gone,
but it can still be helpful for developers to think of the engine as a separate component.

If an off-the-shelf engine like Unity or Unreal is being used, developers usually don't *need* to care about
the Oculus API or OpenVR - the engine handles the basics for them after they click a few buttons - but if they
make their own engine or don't have it as a separate component at all, they will need to build all VR support
from the ground up.

That said, even when using an off-the-shelf engine, games often go beyond the engine-provided features to provide
a better or more immersive user experience; for example:

* it can be jarring to see [HTC Vive] controllers in your hands in game when you are using a [Meta Quest] or
  Valve Index
* controls designed for a Vive, Quest, or Index often feel wrong/intuitive when using a different controller; this
  can be due to location of buttons (especially trigger/grip butons), or push-and-hold vs toggle grip button
  norms

It is generally not possible/practical for anyone other than the game developers to add/remove/change a game
engine, and when it is possible, it is usually a massive task. For a given game, this is an always-present or
always-absent component - not an optimization choice for users.

## APIs

An Application Programming Interface (API) is like a language that different components can use to talk to each
other; if they speak the same language, they can work together.

A Software Development Kit (SDK) is a broader term - sometimes it's just another way of saying the same thing,
or it can be a collection of multiple APIs and other components that are useful for developers.

In 2013 when the original Oculus Rift Developer Kit (DK1) was released, games would use the Oculus API to directly
talk to the Oculus Runtime; this would only work with Oculus headsets. In the mid-late 2010s, other vendors -
Microsoft, Pimax, Varjo - introduced their own similar APIs, and games would implement direct support.

In 2016, OpenVR and SteamVR were released; OpenVR was designed for SteamVR, and is essentially SteamVR's
equivalent of the Oculus API. While the OpenVR banner also includes a hardware abstraction layer that allows
OpenVR games to work with hardware from multiple manufactures, this is a separate layer - the 'OpenVR Drivers'
layer - with SteamVR still sitting in the middle.

That said, while SteamVR was always in the middle, game developers could - and often did - choose to use OpenVR,
as the OpenVR+SteamVR combination makes it relatively straightforward to provide basic support for most
manufacturer's headsets, without having to write separate code for each manufacturer.

In 2017, OpenXR was announced, with 1.0 released in 2019; OpenXR is a collaboration between many companies:
Valve, Meta (formerly Facebook/Oculus), Microsoft, Varjo, Epic Games (Unreal), Unity among others. OpenXR
aims to provide a single API that fits multiple manufacturers and game developers needs, without being
tied to a particular vendor. OpenXR is primarily an API - or 'language' - and includes very little actual
software. The group provide an 'OpenXR loader', which tries to find and load OpenXR-compatible software component,
but these components are generally created by other vendors, not by the OpenXR project. OpenXR seems to offer
the usability of the Oculus API, combined with the flexibility of OpenVR.

## API Translators

A person who only understands English has trouble having conversations with people who only speak French, and
needs a translator; similarly, a game that only supports the Oculus API is unable to 'talk to' SteamVR headsets
without a translator.

[Revive] was one of the first of these, allowing games designed for the Oculus headsets to work with most
SteamVR-compatible headsets by translating the Oculus API to OpenVR:

![Game -> Oculus API -> Revive -> OpenVR -> SteamVR -> ... -> HTC Vive](/assets/images/2022-05-vr/revive.svg)

[OpenComposite] came shortly after, and essentially does the opposite:

![Game -> OpenVR -> OpenComposite -> Oculus API -> Oculus Runtime -> Oculus Headsets](/assets/images/2022-05-vr/opencomposite.svg)

While this wasn't necessary for OpenVR games to work on Oculus headsets, many users found that it provided a
more reliable or more performant experience than using SteamVR. This continued with OpenComposite-ACC, which
is now part of the main OpenComposite project: while OpenComposite historically translated between OpenVR and the Oculus API, OpenComposite-ACC added support for translating between OpenVR and OpenXR: as headset vendors
increasingly support OpenXR themselves, this allows removing SteamVR from the path for more headsets, such
as the [HP Reverb G2]:

![Game -> OpenVR -> OpenComposite -> OpenXR -> Windows Holographic API -> WMR -> WMR Headsets](/assets/images/2022-05-vr/opencomposite-openxr.svg)

While this looks like the same number of layers,
translating between APIs can be relatively easy for your CPU compared to SteamVR; that said, while most users
of OpenComposite+OpenXR report higher or more consistent framerates, this is not universal.

### Should Quest/Rift owners use OpenComposite+OpenXR?

If a game supports the Oculus API directly - unless you want to use other OpenXR software with that game (e.g.
[OpenXR Toolkit]) - probably not. Similarly, if you have a Varjo headset and the game directly supports Varjo, or
a WMR headset and the game directly supports WMR/Windows holographic, probably not. In these cases, OpenComposite
adds steps, instead of removing them:

![Replaces 'Oculus API' with OpenVR -> OpenComposite -> OpenXR](/assets/images/2022-05-vr/opencomposite-oculus.svg)

*Based on [a chart](/assets/images/2022-05-vr/mbucchia-chart.png) by [Matthieu Bucchianeri]*

## Runtimes and Compositors

Runtimes provide 'platform' features such as safety bounds (e.g. 'guardian' and 'chaperone'), system menus, preferences, and 'home environments'; if multiple VR games or apps are running, they also decide which is active at a given time, is affected by controller buttons, etc.

The combination of running apps and the runtime results in a list of layers. This is often just one 'world view'
layer, but can be more complicated - e.g.:

1. "World view"
2. in-game floating menu
3. overlay provided by an external application or runtime-provided 'embed window' feature
4. runtime-provided 'system' menu

Some layers like the 'world view' provide an image for each eye, while others provide a shape (e.g. 'rectangle'
or 'curved rectangle'), an image, and position information. For example, menus are often curved rectangles
displayed at a specific position. A compositor's primary purpose is to merge these layers into a single image
for each eye; ideally, this will then be sent to the headset as directly as possible.

The runtime and compositor are usually bundled together, and can usually be thought of as a single thing; however,
for SteamVR, it can be important: "OpenVR Drivers" can be thought of as a second API translation layer, though
for SteamVR itself, rather than for games. OpenVR drivers (i.e. hardware vendors) can choose whether or not the
SteamVR compositor is used - there are a few possible paths:

![OpenVR drivers can use the SteamVR compositor, a vendor compositor, or both](/assets/images/2022-05-vr/openvr-driver-types.svg)

* 'Vendor A' is the best path here: one runtime, one compositor
* 'Vendor B' is second best: two runtimes, but one compositor
* 'Vendor C' is the worst case: both SteamVR and the vendor's runtime and compositor are used at the same time

Which path is taken is up to the author of the OpenVR driver for that vendor's headsets - it's not something
that end users are able to control, or something that is usually documented. Tools like OpenComposite-with-OpenXR
are likely to help more if your vendor's taken path C.

Some OpenVR drivers are automatically installed when you install SteamVR and your headset drivers, but some
need to be installed separately. For example, there is no need to separately install a SteamVR driver for a
Valve Index, or an Oculus Quest - installing SteamVR and the normal oculus software is enough - but
Windows Mixed Reality for SteamVR needs separate installation. This isn't an 'extra layer' for WMR compared to
other SteamVR-compatible headsets - it's just a matter of how they chose to package it.

## Common Questions

### OpenVR vs OpenXR vs OpenComposite vs Compositors

* OpenVR is primarily an API for SteamVR
* OpenXR is a similar API supported by multiple vendors
* OpenComposite is a translation layer between OpenVR and the Oculus API, or OpenVR and OpenXR
* OpenComposite-ACC was the previous name for OpenComposite's OpenXR support
* OpenXR does not include a compositor, and you can not install 'the OpenXR compositor'
* Many vendors - including Valve and Oculus - provide an OpenXR-compatible compositor
* OpenComposite is not a compositor

### OpenXR, OpenComposite, and OpenXR Toolkit

OpenXR has the concept of 'API layers'; these aren't the visual layers that the compositor deals with - instead,
they allow additional software to insert itself in between the game and the runtime, intercepting, watching,
and/or modifying the OpenXR functions.

This allows things like:
* changing how controllers work in the game, e.g. simulating different kinds of controllers
* adding, removing, or modifying visual layers, e.g. adding overlays
* giving the game differnt information about the hardware, e.g. changing resolution or eye position

OpenXR Toolkit is one of these API layers; while it requires OpenXR, it does not require OpenComposite, and
neither OpenXR or OpenComposite require it.

The OpenXR loader is generally part of the game, and is responsible for finding and loading any wanted API
layers; once the loader is finished, the flow for an OpenXR game might look like this:

![Game -> OpenXR -> OpenXR Tookit -> Windows Holographic API](/assets/images/2022-05-vr/openxr-toolkit.svg)

OpenXR Toolkit is optional in this flow - it can be removed, or, additional API layers can be added, such as
[OpenKneeboard]:

![Game -> OpenXR -> OpenKneeboard -> OpenXR Toolkit -> ...](/assets/images/2022-05-vr/openxr-openkneeboard.svg)

These API layers are usually not needed for the game to work, but you might install them because you want
some extra features or options.

### DCS World and OpenXR

DCS World does not support OpenXR - however, it does support:

* OpenVR
* The Oculus API
* The Varjo SDK

WMR users in particular often hear that OpenXR can improve DCS performance; in this case, the goal is to replace
SteamVR and the OpenVR driver with OpenComposite and OpenXR, however DCS will keep using OpenVR - OpenComposite
'translates':

![OpenVR -> OpenComposite -> OpenXR -> Windows Holographic API](/assets/images/2022-05-vr/opencomposite-openxr.svg)

It would be possible for DCS to include OpenXR support in the future, but if this was based on OpenComposite, the
graph above would be unchanged - direct support would be much more likely to be beneficial:

![Game -> OpenXR -> Windows Holographic API -> WMR -> WMR Headsets](/assets/images/2022-05-vr/openxr-wmr.svg)

### DCS World, Steam, and SteamVR

DCS World can be purchased both through Steam, and direct from the publisher; the two versions are practically
identical, and it does not affect your options for VR:

* You can use SteamVR even if you purchased/downloaded DCS World outside of Steam
* You can use the direct Oculus API support even if you purchased/downloaded DCS World through Steam

## Disclaimer

I used to work for Meta, but in an unrelated area (programming languages). This post and it's opinions are
purely my own, and soley based on public information. I use an Oculus headset, which I purchased retail, and
was not reimbursed for it.

This post is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose and noninfringement. in no event shall the authors or copyright holders be liable for any claim, damages or other liability, whether in an action of contract, tort or otherwise, arising from, out of or in connection with this post or the use or other dealings in this post.

[DCS World]: https://www.digitalcombatsimulator.com/en/
[Unreal]: https://www.unrealengine.com/en-US
[HP Reverb G2]: https://www.hp.com/us-en/vr/reverb-g2-vr-headset.html
[Unity]: https://unity.com
[HTC Vive]: https://www.vive.com
[Valve Index]: https://store.steampowered.com/valveindex
[Meta Quest]: https://store.facebook.com/quest/products/quest-2/
[Revive]: https://github.com/LibreVR/Revive
[OpenComposite]: https://gitlab.com/znixian/OpenOVR
[OpenXR Toolkit]: https://mbucchia.github.io/OpenXR-Toolkit/
[OpenKneeboard]: https://github.com/fredemmott/OpenKneeboard
[Matthieu Bucchianeri]: https://github.com/mbucchia
[get in touch]: /contactme
