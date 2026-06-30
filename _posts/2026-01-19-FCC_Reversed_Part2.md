---
title: Reverse Engineering of FCC Unlocks in DJI Fly clones - Part 2
description: >-
  Understanding the core concepts around the FCC bypassing logic.
author: klcium
date: 2026-06-29 20:55:00 +0800
categories: [Reverse Engineering]
tags: [Drone, FCC, DJI, Reverse]
pin: false
published: true
---


## Introduction

In the [first part of this series](/posts/FCC_Reversed_Part1), we focused on the modified application's architecture. We identified a customized Frida Gadget embedded inside the APK, recovered the obfuscated JavaScript payload and reconstructed its original logic through a series of deobfuscation passes. By the end of that analysis, one question remained unanswered:

> What does the payload actually do once it starts executing?

In this second part, we will follow that execution flow step by step, starting from the anti-tampering bypass and ending with the mechanisms used to keep DJI Fly in a consistent FCC-compatible state throughout its execution.


## Bypassing DJI's packer

One of the first interesting behaviors implemented by the hidden script is the interception of Android's dynamic linker. More specifically, it hooks two internal linker functions:
- `do_dlopen()`
- `call_constructor()` ([source](https://cs.android.com/android/platform/superproject/+/android-latest-release:bionic/linker/linker.cpp))

If you are interested in Android's library loading process, I wrote [another article](2025-08-20-DT_NEEDED.md#android-library-loading-process) covering it in detail. Only the relevant parts are summarized here. `do_dlopen()` is the function ultimately responsible for loading native shared libraries requested through `System.loadLibrary()`.

Once the library has been mapped, relocated, and linked, `do_dlopen()` invokes `call_constructor()`. The latter executes the ELF initialization routines, namely the functions referenced by the `DT_INIT` and `DT_INIT_ARRAY` entries ([source](https://cs.android.com/android/platform/superproject/+/android-latest-release:bionic/linker/linker_soinfo.cpp;l=494)).

These initialization functions execute **before** `JNI_OnLoad()` and even before the library's public entry points become reachable. 
> Many commercial packers leverage this stage to unpack or decrypt their payload before any analysis can begin. DJI's native packer, `libAppGuard.so`, follows exactly this approach.  

Intercepting `call_constructor()` therefore gives the script a chance to hook the packer immediately before its initialization code executes.

### Hooking the Android linker

The JavaScript first hooks `do_dlopen()`. Each time a shared library is loaded, its filename is inspected. When the loaded library is **not** `libAppGuard.so`, execution continues normally. When `libAppGuard.so` is detected, the script prepares a second hook targeting its constructor phase.

This second hook executes just before AppGuard's initialization routines. At that point, the script scans the executable memory of the library for the ARM64 instruction:

```text
01 00 00 D4
```

This instruction corresponds to the `SVC` (Supervisor Call) opcode. On Android, every system call eventually reaches the kernel through an `SVC` instruction.

> Next paragraph is evasive on purpose, sorry.

Whenever an intercepted `SVC` corresponds to the `openat` system call, the hook inspects its arguments. If the requested filename is `[Sorry_no_lawsuite_plz.txt]`, the hook transparently replaces the filename with the embedded `[censored.txt]`. From the application's perspective, the operation succeeds normally. However, instead of reading a specific file for anti-tapering, AppGuard receives an otherone, passing anti-debug checks. 

To my knowledge, the packer should also inspect memory but I no relevant bypass on this side was found. This will require more investigation in the future.




## Bypass of the FCC

Now we will see how it actually enable FCC mode.

> I can't provide the script to avoid getting into too much trouble (I hope).

The embedded Frida runtime builds what can be described as a **state synchronization layer** around DJI Fly. Instead of forcing one configuration value, it continuously intercepts every subsystem involved in regional management and keeps them synchronized around the same fabricated state.

From my analysis, the implementation can be divided into four major stages:
1. runtime initialization,
2. flight limit virtualization,
3. area code synchronization,
4. persistent state management.


### Runtime initialization

The initialization logic executes only once during the application's lifetime.

One of its first tasks is to initialize a small auxiliary DEX bundled with the application. The DEX itself contains only a handful of stub implementations of generic Android callback classes, including `Runnable` and `OnClickListener`. Rather than implementing application logic directly, those classes serve as generic dispatchers between Java and the embedded JavaScript runtime.

Conceptually, their behavior is similar to the following:

```java
public void run() {
    Callback cb = callbacks.get(getCallbackId());
    cb.execute();
    callbacks.remove(getCallbackId());
}
```

The real implementation is installed dynamically through Frida's Java API.

Whenever one of those Java callbacks executes, it retrieves a callback identifier, resolves it through an internal callback registry maintained by the JavaScript runtime, invokes the associated JavaScript function and immediately removes the mapping. This design provides a lightweight bridge between Java and JavaScript without embedding any application-specific logic inside the DEX itself.

The initialization routine also installs hooks preventing the application's update mechanism from completing. Although the exact implementation differs across DJI Fly releases, the objective appears straightforward: prevent the application from downloading an update that would invalidate the installed runtime hooks.

Finally, the runtime prepares its internal state and selects the appropriate hooking routine. Interestingly, this selection is not based on a hardcoded application version. Instead, the script derives a checksum from one of its embedded resources and uses it to resolve the function responsible for installing the remaining hooks.

This indirection allows the same runtime to support multiple payloads without relying on large version-specific switch statements.

### Flight limit virtualization

![cat_plane](/assets/img/shitpost/kittens/cat_plane.jpeg)

Spoofing the country code alone is not sufficient as DJI Fly does not expose operational limits from a single location.

Altitude limits, return height, maximum distance and other regulatory parameters are distributed across several abstraction layers. Some values originate from UI view models, while others are retrieved from more central SDK objects before being presented to the user.

Changing only one of those values would quickly produce an inconsistent application. For example, the user interface could display one altitude limit while another subsystem continued enforcing a different one. Instead of patching individual settings, the runtime intercepts every layer responsible for exposing those values.

For newer application versions, the implementation goes even further by hooking generic bounded value objects shared throughout the application. Those objects expose minimum, maximum and current values and are reused by multiple features.


### Area code synchronization

Area management turned out to be the most sophisticated part of the implementation.

DJI Fly continuously recomputes its operating region using multiple independent information sources.

Rather than relying on a single country variable, regional information propagates through dedicated callback interfaces carrying both the resolved country and the strategy that produced it.

Observed strategies include:
- drone GPS;
- phone GPS;
- mobile network information (MCC);
- IP geolocation;
- cached values;
- internally generated updates.

Instead of attempting to disable this machinery, the runtime integrates directly into it. Some hooks intercept existing notifications before they propagate through the application. Others explicitly invoke the same callback interfaces with synthesized country information, effectively generating new area change events whenever synchronization becomes necessary.

A simplified version of the observed behavior can be represented as follows:

```text
Area update detected
        |
        v
Intercept callback
        |
        +------ Existing update --------+
        |                               |
        +--> Replace country -----------+
        |
        +--> Or trigger a new callback
             with spoofed parameters
```

The implementation also adapts itself to different DJI Fly releases. Depending on the detected application version, different callback implementations are selected while preserving the same high-level behavior.


### Persistent state management

Intercepting callbacks alone would not be sufficient. DJI Fly appears to cache regional information in several runtime objects as well as persistent storage. Without additional synchronization, newly created components could still observe stale values originating from previous sessions.

To avoid this situation, the runtime also intercepts writes to the application's persistence layer. Whenever regional information is written, the stored values are replaced with the spoofed ones before reaching the underlying storage implementation. The same strategy is applied to several runtime caches and event objects.


### Putting everything together

At a high level, the execution flow can be summarized as follows:
1. Bypass the application's integrity verification.
2. Start the customized Frida Gadget.
3. Initialize the Java <-> JavaScript bridge.
4. Disable automatic application updates.
5. Validate the current session and license.
6. Select the appropriate runtime payload.
7. Virtualize APIs exposing regulatory limits.
8. Synchronize area code notifications.
9. Keep the spoofed state consistent across runtime objects and persistent storage.


Rather than modifying firmware or forcing one configuration value, the developers built a runtime synchronization layer that continuously maintains a coherent state throughout the application. User interface components, SDK models, business logic, event dispatchers and persistence mechanisms all observe what appears to be the same legitimate operating region.

## Conclusion

The implementation turned out to be significantly more sophisticated than what I expected: the modified application bypasses native integrity verification and builds a synchronization layer around the application's regional management logic.

Rather than patching one function, the runtime continuously keeps multiple independent subsystems synchronized around the same fabricated state. Regional events, cached values, operational limits, SDK objects and persistent storage all remain consistent throughout the application's execution.