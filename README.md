# How to use this

_Updated 11/30/2021 to Dart 2.15.0-268.8.beta_

You can clone this repo and pull out the dart directory which contains all the headers and libraries
you'll need (see Other Notes about some gotchas with these libs).

DartTest2/DartTest2.cpp does all of the embedding work. Note:

- I tried simplifying this as much as I can but there's still some complexities and code formatting
  differences between my style and the dart team. Sorry.
- You can use `Dart_Invoke` over `Dart_RunLoop` to execute a single Dart function, but doing so does
  not drain the message queue. To do so see [Draining the Message
  Queue](#draining-the-message-queue)
- Debugging should work provided the service isolate starts up correctly.
- Hot reloading works, but requires a you write your own watcher script to trigger it, as VSCode
  doesn't implement it for anything other than Flutter projects. see [this
  issue](https://github.com/Dart-Code/Dart-Code/issues/2708) for more information.
    - The [Hotreloader](https://pub.dev/packages/hotreloader) pub package implements hot reloading
      for the current process, but the code can be ported for connecting to an embedded instance.
- This does not take into account loading pre-compiled dart libraries.

Other Notes -

- This is taken from main.cc in runtime/bin which is complicated because it supports all the various
  ways of booting the dart runtime in AOT, and other modes. I would like to see how to accomplish
  that moving forward
- The startup time is high, mostly because of the overhead of booting the kernel and service
  isolates and compiling the dart.
- The way this is currently written assumes sound null safety. There is a function
  `Dart_DetectNullSafety` that you could use instead of setting the `flags->null_safety` parameter
  directly.
- This can now load a .dill precompiled kernel over a .dart file! Change `loadDill` in main to
  switch between using the .dart file and the .dill

# How I did this

Dart doesn't have anything available that makes embedding easy. The dart.lib and header files
included in the SDK are for creating extensions, not for embedding, so unfortunately, you'll have to
build it yourself.

## Get The Dart Source

Get the Dart SDK source according to the instructions provided at the Dart home page:
https://github.com/dart-lang/sdk/wiki/Building

I most recently compiled this with Visual Studio Community 2019, but 2017 is the only "supported"
version You can override the executable for building this by setting the following environment
variables

```
set GYP_MSVS_VERSION=2017
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
set GYP_MSVS_OVERRIDE_PATH="C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\"
```

Make sure depot_tools is also on your path **and** if you have Python 3 installed on your system
make sure depot_tools comes first in your path. Otherwise portions of the build that require Python
2 will fail.

## Build the SDK

Just in case, let's make sure everything builds properly.

As of this writing, this just involves doing `python tools/build.py --mode=release create_sdk`

## Modify some GN files

Dart needs a lot of extra stuff on top of the VM to get a lot of the embedding story working.
Instead of trying to figure out what was necessary and not, I basically created a library that is
all of dart.exe minus "main.cc" and called it "dart_lib". To do this:

- See modifications below
- Regenerate your ninja files for Dart (buildtools\win\gn.exe gen out\DebugX64)
- Build the library
  - Move to out\DebugX64
  - ninja libdart
- The new library will be in out\DebugX64\obj\runtime\bin
- I copied over a bunch of header files into the dart directory locally. You could just reference
  them directly if you had the dart directory in your include path. You can look in theh repo and
  see what I needed to copy other than the dart_api headers

I made other changes to GN files to suit my development style. For example, I no longer use the
statically linked C runtime when I can avoid it, but dart does. If you are building this for
yourself, you may need to change these settings to suit your needs.

### Current .gn modifications

For simplicity, here are the current full modifications I made to the dart-sdk gn files for the libs
included in this repo. These modifications are current as of the version at the top of the file.

In _runtime/bin/BUILD.gn_ add the following:

```
static_library("libdart") {
  use_product_mode = dart_runtime_mode == "release"

  configs += [
    "..:dart_arch_config",
    "..:dart_config",
    "..:dart_os_config",
  ]
  if (use_product_mode) {
    configs += [ "..:dart_product_config" ]
  } else {
    configs += [ "..:dart_maybe_product_config" ]
  }

  deps = [
    ":crashpad",
    "//third_party/boringssl",
    "//third_party/zlib",
    # extra_deps in dart_executable("dart") call:
    ":dart_snapshot_cc",
    "..:libdart_jit",
    "../platform:libdart_platform_jit",
  ]
  if (use_product_mode) {
    deps += [ ":standalone_dart_io_product" ]
  } else {
    deps += [ ":standalone_dart_io" ]
  }
  if (dart_runtime_mode != "release") {
    deps += [ "../observatory:standalone_observatory_archive" ]
  }
  if (!exclude_kernel_service) {
    deps += [ ":dart_kernel_platform_cc" ]
  }
  if (dart_use_tcmalloc) {
    deps += [ "//third_party/tcmalloc" ]
  }

  defines = []
  if (exclude_kernel_service) {
    defines += [ "EXCLUDE_CFE_AND_KERNEL_PLATFORM" ]
  }

  complete_static_lib = true

  include_dirs = [
    "..",
    "//third_party",
  ]

  sources = [
    "dart_embedder_api_impl.cc",
    "error_exit.cc",
    "error_exit.h",
    "main_options.cc",
    "main_options.h",
    "options.cc",
    "options.h",
    "snapshot_utils.cc",
    "snapshot_utils.h",
    "vmservice_impl.cc",
    "vmservice_impl.h",
    # extra_sources in dart_executable("dart") call:
    "builtin.cc",
    "dartdev_isolate.cc",
    "dartdev_isolate.h",
    "dfe.cc",
    "dfe.h",
    "gzip.cc",
    "gzip.h",
    "loader.cc",
    "loader.h",
    "main.cc",
  ]

  if (dart_runtime_mode == "release") {
    sources += [ "observatory_assets_empty.cc" ]
  }

  if (is_win) {
    ldflags = [ "/EXPORT:Dart_True" ]
  } else {
    # Adds all symbols to the dynamic symbol table, not just used ones.
    # This is needed to make native extensions work. It is also needed to get
    # symbols in VM-generated backtraces and profiles.
    ldflags = [ "-rdynamic" ]
  }

  if (is_win) {
    libs = [
      "iphlpapi.lib",
      "psapi.lib",
      "ws2_32.lib",
      "Rpcrt4.lib",
      "shlwapi.lib",
      "winmm.lib",
    ]
  }
}
```

In the main _BUILD.gn_ file, add:

```
group("libdart") {
  deps = [ "runtime/bin:libdart" ]
}
```

(This isn't really necessary, but it makes it easier to run the build).

In _build/config/compiler/BUILD.gn_ change the following (around line 424):

```diff
- # Static CRT.
+ # Dynamic CRT.
  if (is_win) {
    if (is_debug) {
-     cflags += [ "/MTd" ]
+     cflags += [ "/MDd" ]
    } else {
-     cflags += [ "/MT" ]
+     cflags += [ "/MD" ]
    }
    defines += [
      "__STD_C",
      "_CRT_RAND_S",
      "_CRT_SECURE_NO_DEPRECATE",
+     "_ITERATOR_DEBUG_LEVEL=0",
      "_HAS_EXCEPTIONS=0",
      "_SCL_SECURE_NO_DEPRECATE",
    ]
```

### Build the library

Build the static library by running:

```sh
python tools/build.py --no-goma -m release libdart
```

Copy the resulting `xcodebuild/ReleaseX64/obj/runtime/bin/libdart.a` into this
repo at `dart/lib/mac-release-x64/libdart.a`.

**TODO: Update instructions for other platforms.**

# Draining the Message Queue

If you are using Dart_Invoke over Dart_RunLoop, this doesn't give dart any time to drain its message
queue or perform async operations. To get this to work, you need to invoke a private method in the
`dart:isolate` library for now. Here's the code

```cpp
Dart_Handle libraryName = Dart_NewStringFromCString("dart:isolate");
Dart_Handle isolateLib = Dart_LookupLibrary(libraryName);
if (!Dart_IsError(isolateLib))
{
    Dart_Handle invokeName = Dart_NewStringFromCString("_runPendingImmediateCallback");
    Dart_Handle result = Dart_Invoke(isolateLib, invokeName, 0, nullptr);
    if (Dart_IsError(result))
    {
        // Handle error when drainging the microtask queue
    }
    result = Dart_HandleMessage();
    if (Dart_IsError(result))
    {
        // Handle error when drainging the microtask queue
    }
}
```

# Like this?

Follow me [(@fuzzybinary)](http://twitter.com/fuzzybinary) on Twitter and let me know. I'd love to
hear from you!
