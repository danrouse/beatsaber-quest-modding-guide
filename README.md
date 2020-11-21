# Introduction to Modding Beat Saber for Oculus Quest 

This guide is intended to serve as a starting point for writing your own mods for Beat Saber for Oculus Quest. The modding scene for Quest is fairly small and fast-moving, so certain tools, techniques, and best practices may quickly become outdated, and instructional guides can be hard to find. To successfully write mods you'll need to be at least a little bit scrappy and resourceful.

Certain assumptions are made, like that you are at least vaguely familiar with software development on Windows, and can navigate a terminal. Most resources have been created with this in mind. Mods themselves are written in C++, and this guide does not aim to teach the language.

When in doubt, check out the [BSMG Discord #quest-mod-dev channel](https://discord.gg/beatsabermods), and try searching on Discord before asking questions. Browsing through source code of existing mods is another great way to familiarize yourself with coding patterns and practices.

Last updated: 20 November 2020, Beat Saber 1.13.0

**Table of Contents**
- [Prerequisites](#prerequisites)
  - [Oculus Quest setup](#oculus-quest-setup)
  - [Development environment](#development-environment)
- [Starting a new project](#starting-a-new-project)
- [Build scripts](#build-scripts)
- [Hooks](#writing-hooks)
  - [Finding methods to hook](#finding-methods-to-hook)
  - [Simple hook walkthrough: Modifying a menu](#simple-hook-walkthrough-modifying-a-menu)
  - [Simple hook walkthrough: Modifying gameplay](#simple-hook-walkthrough-modifying-gameplay)
- [Going further](#going-further)
  - [Logging](#logging)
  - [Sharing and distribution](#sharing-and-distribution)
- [Links](#links)
  - [Tools](#tools)
  - [Example repositories](#example-repositories)

---

## Prerequisites

### Oculus Quest setup

Beat Saber should already be modded with the latest [BMBF](https://bmbf.dev/stable), with Developer Mode enabled on your Quest.

**Tips and tricks**:
- To keep the Quest display from turning off when you remove it, you can place a piece of tape over the black sensor in the inside top-center of the headset, above the nose.
- Video can be streamed from your Quest to your computer through SideQuest to see the VR screen without putting the headset on.
- Wireless debugging can be enabled through SideQuest to develop without the Quest plugged in. This will quickly drain your battery and is not recommended.
- In development, you may want to use just a single controller and have extra AA batteries handy, since they'll spend a lot of time being on.

### Development environment

This guide assumes you're using [Visual Studio Code](https://code.visualstudio.com/), however it's not a hard requirement and plenty of modders use other IDEs, though some steps in this guide may differ.

- [Android NDK](https://developer.android.com/ndk/downloads) must be installed. Note its path for project template setup later.
- [Android SDK](https://developer.android.com/studio/releases/platform-tools#downloads) is strongly recommended and this guide assumes it is present in your `PATH`. It's necessary for running `adb` commands.
- [qpm (Quest Package Manager)](https://github.com/sc2ad/QuestPackageManager) is required and should be in your `PATH`. *Note*: qpm releases can be found by going to Actions on GitHub, to the latest build, and downloading the appropriate artifact for your system.

---

## Starting a new project

We will start from [Laurie's template](https://github.com/Lauriethefish/quest-mod-template). It's made to use the VSCode [Project Templates extension by cantonios](https://marketplace.visualstudio.com/items?itemName=cantonios.project-templates).

- *(TODO: flesh this out)* Create template from zip, create project from template.
- Add your NDK path to `includePaths` in `.vscode/c_cpp_properties.json`, e.g. `"C:\path\to\ndk\**"`
- If you extracted a downloaded zip of the template, for each `.ps1` file, go to the file's Properties and check "Unblock" to avoid a confirmation prompty every time you run a script.
- Run `qpm restore`. This will download the dependencies defined in `qpm.json`. When you add or change a dependency, rerun the command. See [the qpm repository](https://github.com/sc2ad/QuestPackageManager) for more information on using qpm.
- To run an initial test build, run `build.ps1`.
  - *Note*: The template includes references to a specific version of `beatsaber_hook` which qpm will likely download a newer version of. This initial build will fail and you'll need to change those versioned references, e.g. find and replace `beatsaber_hook_0_8_2` with `beatsaber_hook_0_8_4` (assuming `0_8_4` is the newest downloaded version)
- Once you get the build succeeding, congratulations! You've compiled a Beat Saber mod. 

**Tips and tricks**:
- Commands can be run from a PowerShell terminal inside VSCode (Terminal > New Terminal or Ctrl+Shift+\`)

---

## Build scripts

- `copy.ps1` will build the mod and copy it directly to your Quest's mods directory (`/sdcard/Android/data/com.beatgames.beatsaber/files/mods`)
  - (Re)start BeatSaber: `adb shell am start com.beatgames.beatsaber/com.unity3d.player.UnityPlayerActivity`
  - Realtime logs: `adb logcat QuestHook[#{id}|#{version}]:* *:S` (see [Logging](#logging))
- `build.ps1` will just build the mod (generates `.so` file) and nothing else. Not super useful outside of confirming code validity.
- `buildBMBF.ps1` will build the mod and package it into a `.zip` that can be installed via BMBF. Once you are ready to share the mod with others, this is the thing you distribute.

---

## Hooks

Hooks are the primary way of interfacing with the game. You find a method that the game calls, and run some of your code whenever that method is called. The hooks themselves are written in two parts. First, you create the hook using `MAKE_OFFSET_HOOKLESS`, then you install the hook at load-time using `INSTALL_HOOK_OFFSETLESS`.

`MAKE_OFFSET_HOOKLESS` takes the args: `hook_name, return_type, self_reference, ...args`, where `hook_name` is whatever you want it to be, `return_type` is the actual type that the original function returns, `self_reference` points to the original object being hooked, and `...args` is all of the arguments passed to the original method.

Hooks replace the original function call, so you generally need to call the original function at some point in your hook. In `void` functions, you'll usually call this at the start:
```c++
MAKE_OFFSET_HOOKLESS(MyHook, void, Il2CppObject* self, SomeType arg1, SomeType arg2) {
  MyHook(self, arg1, arg2);
  /* your code here */
}
```

In functions that return a value, you'll want to make sure to return the original value at the end:
```c++
MAKE_OFFSET_HOOKLESS(MyHook, int, Il2CppObject* self, SomeType arg1, SomeType arg2) {
  // either option A: retrieve the value and return it later
  int original_value = MyHook(self, arg1, arg2);
  /* your code here */
  return original_value;

  // or option B: just return the original value directly
  /* your code here */
  return MyHook(self, arg1, arg2);
}
```

`INSTALL_HOOK_OFFSETLESS` is where you connect your hook code to the original method, using `il2cpp`. For this, you'll need to know the call path to the method, and the number of arguments it takes. If you want to hook `SomeClass::SomeMethod` which takes two args, then it'd look like this:
```c++
INSTALL_HOOK_OFFSETLESS(MyHook, il2cpp_utils::FindMethodUnsafe("", "SomeClass", "SomeMethod", 2))
```

As an example to put these together, let's say you want to a hook a method in the `Foo` class called `SomeMethod` that returns a `float` and takes one `char*` argument:
```c++
MAKE_OFFSET_HOOKLESS(MyHook, float, Il2CppObject* self, char* some_arg) {
  /* do something */
  return MyHook(self, some_arg);
}
extern "C" void load() {
  INSTALL_HOOK_OFFSETLESS(MyHook, il2cpp_utils::FindMethodUnsafe("", "Foo", "SomeMethod", 1));
}
```

### Finding methods to hook

The best way to search through game methods is to use [il2CppDumper](https://github.com/Perfare/Il2CppDumper) and [dnSpy](https://github.com/dnSpy/dnSpy) to look through the game's code:

- Get the Beat Saber APK: From SideQuest, go to "Currently Installed Apps", click the cog icon next to Beat Saber, and then click "Backup APK file".
- Extract from APK: Use an archive tool such as [7zip](https://www.7-zip.org/) to extract `lib/arm64-v8a/libil2cpp.so` and `assets/bin/Data/Managed/Metadata/global-metadata.dat` from the APK.
- Extract DLLs: Run [il2CppDumper](https://github.com/Perfare/Il2CppDumper) and select the two files from the previous step.
- Browse: Use [dnSpy](https://github.com/dnSpy/dnSpy) to open the extracted `DummyDll/main.dll`. From here, you can search the methods in the extracted code. These are the methods you can hook. *TODO: dnSpy screenshot showing a method along with the corresponding code to hook that method.*

One alternative to dumping the code yourself is to search through what's available in `codegen` (which is generated through similar means):

- Add `codegen` as a dependency in `qpm.json`, then run `qpm restore` to download it. *TODO: specifics*
- Use your IDE to search through codegen headers to find hookable methods. *TODO: screenshot of VSCode search finding something*


### Simple hook walkthrough: Modifying a menu

In this walkthrough, we'll modify the main menu and change the text on one of its options.

*TODO: write the thing*


### Simple hook walkthrough: Modifying gameplay

In this walkthrough, we'll modify gameplay by decreasing note jump speed.

*TODO: write the thing*

---

## Going further

With your environment setup and a basic understanding of hooks, you're well on your way to proficiency writing mods. Spend some time exploring the game's methods and where they're called. If you get stuck and can't Google your way out of it, try asking in #quest-mod-dev on the BSMG Discord.


### Logging

Make liberal use of logs to debug when calls are happening and why crashes are occurring. Logs can be viewed in realtime using `adb logcat`. To view logs only for your mod, filter for `QuestHook[$mod_id|v$mod_version]:*`. To view crash logs, `AndroidRuntime:E`.

So, to view logs for your mod, and any crashes (that you probably caused! 😂):

```adb logcat QuestHook[my_mod|v0.1.0]:* AndroidRuntime:E *:S```

You can also pipe logcat to a file to generate dumps which may be helpful in having someone else help you.


### Sharing and distribution

When you're ready to share your work, package it into an installable zip file with `buildBMBF.ps1` and share it with the world! For now, #quest-mods on the BSMG Discord is the primary source of releases.

---

## Links

### Tools
- [qpm (Quest Package Manager)](https://github.com/sc2ad/QuestPackageManager)
- [il2CppDumper](https://github.com/Perfare/Il2CppDumper)
- [dnSpy](https://github.com/dnSpy/dnSpy)
- [7zip](https://www.7-zip.org/)

### Example repositories
- [github search for hooks](https://github.com/search?q=MAKE_HOOK_OFFSETLESS&type=code)
