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
- [Writing hooks](#writing-hooks)
  - [Finding methods to hook](#finding-methods-to-hook)
  - [Simple hook walkthrough](#simple-hook-walkthrough)
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

- (TODO: flesh this out) Create template from zip, create project from template.
- Add your NDK path to `includePaths` in `.vscode/c_cpp_properties.json`, e.g. `"C:\path\to\ndk\**"`
- If you extracted a downloaded zip of the template, for each `.ps1` file, go to the file's Properties and check "Unblock" to avoid a confirmation prompty every time you run a script.
- Run `qpm restore`. This will download the dependencies defined in `qpm.json`. When you add or change a dependency, rerun the command. See [the qpm repository](TODO: link) for more information on using qpm.
- To run an initial test build, run `build.ps1`.
  - *Note*: The template includes references to a specific version of `beatsaber_hook` which qpm will likely download a newer version of. This initial build will fail and you'll need to change those versioned references, e.g. find and replace `beatsaber_hook_0_8_2` with `beatsaber_hook_0_8_4` (assuming `0_8_4` is the newest downloaded version)
- Once you get the build succeeding, congratulations! You've compiled a Beat Saber mod. 

**Tips and tricks**:
- Commands can be run from a PowerShell terminal inside VSCode (Terminal > New Terminal or Ctrl+Shift+\`)

---

## Build scripts

- `copy.ps1` will build the mod and copy it directly to your Quest's mods directory (`/sdcard/Android/data/com.beatgames.beatsaber/files/mods`)
  - (Re)start BeatSaber: `adb shell am start com.beatgames.beatsaber/com.unity3d.player.UnityPlayerActivity`
  - Realtime logs: `adb logcat QuestHook[#{id}|#{version}]:* *:S`
- `build.ps1` will just build the mod (generates `.so` file) and nothing else. Not super useful outside of confirming code validity.
- `buildBMBF.ps1` will build the mod and package it into a `.zip` that can be installed via BMBF. Once you are ready to share the mod with others, this is the thing you distribute.

---

## Writing hooks

*None of this is fleshed out*

Hooks are the primary way of interfacing with the game. You find a method that the game calls, and run some of your code whenever that method is called.

The hooks themselves are written in two parts. First, you create the hook using `MAKE_OFFSET_HOOKLESS`, then you install the hook at load-time using `INSTALL_HOOK_OFFSETLESS`.

```
MAKE_OFFSET_HOOKLESS(
  SomeNameForThisHook, return_type, Foo* self, ...args
) {
  /* do something */
  return SomeNameForThisHook(self, ...args);
}
```
```
extern "C" void load() {
  INSTALL_HOOK_OFFSETLESS(SomeNameForThisHook, il2cpp_utils::FindMethodUnsafe("", "Foo", "SomeMethod", number_of_args));
}
```

### Finding methods to hook

- save apk using sidequest
- use 7zip to extract lib\arm64-v8a\libil2cpp.so and assets\bin\Data\Managed\Metadata\global-metadata.dat
- extract dlls with il2cppdumper
- use dnspy to open dummydll\main.dll
- sloughing through codegen headers as an alternative


### Simple hook walkthrough

maybe multiple simple walkthroughs?
- modifying menu text somewhere
- making something happen when you hit a note


---

## Links

### Tools
- [qpm (Quest Package Manager)](https://github.com/sc2ad/QuestPackageManager)
- il2CppDumper
- dnSpy

### Example repositories
- [github search for hooks](https://github.com/search?q=MAKE_HOOK_OFFSETLESS&type=code)
