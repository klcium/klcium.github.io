---
title: Reverse Engineering of FCC Unlocks in DJI Fly clones - Part 1
description: >-
  Diffing analysis and deobfuscation of an FCC APK.
author: klcium
date: 2026-06-29 20:55:00 +0800
categories: [Reverse Engineering]
tags: [Drone, FCC, DJI, Reverse]
pin: false
published: true
---


## Introduction

Modern Unmanned Aerial Vehicles (UAV) are heavily regulated. Depending on the country, pilots must comply with airspace rules, operational limits, and radio regulations.

To enforce these requirements, manufacturers such as DJI automatically adapt the drone's behavior to the local legislation. In Europe, drones operate in **EC mode**, which applies stricter limits than the **FCC mode** applicable in the United States.

Unsurprisingly, some users started modifying their drones to bypass these restrictions. Most existing solutions patch the firmware, but this comes with several drawbacks: warranty loss, increased risk of detection, and the possibility of permanently bricking the device.

More recently, commercial solutions appeared that claimed to enable FCC mode **without modifying the drone firmware**. Instead, they distributed a patched version of the DJI Fly Android application. Given DJI's extensive anti-tampering mechanisms, this approach immediately caught my attention.

I eventually managed to obtain a copy of one of these APKs. In this article, I document the reverse engineering process and explain how the application continuously maintains DJI drones in FCC mode without modifying their firmware.

> Disclaimer: This article documents the technical implementation of the application. It should not be interpreted as an endorsement of bypassing regulatory restrictions.

### EC mode vs. FCC mode

DJI drones adjust their configuration according to the region where they operate.

In **FCC mode**, the drone uses the operational limits allowed under FCC regulations:

- Maximum altitude: 500 m
- Maximum speed: ~75 km/h
- Maximum range: ~15 km
- Radio power: up to 33 dBm on both 2.4 GHz and 5.8 GHz

In Europe, drones switch to **EC mode**, which applies stricter constraints:

- Maximum altitude: 120 m
- Maximum speed: ~68 km/h (DJI limitation)
- Line-of-sight operation, resulting in an 8 km software limit
- 2.4 GHz limited to 20 dBm
- 5.8 GHz limited to 14 dBm

The most significant difference is the transmit power. FCC mode allows up to 33 dBm, which substantially improves link quality in congested RF environments. A stronger radio link also increases the maximum practical flight distance. 

For most users, these two advantages alone explain the popularity of FCC unlocking.

![cat slaps drone](/assets/img/shitpost/kittens/cat_slaps_drone.gif)

> **I often say "FCC Bypass" to mean "The bypass of EC limitations via the continuous enablement of the FCC mode"**

## Reverse engineering the APK

> If you are only interested in the bypass itself, skip to [Bypass of the FCC - Part 2](/posts/FCC_Reversed_Part2/).

The application under analysis is a modified version of the official **DJI Fly** Android application. It claims to permanently enable FCC mode without reflashing the drone firmware. To understand how it works, I obtained both the patched APK and the original DJI Fly APK built from the same version and compared them.

### First observations

The first thing that stands out is the APK size. The patched application is almost twice as large as the original.

```bash
➜ ls -lh DJIFly.apk
561M DJIFly.apk

➜ ls -lh fcc.apk
1.1G fcc.apk
```

Something is clearly embedded into the application. Extracting both APKs and comparing their contents reveals only a handful of modifications:

- `classes.dex` has been modified.
- Five new files have been added to the native library directory.

```bash
107    libhack.config.so
1022K  libhack_js.so
27M    libhack.so
561M   liborg.so
1.3K   libgui.so
```

At first glance these all look like native shared libraries because they follow Android's `lib*.so` naming convention. However, Android only enforces the filename format. A file named `lib*.so` does not have to be an ELF shared object.

Checking their actual file types gives a more interesting picture.

```bash
➜ cat sussy.txt | xargs file

libhack.config.so: JSON text data
libhack_js.so:     ASCII text, with very long lines, with no line terminators
libhack.so:        ELF 64-bit LSB shared object, ARM aarch64, stripped
liborg.so:         Zip archive data
libgui.so:         Dalvik dex file version 035
```

Only one of the five files is actually a shared library. The others are simply disguised behind a `.so` extension.

*The plot thickens.*

Let's start with the easiest ones: `liborg.so` and `libgui.so`.

### Liborg.so and libgui.so

`liborg.so` immediately looks suspicious. Although stored as a native library, it is actually a ZIP archive. Computing its checksum confirms that it is simply the original DJI Fly APK embedded inside the patched application.

