---
layout: post
title:  "In-Game Overlays: How They Work"
date: 2022-05-31 18:12:00 -0500
tags: VR overlays graphics
---

This post aims to give developers a high-level understanding of how in-game overlays work in a variety of
environments: non-VR, Oculus, SteamVR, and OpenXR. Non-VR overlays are often used for social features and
notifications, like the Steam, Discord, and EA overlays; VR expands the use cases and technical requirements.

<!--more-->

I'll be focussing on Direct3D 11 in this post, but it roughly applies to D3D12, OpenGL, and Vulkan as well.

## Non-VR overlays

DirectX does not have a supported way for one application to draw on top of another, so applications have
to do pretty hacky things. Let's start with a simplified view of how games draw to a monitor:

![Game -> Direct3D -> GPU Driver -> GPU -> Monitor](/assets/images/2022-05-overlays/no-overlays.svg)

Something wanting to draw an overlay has to find a way to fit in to this pipeline, and this problem isn't
specific to overlays: anything wanting to change how a game looks has the same problem, such as [ReShade] or
certain cheats. For NVIDIA's various overlays (mostly branded as GeForce), this seems to be relatively
straightforward: as NVIDIA provide the GPU driver (Graphics Processing Unit, a.k.a. graphics card), they can
add the functionality they need there. NVIDIA also use some of the later techniques, though it's not clear to me
why extending the graphics driver isn't sufficient for them.

Most developers don't have that option, so they need another approach. Their software will need two things:

* access to the games Direct3D state (`ID3D11Device`)
* the ability to modify every frame

For Direct3D, `IDXGISwapChain::Present()` is a convenient target for both of these: its' purpose is essentially
for a game to say "I'm done with this frame, send it to the monitor/window" - it is called for every frame,
can modify the frame, and has access to the Direct3D device (developers can convert the DXGI device to a
Direct3D11 device via `QueryInterface`). This leads to the next problem: how to change the behavior of
`IDXGISwapChain::Present` inside the game; this is usually split into two sub-problems:

* how do you run your code inside another process?
* once your code is running inside the other process, how do you change the behavior of `IDXGISwapChain::Present`?

### Running your code in another process

The main idea is to write a DLL that does what you want, then load the DLL into the game; Windows will run a
DLL's [DllMain] whenever your DLL is loaded (or about to be unloaded), which in turn means that if you write
a DLL main, Windows will run your code when your DLL is loaded. There's a few approaches for this:

#### Trick the game into loading your DLL

![Pretend to be Direct3D](/assets/images/2022-05-overlays/drawing-is-yes.png)

This is usually done by naming the DLL `dxgi.dll` or `d3d11.dll`, and putting it in the same folder
as the game executable. Direct3D games will then load these instead of the actual Direct3D components
from System32 - so your code will run, but your code will then need to load the 'real' `dxgi.dll` or
`d3d11.dll`.

This approach is often taken by shader modifiers, as it's relatively straightforward and doesn't need a launcher
or other program to be running to make it work. The biggest downsides are:

* game updates may remove the extra DLL or otherwise conflict
* each modification needs a different filename, and it needs to be one that the game already loads; for
  example, if both tools need to replace `dxgi.dll`, you can only use one.

Most modifications noawadays are designed to work as either `dxgi.dll` or `d3d11.dll` letting you easily run two,
and some will work as any DLL name that the game tries to load. If you use many mods, this quickly becomes
complex, and there is still a limit: you can only replace DLLs that the game would try to load.

#### Modify the game `.exe` to load your code

This is commonly done for cheats as they usually want to make other intrusive changes to the game anyway,
or for other changes like fixing support for ultrawidescreen monitors. It's not common to do this for
overlays - it can cause a lot of problems for users, so isn't suitable for widespread use:

