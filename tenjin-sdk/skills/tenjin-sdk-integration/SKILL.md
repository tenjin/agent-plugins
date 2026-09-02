---
name: tenjin-sdk-integration
description: Use when integrating the Tenjin SDK into a mobile or game app — iOS, Android, Flutter, Ionic, React Native, or Unity. Detects the project's platform and routes to the correct bundled integration guide covering installation, initialization, event and purchase tracking, and attribution.
---

<!-- Generated from upstream guides/llm-guide.md by tenjin/mcp-server tooling. Do not edit by hand. -->

> **Bundled locally:** the references linked below ship inside this plugin. Read the given file paths directly — do not fetch them over the network.

# Tenjin SDK Integration Guide

> **For AI assistants:** This is the entry point for Tenjin SDK integration. Detect the platform first, then read the appropriate guide.

## Step 1: Detect Platform

Check the user's project for these indicators:

| Platform | Look for |
|----------|----------|
| iOS | `*.xcodeproj`, `*.xcworkspace`, `Podfile`, `Package.swift`, `AppDelegate.swift`, `Info.plist` (without cross-platform markers) |
| Android | `build.gradle`, `AndroidManifest.xml`, `MainActivity.kt`, `*.java` in android structure (without cross-platform markers) |
| Flutter | `pubspec.yaml`, `lib/main.dart`, `android/` + `ios/` folders, `.dart` files |
| Ionic | `ionic.config.json`, `capacitor.config.ts`, `package.json` with `@ionic` or `@capacitor`, `angular.json` or `ionic` in dependencies |
| Unity | `Assets/`, `ProjectSettings/`, `*.unity`, `*.cs` scripts |
| React Native | `package.json` with `react-native`, `App.js` or `App.tsx`, `android/` + `ios/` folders |

If you can't detect the platform, ask:

> "Which platform are you building for? (iOS, Android, Flutter, Ionic, Unity, React Native)"

## Step 2: Read Platform Guide

Once you know the platform, read the corresponding guide:

| Platform | Guide file |
|----------|-----------|
| iOS | [references/ios.md](references/ios.md) |
| Android | [references/android.md](references/android.md) |
| Flutter | [references/flutter.md](references/flutter.md) |
| Ionic | [references/ionic.md](references/ionic.md) |
| Unity | [references/unity.md](references/unity.md) |
| React Native | [references/react-native.md](references/react-native.md) |

## Step 3: Follow the Guide

The platform-specific guide contains:
- Installation instructions
- Code examples
- Integration checklist
- Common mistakes

Follow it step by step.