```bash
➜ md5sum liborg.so
111e4f2f113d8c30ade76f75688d5ab5 liborg.so

➜ md5sum DJIFly.apk
111e4f2f113d8c30ade76f75688d5ab5 DJIFly.apk
```

The patched APK contains an untouched copy of the original application.

Opening `libgui.so` reveals another surprise. It is not native code either but a small DEX file containing only two almost empty classes.

The interesting part is that both `run()` and `OnClickListener()` have empty implementations.

![customclass1](/assets/img/posts/2026-01-19-FCC_Reversed/customclass_1.png)

![customclass2](/assets/img/posts/2026-01-19-FCC_Reversed/customclass_2.png)

At this stage their purpose is still unclear.

### The libhack files

Three files remain:
- `libhack.so`: a genuine ARM64 shared library.
- `libhack_js.so`: a JavaScript file.
- `libhack.config.so`: a JSON configuration file.

### Libhack_js.so

Opening `libhack_js.so` immediately shows that it is valid JavaScript. It is also heavily obfuscated.

The file is approximately 1 MB, consists almost entirely of a single line, and contains more than 7,000 identifiers. Reading it manually is essentially impossible. A small extract illustrates the level of obfuscation:

```javascript
const _0x3b2f=['LmQzP','rT6vB'];
_0x1f4c=function(_0x8a1b,_0x2d3f,_0x5c6e,_0x9ab1,_0x4f20){
    return y(_0x2d3f- -0x2f,_0x8a1b);
};

_0x7d9a[_0x2c1e(0x4f1,0x2b3,0x6a,0x3d2,0x5b0)]
_0x1f4c(_0x6e2d(0xa2b),0x7f,0x1b4,0x3c7,0x9d)
...
```

At this point there is no indication of what the script actually does.

### Libhack.so

Unlike the other files, `libhack.so` is a legitimate ARM64 ELF shared library. 

We can confirm it's loading by inspecting the updated `classes.dex` file. In the latter, the first instruction of the constructor of the first class has been changed to load the `hack` library.

![classes_dex](/assets/img/posts/2026-01-19-FCC_Reversed/classes_dex.png)

Examining its dynamic section (of `libhack.so`) reveals something far more interesting.

```bash
➜ readelf -d libhack.so

Shared library: [libm.so]
Shared library: [liblog.so]
Shared library: [libdl.so]
Shared library: [libc.so]

Library soname: [libfrida-gadget-raw.so]
```

The SONAME of the library is **libfrida-gadget-raw.so**. This is a strong indication that the developers embedded **Frida Gadget** inside the application.

For readers unfamiliar with Frida, it is a dynamic instrumentation framework capable of injecting a JS runtime into a running process. It is widely used for reverse engineering, runtime hooking, memory inspection, and binary instrumentation.

Frida Gadget embeds the instrumentation engine directly inside the target application. At startup, it reads a configuration file that specifies how JavaScript should be loaded.

Two execution modes exist:
- **Remote mode**: the Gadget opens a TCP port and waits for a controller to connect.
- **Autonomous mode**: the Gadget immediately loads a local JavaScript file.

The configuration is stored in a file named `lib<agent>.config.so`, making the presence of `libhack.config.so` further indicating we are in the presence of a Frida gadget. Opening it confirms the hypothesis.

```json
{
  "interaction": {
    "type": "script",
    "path": "libhack_js.so",
    "on_change": "ignore"
  }
}
```

The configuration instructs the Gadget to execute the local script `libhack_js.so` immediately after initialization.

At this point we can already reconstruct the overall architecture.

- `classes.dex` loads `libhack.so`.
- `libhack.so` is a customized Frida Gadget.
- `libhack.config.so` configures the Gadget to run autonomously.
- `libhack_js.so` contains the instrumentation logic executed by the Gadget.
- `liborg.so` embeds the original DJI Fly APK, likely to bypass integrity or anti-tampering checks.
- `libgui.so` contains additional DEX classes whose purpose is not yet understood.

**The overall design is considerably more sophisticated than a simple APK patch. Instead of modifying large portions of the DJI application, the developers embedded a complete dynamic instrumentation framework capable of hooking the application at runtime.**

![happy cat dance](/assets/img/shitpost/kittens/povjisoo_cat_dance.gif)

Unfortunately, identifying Frida only answers **how** code is injected. It does not explain **what** the injected code actually does.

## Deobfuscating the JavaScript

The next objective was to understand what `libhack_js.so` actually does. For this, it has to be deobfuscated. The obvious first step was to try existing JavaScript deobfuscators but none of them produced usable results.