* it invalidates the executable signatures, so will not work on many users' systems
* future updates are unlikely to work; a reinstall - and updated modification - will likely be required
* even when the modification is not a cheat, it is the most likely approach to be flagged by anti-cheat
  software

There are three common kinds of changes:

* modify the list of DLLs that the game loads
* modify the game's code to call `LoadLibrary()` with a path to your DLL
* modify the game's code to directly include the changes you want

#### Act as a launcher, and modify the game in-memory to load your code before it starts

This is similar to the previous approach, but requires that the overlay application be used to launch
the game. This removes the signature and update problems, but - like all the approaches - still has a chance of
triggering anti-cheat software.

This approach involves using `CreateProcess()` with the `CREATE_SUSPENDED` flag, making the changes in memory,
then using `ResumeThread` to actually start the process. Any of the changes that would work on disk also work
here, though modifying the DLL list is probably the most common; `DetourCreateProcessWithDlls()` is a convenient
way to do this if you're already using Microsoft's [Detours] library.

The biggest disadvantage is that most launchers only support loading their own DLL and can't be chained - i.e.
you generally can't use more than one modification that takes this approach.

#### Load the DLL after the game is already running

This has several advantages: there's no practical limit on how many modifications can take this approach,
and you don't need to remember to set things up before you start the game. However, it is the hardest approach
to do reliably: for all the other approaches, your code is loaded as soon as the game starts, before it does
anything else. You know that Direct3D hasn't been initialized yet. If your code loads later, perhaps
that's still the case, or maybe the game's been running for a few hours - it must handle every case.

There are many ways to load a DLL into another process on Windows; two of the most common ones are:

* Use `SetWindowsHookEx()` to hook [Window messages] for either a specific thread (including threads in another
  process), or for all threads on the system
* Use `CreateRemoteThread()` to create a new thread in the game process that loads your DLL

I prefer the `CreateRemoteThread()` approach as it targets a process, rather than a specific thread/window,
but in addition to the usual challenges of multithreaded code, it is also often useful to run code in
the thread that owns the window.

`SetWindowsHookEx()` isn't a universal problem for those solutions either though - you may need to handle
the case that a thread was created for a splash screen or other temporary window and no longer exists - or
may simply be the wrong thread/window for the game itself. Those problems can be mitigated by hooking all
threads in the system, but this increases the risk of unintended side effects or false positives from
anti-malware software.

`CreateRemoteThread()` needs combining with some other details to do this; the approach I use in
[OpenKneeboard] is:

1. use `OpenProcessToken()` and `AdjustTokenPrivileges()` to give my app debug privileges (this does not
  require UAC/administrator)
2. use `VirtualAllocEx()` to allocate memory in the game process to hold the path to my DLL
3. use `WriteProcessMemory()` to write the DLL path to that memory
4. use `GetModuleHandle()` and `GetProcAddress()` to find the address of `LoadLibraryW()`
5. by a very handy coincidence, `LoadLibraryW()` is ABI-compatible with `LPTHREAD_START_ROUTINE` - so call
  `CreateRemoteThread()`, passing the address of `LoadLibraryW()` as `lpStartAddress`, and the address we got
  from `VirtualAllocEx()` as the `LPVOID` `lpParameter`

There's two common reasons this may not work:

