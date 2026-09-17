# expo-modules-jsi build-xcframework.sh — Xcode 27 "exit code 0 but produced no further output" repro

Minimal reproduction for https://github.com/expo/expo/issues/50279.

This is a completely blank `npx create-expo-app` project (`expo`, `expo-router`, and their
transitive deps only — no app code touched) with `expo`/`react-native` bumped to the SDK 58
preview / RC versions. The bug is inside `expo-modules-jsi`'s own build script, not app code,
so it reproduces the same way on any project at these versions.

## Environment this was reproduced on

- macOS 27.0 (build 26A428)
- Xcode 27.0 (build 27A266a)
- `expo`: `^58.0.0-preview.3`
- `react-native`: `0.88.0-rc.0`
- `expo-modules-jsi`: `58.0.2`
- Node v26.7.0 / npm 11.19.0

## Steps

```bash
npm install
npx expo prebuild -p ios --non-interactive
cd ios && pod install && cd ..

# Reproduce directly (skips the slower full `eas build`/Xcode archive path):
PLATFORM_NAME=iphoneos PODS_ROOT="$PWD/ios/Pods" \
  node_modules/expo-modules-jsi/apple/scripts/build-xcframework.sh --clean
```

## Result

The script's nested `xcodebuild build -scheme ExpoModulesJSI ...` prints:

```
.../ios/Pods/Headers/Public/React-jsi/jsi/jsi.h:2267:1: warning: umbrella header for module 'jsi' does not include header 'jsilib.h'
2265 |
2266 | #include <jsi/jsi-inl.h>
2267 |
     | `- warning: umbrella header for module 'jsi' does not include header 'jsilib.h'
error: the following command failed with exit code 0 but produced no further output
SwiftCompile normal arm64 (in target 'ExpoModulesJSI' from project 'ExpoModulesJSI')
```

immediately followed by more (real) Swift compiler warnings, and then the script still finishes
with:

```
[ExpoModulesJSI] Built xcframework successfully in 10s (...)
```

So the script's own exit code is `0` and the xcframework is genuinely produced — but this
`error:`-tagged line, when this same script phase runs inside a real Xcode archive (e.g. via
`eas build --local`, `fastlane`/`gym`), gets picked up by Xcode's new build system as a real
build failure even though nothing actually failed. See the parent issue for the full `eas build`
log and `.xcresult` excerpt showing that downstream failure.

## Workaround

Removing `-quiet` from the nested `xcodebuild build` invocation in
`node_modules/expo-modules-jsi/apple/scripts/build-xcframework.sh` makes the phantom `error:`
line disappear on repeated clean (`--clean`) rebuilds.