The two main issues were:
- They have no understanding of Frida-specific APIs. Many simplification passes require evaluating runtime objects, which is impossible without a complete Frida environment.
- The script is almost entirely contained in a single multi-megabyte line. Most deobfuscators eventually become unusable due to excessive object propagation, and even VSCode struggles to parse it.

Rather than trying more tools, I wrote a dedicated AST-based deobfuscator.

For readers unfamiliar with the concept, an **Abstract Syntax Tree (AST)** represents the syntactic structure of a program as a tree, making it possible to transform source code without executing it.

Working directly on the AST allowed me to simplify the script incrementally while preserving its semantics. I already covered this technique in [another article](/posts/ASTooGood), so I will not repeat the implementation details here. Moreover, I highly recommend you the website of William Khem Marquez [@Steak Enthusiast](https://steakenthusiast.github.io).

The obfuscator appears to rely on several classical techniques, including:
- string splitting;
- string decoding routines;
- alias chains;
- opaque predicates;
- unreachable code;
- encoded expressions.

Even after several simplification passes, the resulting script remained far from readable, but it was finally structured enough to understand its overall behavior.

After analysing the contents of the file, I observed that:
- In-script domains were different from DNS queries seen from dynamic analysis.
- There was no DJI class name / method and no patching logic.  

![gru_fail](/assets/img/posts/2026-01-19-FCC_Reversed/js_fail_meme.png)


### Analyzing the customized Frida Gadget
Since the JS file was not the answer and probably a mere decoy, I shifted the focus to the Gadget itself. The first objective was to determine which version of Frida had been modified.

Running `strings` on the binary quickly revealed several Frida-specific symbols.

```bash
➜ strings libhack.so

Android
r25b
8937393
...
FridaGadgetConnectInteraction
FridaGadgetLocation
bundle-name
Frida.Gadget.BaseController.join_portal
...
```

However, something immediately stood out: among the expected Frida strings were long obfuscated JavaScript fragments.

```text
... 0xF00D['type']='jaCa.'+FnX(...)
```

Those strings do not exist in the official Gadget and yes, they are valid JS code ! This strongly suggests that the developers modified the Gadget itself rather than simply embedding an official release.

At this point, I could have simply extracted the JS code and analysed it once again. But I was curious to know which part of the Frida gadget has been tampered.  Instead of treating the whole gadget as a black box, I decided to identify the original version it was based on and compare both binaries.

The benefits are significant:
- recover the original function names during reverse engineering;
- identify modified functions automatically;
- focus only on the vendor's changes instead of reversing the entire Gadget.

### Identifying the original Gadget version

Since the binary had been modified, comparing hashes was useless. Instead, I compared entropy graphs generated with `binwalk -E`.

> *Entropy* is a statistical measure of how random or structured a sequence of bytes appears to be. Executable code, compressed data and encrypted payloads typically exhibit different entropy profiles.  
> `binwalk -E` computes the entropy across a binary and plots it as a graph. The higher the entropy, the more random the underlying data appears. Entropy values close to "1" generally indicate compressed or encrypted data, whereas lower entropy regions usually correspond to structured content such as executable code, metadata, strings or other predictable data structures.


The customized Gadget produced the following profile:
![hack entropy](/assets/img/posts/2026-01-19-FCC_Reversed/entropy_graph_libhack.png)

After comparing multiple Frida releases, the closest match was:
```bash
frida-gadget-16.5.4-android-arm64.so
```

![gadget entropy](/assets/img/posts/2026-01-19-FCC_Reversed/entropy_graph_gadget.png)

Although the graphs are not identical, they share nearly the same structure. The largest difference is an additional data region around the `0.3e7` offset, while the remainder of the binary aligns closely enough for diffing. This was sufficient to recover most function names in Ghidra.

> Note: I had to recompile Frida at this specific version to get the symbols.

### Understanding Frida's source code

One point is worth mentioning before looking at the diff. Frida Gadget is primarily written in **Vala**, not C. During compilation, the Vala compiler generates intermediate C code, which is then compiled into the final binary. As a result, the function names visible in Ghidra are derived from the generated C code rather than the original Vala source.

For example, the following Vala method:
```vala
private async void scan () throws Error
```

becomes:
```c
frida_gadget_script_directory_runner_scan_co(...)
```

The generated C is difficult to read, but the naming convention is extremely useful. From a symbol such as:

```text
frida_gadget_script_directory_runner_scan_co
```

we can immediately infer:

- Class: `ScriptDirectoryRunner`
- Method: `scan()`

This makes it possible to navigate the original Vala source while reversing the compiled binary.

### Locating the modified code

After diffing the customized Gadget against the official release, one function immediately stood out: the function responsible for loading JavaScript had been modified. It has been modified to concatenate a large embedded string before creating the script.

Tracing its callers showed that it was invoked from `frida_gadget_script_load()`, allowing the function to be identified despite the missing symbols.

The original implementation is straightforward. You can check the original [gadget.vala](https://github.com/frida/frida-core/blob/e0ec2bee624b6ea11b72d5cc74639132449789c2/lib/gadget/gadget.vala#L379).

```vala
private sealed class Script : Object, RpcPeer {
  [...]
  private async void load () throws Error {
    load_in_progress = true;

    /*strduplicate(part 2 of the JS)*/

    try {
      var path = this.path;

      uint8[] contents;
      try {
        FileUtils.get_data (path, out contents);
      } catch (FileError e) {
        throw new Error.INVALID_ARGUMENT ("%s", e.message);
      }

      var options = new ScriptOptions ();
      options.name = Path.get_basename (path).split (".", 2)[0];

      ScriptEngine.ScriptInstance instance;
      if (contents.length > 0 && contents[0] == QUICKJS_BYTECODE_MAGIC) {
        instance = yield engine.create_script (null, new Bytes (contents), options);
      } else {

        /*RIGHT HERE!*/
        instance = yield engine.create_script ((string) contents, null, options);
      }

      if (id.handle != 0)
        yield engine.destroy_script (id);
      id = instance.script_id;

      yield engine.load_script (id);
      yield call_init ();
    } finally {
      load_in_progress = false;
    }
  }
}

```

Under normal circumstances, the Gadget simply reads the JavaScript file specified in the configuration and passes its contents to the scripting engine.

The patched version behaves differently: instead of directly loading `libhack_js.so`, it checks which script is being requested. When the requested file is `libhack_js.so`, the Gadget silently substitutes another JavaScript payload embedded inside the binary itself.

In other words, the visible `libhack_js.so` is almost **not** the script that actually executes. Instead, the real payload is hidden inside the customized Gadget. 

At this point, my understanding of the startup sequence was the following:
1. `classes.dex` loads `libhack.so`.
2. The shared object's initialization routines execute.
3. Frida Gadget starts.
4. The Gadget reads `libhack.config.so`.
5. The configuration requests the execution of `libhack_js.so`.
6. The patched `Script.load()` intercepts that request.
7. Instead of loading `libhack_js.so`, it reconstructs a hidden JavaScript payload embedded inside the Gadget and executes it.


### Overview of the embedded script

At this stage, it became clear that the visible `libhack_js.so` was merely a decoy. The next logical target was therefore the hidden JavaScript payload embedded inside the customized Gadget.

After extracting it, I saved it as `gadget.js` to distinguish it from the original `libhack_js.so`. Opening the file immediately revealed another heavily obfuscated JavaScript blob `>:(`.

![LP it starts with](/assets/img/posts/2026-01-19-FCC_Reversed/LP_itstartswith.png)

As if this wasn't funny enough, the obfuscation patterns were different and I had to rebuild a whole new deobfuscation script. Using Babel & Abstract Syntax Trees, I eventually managed to get a readable JS script from which I could rename the variables manually.

At this point, I could have used an AI (to rename the variables) but since I wasn't sure the copilot would backtrack all the references, I did it manually.

I eventually came up with a JS code that would look like this:
```js
const appConfig = {};
appConfig.identifier = "dji.go.v5";
appConfig.matchWholeWord = true;
runtimeFlags.loadPairedSymbol = true;
runtimeFlags.useLegacyMode = false;
runtimeFlags.enableStrictChecks = false;
runtimeFlags.injectHooksOnStart = false;
```

## Conclusion

So far, we've seen what FCC mode is and why some users seek to enable it permanently.

We then analyzed a modified DJI Fly APK claiming to keep FCC mode continuously enabled in order to understand how it achieves this.

The original DJI Fly APK was repackaged with an embedded Frida Gadget and a heavily obfuscated JavaScript payload controlling the instrumentation runtime. The actual script was embedded inside the customized Frida Gadget, while a decoy library was left in the native library directory.

The rest of the original application remains largely untouched, with most of the additional behavior being injected dynamically after launch.

Recovering the original script required building custom BabelJS deobfuscation passes by reverse engineering the obfuscation techniques and progressively reconstructing the original code.

In [Part 2 of this series](/posts/FCC_Reversed_Part2), we will examine how the script bypasses the application's anti-tampering mechanisms and continuously maintains DJI Fly in an FCC-compatible state.