* the game is running as administrator but the overlay application isn't; to fix, don't run the game
  as administrator 😉 If you are using a launcher (e.g. Skatezilla's launcher for DCS World), don't run
  the launcher as administrator either
* anti-cheat detects and blocks this. To fix, work with the anti-cheat vendors to recognize your software and
  allow it

### Drawing your overlay

Now you need to make it so that when the game tries to use `IDXGISwapChain::Present()` it actually uses your
code instead. If you're modifying the game exe on disk or doing something game-specific in memory, it might be
easiest to find and directly change the call sites in the exe. If you don't want to do that, or if your code is
running in a DLL in an unmodified game, you can
instead change `IDXGISwapChain::Present()` to call your code. This again has subproblems:

1. Find the address of `IDXGISwapChain::Present()`
2. Figure out how to change the start of it to call your code
3. Figure out how to call the original code; you're going to have to overwrite some of it with the code to call
   your function, so, you'll need to copy the original code somewhere else, modify it, and jump back to the
   remainder
4. Replace the start of the original function with your code from step 2

Your code will usually do its own thing (e.g. drawing your overlay on to the swap chain's active buffer), then
after doing some modifications go back to the original `IDXGISwapChain::Present()` code:

![`IDXGISwapChain::Present` modified to call the overlay DLL](/assets/images/2022-05-overlays/dxgi-hook.svg)

#### Finding the address

Finding the address of a public C-ABI function in a DLL is easy via `GetProcAddress()`; C++ can be much more
complicated, but in this case, we can take advantage of the fact and `IDXGISwapChain` is a COM interface,
so we can retrieve the address of the implementation via the COM C API:

```C
void* Find_IDXGISwapChain_Present(struct IDXGISwapChain* swapChain) {
  return swapChain->lpVtbl->Present;
}
```

This code must be compiled as C, not C++: when compiled as C++, `lpVtbl` is not defined. There's also another
problem: this returns the `Present()` function for a particular `IDXGISwapChain` instance, but we don't have an
instance.

If your DLL is loaded as the application starts, the most reliable way to get one is to also hook
`D3D11CreateDevice()`, `D3D11CreateDeviceAndSwapChain()`, and `IDXGIFactory::CreateSwapChain()` via the same
techniques; however, if your code is loaded once the application is already running, you need another approach:

Fortunately, all `IDXGISwapChain`'s appear to have the same `Present()` function, so you can create your own
Direct3D device and swap chain, and pass that to the `Find_IDXGISwapChain_Present()` function above - however,
this is an unsupported implementation detail of Windows/Direct3D, and could change without warning in a future
Windows/Direct3D update. I'm personally fine with that given that's true of drawing overlays on Direct3D in
general, not just this specific part.

#### Modifying the function

This is hard, as CPU instructions have varying lengths, layout requirements, and you also need to restore the
original registers/stack - except when those correspond to parameters you've changed; Microsoft's Detours wiki
has [more details](https://github.com/microsoft/Detours/wiki/OverviewInterception) on this problem.

Fortunately, this is a common enough problem that there are convenient libraries for this - the most popular are:

* Microsoft [Detours]
* [MinHook]

Previously, Detours's free version was severely limited - however it is now entirely open source, incluing x64 support. These libraries do the vast majority of the heavy lifting of rewriting the functions, and a library is practically essential for doing this kind of work in a reliable way.

### Problems

#### Brittleness

While Detours is a Microsoft project, it does not make using it to rewrite parts of Direct3D 'supported';
there is no supported way to do this, so all we can aim for is 'good enough'.

#### Conflicts

Pretty much every overlay application needs to change `IDXGISwapChain::Present()`; whichever gets there first
will work fine, but when the second one comes along, it might not be able to understand the
already-rewritten-once version; the more overlay applications you run, the more likely it is that you'll have a
problem, regardless of how they load into the game.

If you have trouble with the game crashing, try stopping any other overlay applications or disabling their
overlay functionality (e.g. MSI Afterburner, Discord, Steam overlay, RivaTuner, NVIDIA overlay, EA overlay); if
you're a user and find which combination fails, try reporting to the developers of both overlays; they may be
able to find a way to make them work nicely together.

One way for a developer to make things work nicely together is to make yours piggy-back off the other. As a real
example, if [OpenKneeboard] was loaded in after the Steam overlay, games would crash; I fixed this
by checking if the Steam overlay DLL is loaded, and if so, instead of rewriting `IDXGISwapChain::Present`, I
use the same techniques to modify the wrapper function in the Steam overlay DLL and insert my overlay there.
This led to another problem: the Steam DLL does not export this function or
provide a documented way to find it - so I resorted to [pattern matching the native code].

#### Anti-cheat false positives

Everything here is modifying the game to do stuff it wasn't meant to do; the problems and solutions are shared
with cheats. Signing your DLLs *may* help, but really the only way to fix this is to stick to games without
anti-cheat, or work with the anti-cheat maintainers so that they recognize and allow your software.

## Virtual Reality

While many appreciate non-VR overlays like Steam and Discord, the VR community has a much more pressing
need for them: VR players can't quickly look at another window, another monitor, or alt-tab. The technical
requirements also change:

* even if a flat HUD-style display is wanted, it needs distortion correction, and usually to be rendered in both
  eyes
* in-world positions are often wanted, like 'on my wrist', 'by my feet', or in the case of OpenKneeboard, it's
  intended to show information on your knee while sitting and playing a flight simulator

Fortunately, every VR API has the concept of visual layers - like the world itself and any in-game HUDs or
menus. These layers all have textures (images), and can have varying types, like 'world view for both eyes', or
'WxH rectangle at (x, y, z)' - so the goal here is to add an extra layer with a new texture, rather than to
modify the texture that the game is already creating.

Some VR APIs also provide a way to specify the coordinate system - you can then choose whether `(0,0,0)` means:

* user-chosen seated 'center position' at eye level
* user-chosen standing position at floor level
* other less well-defined positions

Available options vary by API and by headset. If you're not able to choose, you will need to retrieve the
controller or HMD positions, and apply the desired translations yourself; if there isn't enough information
available, you may need to build a recentering feature. For the Oculus API, there is a setting, but it applies
to all layers; you must check what the game developers chose, and do the math as needed to get the results you
want.

## Overlays with the Oculus API

Like non-VR Direct3D, the Oculus API does not provide a way for applications to add overlays (layers) to
other games, and we essentially need the same techniques: load our code, and rewrite some functions. The Oculus
API is designed so that at that the end of every frame, the game submits a list of layers, their descriptions
('WxH rectangle at (x, y, z)`), and textures; if we intercept these calls, we can add additional layers.

Modern code should use `ovr_EndFrame` (or better, OpenXR instead of the Oculus API), however older code may also
use `ovr_SubmitFrame` or `ovr_SubmitFrame2`; while the difference is important to game developers, from the
perspective of an overlay, these can be treated identically - and conviently have identical signatures.

Small overlays will want to add an `ovrLayerQuad`; larger overlays may want to use an `ovrLayerCylinder` to
provide some curvature.

While we no longer need to hook `IDXGISwapChain::Present` for every frame, it can still be a convenient way to
get the game's Direct3D device, which you will need to optain to pass to `ovr_CreateTextureSwapChainDX()`.

### Problems with Oculus API Overlays

Oculus API overlays have the same potential problems as non-VR Direct3D overlays: brittleness, conflicts, and
anti-cheat issues. There's also a few extra quirks to keep in mind:

#### Layer limit

The Oculus API has a fixed limit on the number of layers: this limits the number of overlays you can use, which
can vary by game, and by state: it's possible that pausing the game will add an extra layer, so pausing the game
might hide your overlay. dditionally, some games - like the Oculus World Demo in the SDK - give a list of layers
that is already the maximum size.

In these cases, it's likely that not all of the entries are in use; in particular, entries in the list can be a
`nullptr`, in which case they have no effect, so you can simply remove them from the list, making space for your
own layer to be appended.

#### Interaction with depth data

When games provide the main 'world view' layer, they can provide depth information along with the RGBA (color)
data for each pixel; this is required for some of the framerate-compensation technologies like [ASW 2.0], but
also interacts with the other layers in the list - later layers appea not to be rendered if the depth data from
another layer says they will not be visible because they're behind the other layer in 3D space.

Some games appear to provide incorrect depth data which completely disables all overlays - even the
Oculus Debug Tool performance HUDs. This can be fixed by replacing any layers with `ovrLayerType_EyeFovDepth`
with copies set to `ovrLayerType_EyeFov`. While `EyeFovDepth` is documented as being required for some
framerate-compensation technologies, it seems unlikely that they were working as intended if the depth data was
incorrect.

## Overlays with SteamVR

This is where things get better: Valve saw the need for third-party overlays -perhaps due to their experience
with the non-VR Steam overlay - and made it a built-in feature of SteamVR/OpenVR:

![The game and overlay are separate processes, independently talking to SteamVR via OpenVR](/assets/images/2022-05-overlays/steamvr.svg)

There is no need for the overlay to interfere with the game: the game and the overlay app are separate processes,
independently communicating with SteamVR via OpenVR. SteamVR is responsible for setting up the layers,
coordinating input, and so on; SteamVR will also manage translating between Direct3D 11, 12, OpenGL etc as needed.

### Problems with OpenVR/SteamVR overlays

While the presence of an overlay API is a huge improvement and the APIs themselves seem fine, there are some
long-standing issues in the implementation that overlay developers should be aware of:

#### Use `vr::TextureType_DXGISharedHandle`

While OpenVR supports image files, raw pixel data, normal textures, and DXGI shared handles, the first two are
extremely slow, and the first 3 can flicker, and stop working entirely after a few hundred frames. For
long-running or high-framerate overlays, use DXGI shared handles for reliability and to avoid flickering.

These must be created via the legacy `IDXGIResource::GetSharedHandle()` function on a texture created with
`D3D11_RESOURCE_MISC_SHARED` - OpenVR does not appear to support `D3D11_RESOURCE_MISC_SHARED_NTHANDLE` and
`D3D11_RESOURCE_MISC_SHARED_KEYEDMUTEX`.

#### Don't poll `VR_Init()` or `VR_IsHmdPresent()` frequently

These functions [leak memory](https://github.com/ValveSoftware/openvr/issues/310); `VR_Init()` is required,
but use a different technique first to reduce the number of calls and the speed of the leak. For example,
OpenKneeboard only calls `VR_Init()` if SteamVR's `vrmonitor.exe` is running.

#### Use `vr::TrackingUniverseStanding`

`TrackingUniverseSeated` overlays [do not work correctly](https://github.com/ValveSoftware/openvr/issues/830).

## OpenXR

[XR_EXTX_overlay] is a provisional extension to OpenXR which would allow separate overlay applications in a very
similar way to SteamVR, however it's not yet widely available. While OpenXR does not currently have a fully
supported overlay API, it does provide a much more flexible system: [API layers].

API layers provide a supported way to insert DLLs wrapping any OpenXR functionality, and there is even
[a test implementation of XR_EXTX_overlay as a layer](https://github.com/LunarG/OpenXR-OverlayLayer). In the
case of overlays, our goals are very similar to when hooking the Oculus API: we want a Direct3D device which we
can get by intercepting `xrCreateSession()`, and we want to add a layer by intercepting `xrEndFrame()`. The key
improvement is that by design it will load our DLLs without unsupported hacks, and we do not need to use Detours
or similar code rewriting techniques to intercept or wrap functions.

In the simplest case with no extra layers, the OpenXR "trampoline" - part of the OpenXR loader - will simply
'bounce' everything on to the active OpenXR runtime:

![Game -> OpenXR Trampoline -> OpenXR Runtime](/assets/images/2022-05-overlays/openxr-no-layers.svg)

The OpenXR loader will also look for information on installed layers in the registry and environment variables;
if it finds any, it will automatically load them into this pipeline, transparently to the game. For example,
if a single layer wants to wrap `xrEndFrame()`, the result should look like this:

![Calls to `xrEndFrame()` split off to overlay DLL; everything else goes direct to runtime](/assets/images/2022-05-overlays/openxr-single-layer.svg)

There can also be multiple layers, which are also all transparent to the game and other layers:

![Chaining two overlays together](/assets/images/2022-05-overlays/openxr-multiple-layers.svg)

While I've used two layers that both intercept `xrEndFrame()` for this example, it could be any OpenXR function,
or they could intercept different OpenXR functions. DLL injection and Detours, you are unable to install a new
API layer into a game that is already running - but, it's reasonable to have your DLL always be active, as long as
it is designed to have near-zero overhead when inactive.

## Advice

Several VR SDK vendors decided to include basic matrix math libraries; I recommend using a separate matrix library
instead, such as [DirectXTK]'s [SimpleMath]:

* some of the vendor matrix libraries are buggy
* if you support multiple VR APIs, sharing math code is useful
* it is unclear if the licenses of some vendor SDKs permit using their matrix code in code that does not target
  their headsets

If you're supporting any flow where your code is running in someone else's process (every flow here except for
SteamVR or XR_EXTX_Overlay) , I strongly recommend using native code - like C++ - instead of something that
will also load the .NET runtime into the game's process. 

More generally, if you have a DLL that is loaded into another process (including via the OpenXR loader), I
recommend doing as little as possible in that DLL: if you have bugs, it is usually much better for those bugs
to take down a separate overlay application than the game. For example, [OpenKneeboard] does the vast majority
of the work in the `OpenKneeboardApp.exe` process, but creates shared memory and shared textures to communicate
with the DLL; the DLLs do the bare minimum - they install the hooks (if needed), set up the layers, and copy the
textures:

![OpenKneeboard's OpenXR layer runs in the game process, but displays information from the OpenKneeboard process](/assets/images/2022-05-overlays/openxr-openkneeboard.svg)

I use the same approach for Oculus and non-VR overlays.

## Real-world example

OpenKneeboard currently supports overlays with:

* SteamVR
* Non-VR Direct3D 11
* The Oculus API combined with either Direct3D 11 or Direct3D 12
* OpenXR Direct3D 11 games, via a custom API layer

This code is available in [`src/injectables`](https://github.com/fredemmott/OpenKneeboard/tree/03c9ad1873d2e8b28b6f836b881338133db9253a/src/injectables) folder of the repository, and is under the GPLv2 license.

## Disclosure

I used to work for Meta, but in an unrelated area (programming languages). This post and it's opinions are
purely my own, and soley based on public information. I use an Oculus headset, which I purchased retail, and
was not reimbursed for it.

[ASW 2.0]: https://www.oculus.com/blog/introducing-asw-2-point-0-better-accuracy-lower-latency/
[`DetourCreateProcessWithDlls`]: https://github.com/microsoft/Detours/wiki/DetourCreateProcessWithDlls
[ReShade]: https://reshade.me
[MinHook]: https://github.com/TsudaKageyu/minhook
[Window messages]: https://docs.microsoft.com/en-us/windows/win32/learnwin32/window-messages
[Steam Overlay]: https://help.steampowered.com/en/faqs/view/3978-072C-18DF-FBF9
[Detours]: https://github.com/microsoft/Detours
[OpenKneeboard]: https://github.com/fredemmott/OpenKneeboard
[pattern matching the native code]: https://github.com/fredemmott/OpenKneeboard/blob/03c9ad1873d2e8b28b6f836b881338133db9253a/src/injectables/IDXGISwapChainPresentHook.cpp#L39-L68
[XR_EXTX_overlay]: https://www.khronos.org/registry/OpenXR/specs/1.0/html/xrspec.html#XR_EXTX_overlay
[API layers]: https://www.khronos.org/registry/OpenXR/specs/1.0/html/xrspec.html#api-layers
[DirectXTK]: https://github.com/Microsoft/DirectXTK
[SimpleMath]: https://github.com/microsoft/DirectXTK/wiki/SimpleMath
