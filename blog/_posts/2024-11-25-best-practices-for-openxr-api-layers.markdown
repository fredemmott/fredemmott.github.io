---
layout: post
title: "Best Practices for OpenXR API Layers on Windows"
date: 2024-11-25 07:59:00 -0500
tags: [VR, OpenXR]
---

There are several re-occurring problems for OpenXR API layers on Windows, and best practices have evolved to address them; these problems are mostly caused by implementation decisions of other software in the ecosystem, so largely out of scope of the OpenXR specification - however, they still need to be addressed in order for your software to work well on a broad variety of end user systems.

This post is intended for developers of OpenXR API layers, but also contains some advice for developers of OpenXR runtimes, games, and game engines; it's intended to help you:
- avoid common problems
- maximize your software's compatibility
- find issues in your software before your users
- simplify and accelerate debugging, both for you and developers of other components in the OpenXR stack

Most of the advice in this post can only be applied by OpenXR *developers*, not end-users.

This post is based on my experience developing [OpenKneeboard](https://openkneeboard.com) and [HTCC](https://htcc.fredemmott.com), and investigating interactions between these layers and layers from other vendors.

## Don't use API layers if you can avoid it

OpenXR API layers exist in between the game and the runtime, can modify the behavior of any OpenXR function call, or implement extensions implementing additional functions. However, layers bring their own set of complications such as:
- conflicts
- ordering requirements
- dependencies that may be implicit
- changes to runtimes or later API layers may break your API layer, especially if additional extensions are implemented in an area relevant to your API layer

More generally, an API layer adds a moving part with many ways to misconfigure it; avoiding API layers reduces the possible number of configurations to support and test, and reduces the scope for user error.

Prefer to build functionality into the runtime, game, or engine instead if possible. API layers are only the best solution when either:
- you need to implement something that works in multiple games *and* multiple runtimes
- you are unable to modify either the game or the runtime, e.g. if you are making a third-party tool

## Use `HKEY_LOCAL_MACHINE` (HKLM) instead of `HKEY_CURRENT_USER` (HKCU) for OpenXR registry locations

*Applies to API layers and runtimes*.

- Register your layer or runtime in `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\OpenXR\<major_api_version>`
- DO NOT register your layer or runtime in `HKEY_CURRENT_USER\SOFTWARE\Khronos\OpenXR\<major_api_version>`
- Feel free to use `HKEY_CURRENT_USER\SOFTWARE\YourName\YourProduct` or similar for other settings

The order of OpenXR API layers is important; for example:
- `XR_APILAYER_FREDEMMOTT_HandTrackedCockpitClicking` *uses* `XR_EXT_handtracking`
- `XR_APILAYER_ULTRALEAP_hand_tracking` *implements* `XR_EXT_handtracking`

So, if an add-on Ultraleap hand tracker is being used, `XR_APILAYER_FREDEMMOTT_HandTrackedCockpitClicking` depends on `XR_APILAYER_ULTRALEAP_hand_tracking`, so the ordering needs to be:

1. Game
2. `XR_APILAYER_FREDEMMOTT_HandTrackedCockpitClicking`
3. `XR_APILAYER_ULTRALEAP_hand_tracking`
4. The active OpenXR Runtime

Ordering can only be controlled within a single registry hive - so, if layer A is in HKLM but layer B is in HKCU, the relative ordering of A and B can not be controlled. As the ordering frequently *needs* to be configurable, layers should aim to be installed in the same registry hive.

As the vast majority of existing OpenXR API layers are installed into HKLM, your layer should also be installed into HKLM, so that the order can be controlled in order to solve dependencies or mitigate conflicts or other ordering requirements.

## Sign your DLLs

*Applies to API layers and runtimes*.

All DLLs that get loaded into third-party processes should be [signed with a code-signing certificate that is trusted by Windows](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/authenticode); this is required by most anti-cheat software.

Even if your API layer or runtime is intended for a specific game that does not use anti-cheat and does nothing in other games, runtimes and enabled implicit API layers will still have their DLL loaded into any OpenXR game. If your DLL is unsigned, users will need to uninstall or disable your API layer/runtime in order to play any other OpenXR games that do use anti-cheat.

If your runtime is not signed, it is likely that users of your headset can not play any OpenXR  games that use anti-cheat.

## Use a timestamp server when signing your DLLs

*Applies to API layers and runtimes*.

Code-signing certificates have an expiration date; after this point, they are no longer valid. While signatures to include the time when the signature is created, this can easily be forged by changing the clock on the computer that does the signing. For this reason, when a certificate expires, signatures create by that signature also expire - so your DLLs become untrusted.

To solve this problem, your signatures can be cross-signed by a trusted third-party timestamp server; for example:

```
signtool ... /t http://timestamp.example.com
```

If you use a timestamp server that Windows trusts, Windows is able to confirm when the signature was created - so Windows will continue to trust the signature (and DLL) even if the certificate has expired.

As of the [CAB baseline requirements v3.9](https://cabforum.org/working-groups/code-signing/documents/) section 6.3.2:
- the maximum validity period for a code signing certificate is 39 months
- the maximum validity period for a timestamp certificate is 135 months

So, roughly, if you do not use a timestamp server, your DLLs are trusted for up to 3 years, if you sign them *immediately* after purchasing a new 3-year certificate. Using a timestamp server extends this to up to 11 years, regardless of when you purchased your certificate and your certificates validity period.

For details and recommended timestamp servers, see your certificate authority's documentation.

## Add a `VERSIONINFO` resource to all your binaries

*Applies to API layers, runtimes, and games*.

A [`VERSIONINFO`](https://learn.microsoft.com/en-us/windows/win32/menurc/versioninfo-resource) can contain vendor information, the name of your software, and the version; populate all of these, and make sure that your version information is automatically updated whenever your software's version is updated.

Windows automatically includes this information for all binaries when creating a minidump file; when there is a crash, including this information saves time for every developer involved in investigating the crash, regardless of where the fault lies. It is also shown in the 'Details' tab of file properties in Windows explorer.

## Set ACLs for 'All Packages' and 'All Restricted Packages'

*Applies to API layers and runtimes*.

Sandboxed applications use either the 'All Packages' or 'All Restricted Packages' identities; these identities do not have access to the vast majority of files that are readable by 'all users' or 'everyone'. Sandboxed applications include:
- WebXR in Google Chrome
- Some applications from the Microsoft Store, such as 'OpenXR Tools for Windows Mixed Reality'

If your project uses named pipes, shared memory, or other shared resources, you will also need to set ACLs for these.

Alternatives include:
- install into Program Files; this is the simplest and most common way to solve this problem, as the *default* ACL for Program Files allows access by these identities - however, some users and tools do change the permissions, so it is still better to explicitly set the ACLs
- install into ProgramData instead; this is largely equivalent to Program Files. While it has the benefit of allowing non-administrator installation, it has the downside of allowing modification without elevation. Administrator installation is also required anyway if you are following the best practice of installing your API layer or runtime into `HKEY_LOCAL_MACHINE` instead of `HKEY_CURRENT_USER`.
- runtimes can choose to not implement the `XR_EXT_win32_appcontainer_compatible` extension; this is not a possible solution for API layers, as API layers can not hide advertised extensions from games

Sandboxed OpenXR games on the Microsoft Store are rare; the practical problem here is the "OpenXR Tools for Windows Mixed Reality" tool:
- this tool is sandboxed
- while this tool is only intended for Windows Mixed Reality, there are a *lot* of user guides telling people to use this for any headset and runtime, especially within the flight and racing simulator communities

Even if this tool is not relevant to you or any games/runtimes you support, due to its popularity, I highly recommend making your software compatible with it anyway: supporting this tool will avoid a large amount of misguided support requests and negative reviews.

## If you support Vulkan, use or require `XR_KHR_vulkan_enable2`

*Applies to API layers, games, and game engines*.

Non-VR Vulkan games need to enable the Vulkan features that they use; when OpenXR is involved, the features required by the runtime and any OpenXR API layers must also be enabled. `XR_KHR_vulkan_enable2` was introduced because `XR_KHR_vulkan_enable` is not flexible enough to enable modern Vulkan features.

For example, OpenKneeboard uses `VK_KHR_timeline_semaphore`; this requires `VkPhysicalDeviceTimelineSemaphoreFeaturesKHR` to be in the `next` chain of `VkDeviceCreateInfo`. `XR_KHR_vulkan_enable` does not provide a way to specify this requirement, but with `XR_KHR_vulkan_enable2`, as the `vkCreateDevice` call goes through OpenXR, the OpenKneeboard API layer is able to modify the call to add its' requirements.

[**Every** conformant runtime](https://github.khronos.org/OpenXR-Inventory/extension_support.html#matrix) that currently supports `XR_KHR_vulkan_enable` also supports `XR_KHR_vulkan_enable2`; you should use `XR_KHR_vulkan_enable2` instead where practical. You **MUST NOT** mix-and-match the `XR_KHR_vulkan_enable` and `XR_KHR_vulkan_enable2` flows.

Alternatives:

- let the game crash when a component attempts to use a Vulkan feature that has not been enabled by the game
- additionally implement a Vulkan API layer to modify the behavior of `vkCreateDevice`
   - this repeats all the problems with OpenXR API layers
   - well-behaved OpenXR API layers should then attempt to only enable their behavior if the corresponding Vulkan API layer is in use, and the Vulkan API layer should also attempt to do nothing unless the corresponding OpenXR API layer is in use
   - this increases the risk of your software causing issues for non-VR games
   - past minor updates to Vulkan have broken ABI-compatibility for API layers; it seems likely this will continue, and you should expect to need to keep your Vulkan API layer up-to-date with changes to Vulkan itself, even if none are directly relevant to VR

`VK_KHR_vulkan_enable` carries the risk of your software crashing if other software in the stack uses modern Vulkan features.

As `VK_KHR_vulkan_enable2` moves device creation to the OpenXR runtime, it does bring its own issues - especially for games/engines that switch between VR and non-VR. Migrating may be impractical for some games/engines. API layer developers may need to continue supporting `VK_KHR_vulkan_enable` and additionally implement their own *Vulkan* API layer.

## Implement `xrEnumerateApiLayerProperties` and `xrEnumerateInstanceExtensionProperties`

*Applies to API layers and runtimes*.

In the past, some vendors have removed or not implemented these as they appear to be unreachable code - however, they are invocable by API layers. They appear to be dead code as when they are invoked by an OpenXR application such as a game - or the OpenXR Conformance Test Suite - the implementation in the OpenXR loader is used instead, which reads from the JSON manifest files.

The behavior is described in [the OpenXR loader developer documentation](https://registry.khronos.org/OpenXR/specs/1.0/loader.html).

## Do not depend on `xrEnumerateApiLayerProperties` or `xrEnumerateInstanceExtensionProperties`

*Applies to API layers*.

While it is a best practice to implement these functions, it is not *required* by the specification; these functions may not be usable, and when available, implementation behavior varies and is not reliable.

Alternatives:
- a list of layer API layers is available in the `XrApiLayerCreateInfo` passed to `xrCreateApiLayerInstance`
- attempt to enable any extensions your API layer *can* use, even if they don't appear to be available. Handle `XR_ERROR_EXTENSION_NOT_PRESENT` from `xrCreateApiLayerInstance()`, and try again with a smaller set of extensions if it fails

This does not affect games or engines, as the OpenXR Loader's implementation of these functions uses the JSON manifest files instead of the API layer implementations.

## Test on multiple runtimes

*Applies to API layers, games, and game engines*

It is easy to accidentally depend on behavior that is not part of the specification; you should test with as many runtimes as you can to reduce issues.  For example, one vendor's runtime creates D3D12 swapchains that are incompatible with Microsoft's D3D11on12 framework; compatibility with this framework is not required by the OpenXR specification, but is often required by games or overlays.

Several vendors provide simulators that allow you to test compatibility with their runtimes without needing to have access to their hardware.

## Run the OpenXR Conformance Test Suite

*Applies to API layers and runtimes*.

While the CTS does not specifically test API layers, it can be used to check that the *combination* of your API layer and the active runtime still shows conformant behavior.

- API layers *MUST NOT* make conformant runtimes or conformant combinations of runtimes and other API layers exhibit non-conformant behavior.
- Run the full test suite; API layers often have a subtle impact outside the areas that seem relevant
- API layer developers: run the CTS as many runtimes as you can

## If you require a specific runtime, test for it

*Applies to API layers*.

Several vendors have provided PCVR support for their headset by implementing an OpenVR driver and several OpenXR API layers implementing additional OpenXR extensions; these API layers will often require SteamVR - but some crash if any other runtime is used.

While ideally such API layers would function on all runtimes, at a minimum, they should gracefully degrade on other runtimes; for example, if an API layer provides `XR_EXT_hand_tracking` but is unable to function in the current environment, it should:
- pass through calls verbatim if `XR_EXT_hand_tracking` is also provided by the runtime or a later OpenXR API layer
- otherwise, set `supportsHandTracking` to `FALSE` in `XrSystemHandTrackingPropertiesEXT`, and return `XR_ERROR_FUNCTION_UNSUPPORTED` from `xrCreateHandTrackerEXT()`

You should [test your API layer with multiple runtimes](#test-on-multiple-runtimes), *especially* if your API layer is only intended to be functional with a specific runtime - this includes running the OpenXR conformance test suite.

If your API layer crashes on other runtimes, this causes issues for:
- users who switch between multiple runtimes (e.g. a hardware manufacturer's runtime and Virtual Desktop)
- people who upgrade their headset
- people who switch between multiple headsets (e.g. developers)
- game and engine developers when they see crashes caused by your software, even when the relevant hardware or runtime is not in use
- your support staff, as they need to understand and deal with these crash reports

## Disclaimer

I would like to thank the Khronos OpenXR Working Group for their feedback and suggestions.

The opinions in this post are my own; in particular, this post does not express the opinions of The Khronos Group, the Khronos OpenXR Working Group, or its other members.

OpenXR™ and the OpenXR logo are trademarks owned by The Khronos Group Inc., and are registered as a trademark in China, the European Union, Japan, and the United Kingdom.
