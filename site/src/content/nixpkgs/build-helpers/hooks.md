---
title: Hooks reference
---

# Autoconf

The `autoreconfHook` derivation adds `autoreconfPhase`, which runs autoreconf, libtoolize and automake, essentially preparing the configure script in autotools-based builds. Most autotools-based packages come with the configure script pre-generated, but this hook is necessary for a few packages and when you need to patch the package’s configure scripts.

# Automake

Adds the `share/aclocal` subdirectory of each build input to the `ACLOCAL_PATH` environment variable.

# autoPatchcilHook

This is a special setup hook which helps in packaging .NET assemblies/programs in that it automatically tries to find missing shared library dependencies of .NET assemblies based on the given `buildInputs` and `nativeBuildInputs`.

As the hook needs information for the host where the package will be run on, there's a required environment variable called `autoPatchcilRuntimeId` which should be filled in with the RID (Runtime Identifier) of the machine where the output will be run on. If you're using `buildDotnetModule`, it will fall back to `dotnetRuntimeIds` (which is set to `lib.singleton (if runtimeId != null then runtimeId else systemToDotnetRid stdenvNoCC.hostPlatform.system)`) for you if not provided.

In certain situations you may want to run the main command (`autoPatchcil`) of the setup hook on a file or a set of directories instead of unconditionally patching all outputs. This can be done by setting the `dontAutoPatchcil` environment variable to a non-empty value.

By default, `autoPatchcil` will fail as soon as any .NET assembly requires a dependency which cannot be resolved via the given build inputs. In some situations you might prefer to just leave missing dependencies unpatched and continue to patch the rest. This can be achieved by setting the `autoPatchcilIgnoreMissingDeps` environment variable to a non-empty value. `autoPatchcilIgnoreMissingDeps` can be set to a list like `autoPatchcilIgnoreMissingDeps = [ "libcuda.so.1" "libcudart.so.1" ];` or to `[ "*" ]` to ignore all missing dependencies.

The `autoPatchcil` command requires the `--rid` command line flag, informing the RID (Runtime Identifier) it should assume the assemblies will be executed on, and also recognizes a `--no-recurse` command line flag, which prevents it from recursing into subdirectories.

::: {.note}
Since, unlike most native binaries, .NET assemblies are compiled once to run on any platform, many assemblies may have PInvoke stubs for libraries that might not be available on the platform that the package will effectively run on. A few examples are assemblies that call native Windows APIs through PInvoke targeting `kernel32`, `gdi32`, `user32`, `shell32` or `ntdll`.

`autoPatchcil` does its best to ignore dependencies from other platforms by checking the requested file extensions, however not all PInvoke stubs provide an extension so in those cases it will be necessary to list those in `autoPatchcilIgnoreMissingDeps` manually.
:::

# autoPatchelfHook

This is a special setup hook which helps in packaging proprietary software in that it automatically tries to find missing shared library dependencies of ELF files based on the given `buildInputs` and `nativeBuildInputs`.

You can also specify a `runtimeDependencies` variable which lists dependencies to be unconditionally added to the rpath of all executables. This is useful for programs that use `dlopen` to load libraries at runtime.

In certain situations you may want to run the main command (`autoPatchelf`) of the setup hook on a file or a set of directories instead of unconditionally patching all outputs. This can be done by setting the `dontAutoPatchelf` environment variable to a non-empty value.

By default `autoPatchelf` will fail as soon as any ELF file requires a dependency which cannot be resolved via the given build inputs. In some situations you might prefer to just leave missing dependencies unpatched and continue to patch the rest. This can be achieved by setting the `autoPatchelfIgnoreMissingDeps` environment variable to a non-empty value. `autoPatchelfIgnoreMissingDeps` can be set to a list like `autoPatchelfIgnoreMissingDeps = [ "libcuda.so.1" "libcudart.so.1" ];` or to `[ "*" ]` to ignore all missing dependencies.

The `autoPatchelf` command also recognizes a `--no-recurse` command line flag, which prevents it from recursing into subdirectories.

# aws-c-common

This hook exposes its own [CMake](#cmake) modules by setting [`CMAKE_MODULE_PATH`](https://cmake.org/cmake/help/latest/variable/CMAKE_MODULE_PATH.html) through [the `cmakeFlags` variable](#cmake-flags)
to the nonstandard `$out/lib/cmake` directory, as a workaround for [an upstream bug](https://github.com/awslabs/aws-c-common/issues/844).

# bmake

[bmake](https://www.crufty.net/help/sjg/bmake.html) is the portable variant of
NetBSD `make` utility.

In Nixpkgs, `bmake` comes with a hook that overrides the default build, check,
install and dist phases.

# breakpointHook

This hook makes a build pause instead of stopping when a failure occurs. It prevents Nix from cleaning up the build environment immediately and allows the user to attach to the build environment. Upon a build error, it will print instructions that can be used to enter the environment for debugging. The `breakpointHook` is only available on Linux. To use it, add `breakpointHook` to `nativeBuildInputs` in the package to be inspected.

```nix
{ nativeBuildInputs = [ breakpointHook ]; }
```

When a build failure occurs, an instruction will be printed showing how to attach to the build sandbox.

::: {.note}
Caution with remote builds

For remote builds, the printed instructions need to be run on the remote machine, as the build sandbox is only accessible on the machine running the builds. Remote builds can be turned off by setting `--option builders ''` for `nix-build` or `--builders ''` for `nix build`. :::
:::

# CERNLIB

This hook sets the `CERN`, `CERN_LEVEL`, and `CERN_ROOT` environment variables. They are part of [CERNLIB's build system](https://cernlib.web.cern.ch/install/install.html) and are needed for some programs to compile correctly.

# cmake

Overrides the default configure phase to run the CMake command.

By default, we use the Make generator of CMake.
But when Ninja is also available as a `nativeBuildInput`, this setup hook will detect that and use the ninja generator.

Dependencies are added automatically to `CMAKE_PREFIX_PATH` so that packages are correctly detected by CMake.
Some additional flags are passed in to give similar behavior to configure-based packages.

By default, parallel building is enabled as CMake supports parallel building almost everywhere.

You can disable this hook’s behavior by setting `configurePhase` to a custom value, or by setting `dontUseCmakeConfigure`.

## Variables controlling CMake

### CMake Exclusive Variables

#### `cmakeFlags`

Controls the flags passed to `cmake setup` during configure phase.

#### `cmakeBuildDir`

Directory where CMake will put intermediate files.

Setting this can be useful for debugging multiple CMake builds while in the same source directory, for example, when building for different platforms.
Different values for each build will prevent build artifacts from interfering with each other.
This setting has no tangible effect when running the build in a sandboxed derivation.

The default value is `build`.

#### `cmakeBuildType`

Build type of cmake output.

Internally populates the `CMAKE_BUILD_TYPE` cmake flag.

The default value is `Release`.

#### `dontUseCmakeConfigure`

When set to true, don't use the predefined `cmakeConfigurePhase`.

## Controlling CTest invocation

By default tests are run by make in [`checkPhase`](#ssec-check-phase) or by [ninja](#ninja) if `ninja` is
available in `nativeBuildInputs`. Makefile and Ninja generators produce the `test` target, which invokes `ctest` under the hood.
This makes passing additional arguments to `ctest` difficult, so it's possible to invoke it directly in `checkPhase`
by adding `ctestCheckHook` to `nativeCheckInputs`.

### CTest Variables

#### `disabledTests`

Allows to disable running a list of tests. Note that regular expressions are not supported by `disabledTests`, but
it can be combined with `--exclude-regex` option.

#### `ctestFlags`

Additional options passed to `ctest` together with `checkFlags`.

# desktop-file-utils

This setup hook removes the MIME cache (located at `$out/share/applications/mimeinfo.cache`) in the `preFixupPhase`.

This hook is necessary because `mimeinfo.cache` can be created when a package uses `desktop-file-utils`, resulting in collisions if multiple packages are installed that contain this file (as in [#48295](https://github.com/NixOS/nixpkgs/issues/48295)).

# gdk-pixbuf

Exports `GDK_PIXBUF_MODULE_FILE` environment variable to the builder. Add librsvg package to `buildInputs` to get svg support. See also the [setup hook description in GNOME platform docs](#ssec-gnome-hooks-gdk-pixbuf).

# GHC

Creates a temporary package database and registers every Haskell build input in it (TODO: how?).

# GNOME platform

Hooks related to GNOME platform and related libraries like GLib, GTK and GStreamer are described in [](#sec-language-gnome).

# `haredo`

This hook uses [the `haredo` command runner](https://sr.ht/~autumnull/haredo/) to build, check, and install the package. It overrides `buildPhase`, `checkPhase`, and `installPhase` by default.

The hook builds its targets in parallel if [`enableParallelBuilding`](#var-stdenv-enableParallelBuilding) is set to `true`.

## `buildPhase`

This phase attempts to build the default target.

[]{#haredo-hook-haredoBuildTargets} Targets can be explicitly set by adding a string to the `haredoBuildTargets` list.

[]{#haredo-hook-dontUseHaredoBuild} This behavior can be disabled by setting `dontUseHaredoBuild` to `true`.

## `checkPhase`

This phase searches for the `check.do` or `test.do` targets, running them if they exist.

[]{#haredo-hook-haredoCheckTargets} Targets can be explicitly set by adding a string to the `haredoCheckTargets` list.

[]{#haredo-hook-dontUseHaredoCheck} This behavior can be disabled by setting `dontUseHaredoCheck` to `true`.

## `installPhase`

This phase attempts to build the `install.do` target, if it exists.

[]{#haredo-hook-haredoInstallTargets} Targets can be explicitly set by adding a string to the `haredoInstallTargets` list.

[]{#haredo-hook-dontUseHaredoInstall} This behavior can be disabled by setting `dontUseHaredoInstall` to `true`.

# Hooks reference

Nixpkgs has several hook packages that augment the stdenv phases.

The stdenv built-in hooks are documented in [](#ssec-setup-hooks).

```
sections
autoconf.section.md
automake.section.md
autopatchcil.section.md
autopatchelf.section.md
aws-c-common.section.md
bmake.section.md
breakpoint.section.md
cernlib.section.md
cmake.section.md
desktop-file-utils.section.md
gdk-pixbuf.section.md
ghc.section.md
gnome.section.md
haredo.section.md
installShellFiles.section.md
installFonts.section.md
julec.section.md
just.section.md
libglycin.section.md
libiconv.section.md
libxml2.section.md
meson.section.md
mpi-check-hook.section.md
ninja.section.md
nodejs-install-executables.section.md
nodejs-install-manuals.section.md
npm-build-hook.section.md
npm-config-hook.section.md
npm-install-hook.section.md
openmp-check-hook.section.md
patch-rc-path-hooks.section.md
perl.section.md
pkg-config.section.md
pnpm.section.md
postgresql-test-hook.section.md
premake.section.md
python.section.md
scons.section.md
tauri.section.md
tetex-tex-live.section.md
udevCheckHook.section.md
unzip.section.md
validatePkgConfig.section.md
versionCheckHook.section.md
waf.section.md
writable-tmpdir-as-home-hook.section.md
zig.section.md
xcbuild.section.md
xfce4-dev-tools.section.md
```

# `installFonts`

This hook installs common font formats to the proper location. In its default state, the hook automatically handles ttf, ttc, otf, bdf, and psf. Given a `webfont` output, woff and woff2 formats will be installed under this output.

The automatic behavior of the hook can be disabled by setting the `dontInstallFonts` variable to true.

Additionally, it exposes the `installFont` function that can be used from your `postInstall`
hook, to install additional formats:

## `installFont`

The `installFont` function takes two arguments, a file extension to move (_without_ a preceding dot), and the install location.

### Example Usage

```nix
{
  nativeBuildInputs = [ installFonts ];

  postInstall = ''
    installFont svg $out/share/fonts/svg
  '';
}
```

# `installShellFiles`

This hook adds helpers that install artifacts like executable files, manpages
and shell completions.

It exposes the following functions that can be used from your `postInstall`
hook:

## `installBin`

The `installBin` function takes one or more paths to files to install as
executable files.

This function will place them into [`outputBin`](#outputbin).

### Example Usage

```nix
{
  nativeBuildInputs = [ installShellFiles ];

  # Sometimes the file has an undesirable name. It should be renamed before
  # being installed via installBin
  postInstall = ''
    mv a.out delmar
    installBin foobar delmar
  '';
}
```

## `installManPage`

The `installManPage` function takes one or more paths to manpages to install.

The manpages must have a section suffix, and may optionally be compressed (with
`.gz` suffix). This function will place them into the correct
`share/man/man<section>/` directory in [`outputMan`](#outputman).

### Example Usage

```nix
{
  nativeBuildInputs = [ installShellFiles ];

  # Sometimes the manpage file has an undesirable name; e.g., it conflicts with
  # another software with an equal name. To install it with a different name,
  # the installed name must be provided before the path to the file.
  #
  # Below install a manpage "foobar.1" from the source file "./foobar.1", and
  # also installs the manpage "fromsea.3" from the source file "./delmar.3".
  postInstall = ''
    installManPage \
        foobar.1 \
        --name fromsea.3 delmar.3
  '';
}
```

The manpage may be the result of a piped input (e.g. `<(cmd)`), in which
case the name must be provided before the pipe with the `--name` flag.

```nix
{
  nativeBuildInputs = [ installShellFiles ];

  postInstall = ''
    installManPage --name foobar.1 <($out/bin/foobar --manpage)
  '';
}
```

If no parsing of arguments is desired, pass `--` to opt-out of all subsequent
arguments.

```nix
{
  nativeBuildInputs = [ installShellFiles ];

  # Installs a manpage from a file called "--name"
  postInstall = ''
    installManPage -- --name
  '';
}
```

## `installShellCompletion`

The `installShellCompletion` function takes one or more paths to shell
completion files.

By default it will autodetect the shell type from the completion file extension,
but you may also specify it by passing one of `--bash`, `--fish`, `--zsh`, or
`--nushell`. These flags apply to all paths listed after them (up until another
shell flag is given). Each path may also have a custom installation name
provided by providing a flag `--name NAME` before the path. If this flag is not
provided, zsh completions will be renamed automatically such that `foobar.zsh`
becomes `_foobar`. A root name may be provided for all paths using the flag
`--cmd NAME`; this synthesizes the appropriate name depending on the shell
(e.g. `--cmd foo` will synthesize the name `foo.bash` for bash and `_foo` for
zsh).

### Example Usage

```nix
{
  nativeBuildInputs = [ installShellFiles ];
  postInstall = ''
    # explicit behavior
    installShellCompletion --bash --name foobar.bash share/completions.bash
    installShellCompletion --fish --name foobar.fish share/completions.fish
    installShellCompletion --nushell --name foobar share/completions.nu
    installShellCompletion --zsh --name _foobar share/completions.zsh
    # implicit behavior
    installShellCompletion share/completions/foobar.{bash,fish,zsh,nu}
  '';
}
```

The path may also be the result of process substitution (e.g. `<(cmd)`), in which
case the shell and name must be provided (see below).

If the destination shell completion file is not actually present or consists of
zero bytes after calling `installShellCompletion` this is treated as a build
failure. In particular, if completion files are not vendored but are generated
by running an executable, this is likely to fail in cross compilation
scenarios. The result will be a zero byte completion file and hence a build
failure. To prevent this, guard the completion generation commands.

### Example Usage

```nix
{
  nativeBuildInputs = [ installShellFiles ];
  postInstall = lib.optionalString (stdenv.buildPlatform.canExecute stdenv.hostPlatform) ''
    # using process substitution
    installShellCompletion --cmd foobar \
      --bash <($out/bin/foobar --bash-completion) \
      --fish <($out/bin/foobar --fish-completion) \
      --nushell <($out/bin/foobar --nushell-completion) \
      --zsh <($out/bin/foobar --zsh-completion)
  '';
}
```

# `juce.projucerHook`

[Projucer](https://juce.com/tutorials/tutorial_new_projucer_project/) is a graphical project management utility and build system for the [JUCE](https://juce.com/) audio programming framework. It is available in nixpkgs under the `juce` package.

The `juce.projucerHook` setup hook overrides the configure and install phases. It is only supported on Linux and requires your project's `.jucer` file to contain a `LinuxMakefile` exporter.

## Example

```nix
{
  juce,
  stdenv,
}:
stdenv.mkDerivation {
  # ...
  nativeBuildInputs = [ juce.projucerHook ];

  jucerFile = "Microbiome.jucer";

  dontUseProjucerInstall = true;
  # ...
}
```

## Variables controlling `juce.projucerHook`

### `dontUseProjucerConfigure`

Disables `projucerConfigurePhase`

### `dontUseProjucerInstall`

Disables `projucerInstallPhase`

# julec.hook

[Jule](https://jule.dev) is an effective programming language designed to build efficient, fast, reliable and safe software while maintaining simplicity.

In Nixpkgs, `jule.hook` overrides the default build, check and install phases.

## Example code snippet

```nix
{
  julec,
  clangStdenv,
}:

clangStdenv.mkDerivation (finalAttrs: {
  # ...

  nativeBuildInputs = [ julec.hook ];

  # Customize filenames if needed
  JULE_SRC_DIR = "./src";
  JULE_OUT_DIR = "./bin";
  JULE_OUT_NAME = "hello-jule";
  JULE_TEST_DIR = "./tests";
  JULE_TEST_OUT_DIR = "./test-bin";
  JULE_TEST_OUT_NAME = "hello-jule-test";

  # ...
})
```

## Variables controlling julec.hook

### `JULE_SRC_DIR`

Specifies the source directory containing `main.jule`.
Default is `./src`.

### `JULE_OUT_DIR`

Specifies the output directory for the compiled binary.
Default is `./bin`.

### `JULE_OUT_NAME`

Specifies the name of the compiled binary.
Default is `output`.

### `JULE_TEST_DIR`

Specifies the directory containing test files.
Default is the value of [`JULE_SRC_DIR`](#julec-hook-variable-jule-src-dir).

### `JULE_TEST_OUT_DIR`

Specifies the output directory for compiled test binaries.
Default is the value of [`JULE_OUT_DIR`](#julec-hook-variable-jule-out-dir).

### `JULE_TEST_OUT_NAME`

Specifies the name of the compiled test binary.
Default is the value of [`JULE_OUT_NAME`](#julec-hook-variable-jule-out-name) with `-test` suffix.

### `dontUseJulecBuild`

When set to true, doesn't use the predefined `julecBuildHook`.
Default is false.

### `dontUseJulecCheck`

When set to true, doesn't use the predefined `julecCheckHook`.
Default is false.

### `dontUseJulecInstall`

When set to true, doesn't use the predefined `julecInstallHook`.
Default is false.

# `just`

This setup hook attempts to use [the `just` command runner](https://just.systems/man/en/) to build, check, and install the package. The hook overrides `buildPhase`, `checkPhase`, and `installPhase` by default.

[]{#just-hook-justFlags} The `justFlags` variable can be set to a list of strings to add additional flags passed to all invocations of `just`.

## `buildPhase`

This phase attempts to invoke `just` with [the default recipe](https://just.systems/man/en/the-default-recipe.html).

[]{#just-hook-dontUseJustBuild} This behavior can be disabled by setting `dontUseJustBuild` to `true`.

## `checkPhase`

This phase attempts to invoke the `just test` recipe, if it is available. This can be overridden by setting `checkTarget` to a string.

[]{#just-hook-dontUseJustCheck} This behavior can be disabled by setting `dontUseJustCheck` to `true`.

## `installPhase`

This phase attempts to invoke the `just install` recipe.

[]{#just-hook-dontUseJustInstall} This behavior can be disabled by setting `dontUseJustInstall` to `true`.

# libglycin

[Glycin](https://gitlab.gnome.org/GNOME/glycin) is a library for sandboxed and extendable image loading.

[]{#libglycin-setup-hook} For most applications using it, individual image formats are loaded through binaries provided by `glycin-loaders`. The paths of these loaders must be injected into the environment, e.g. using [`wrapGAppsHook`](#ssec-gnome-hooks). `libglycin.setupHook` will do that.

[]{#libglycin-patch-vendor-hook} Additionally, for Rust projects `glycin` Rust crate itself requires a patch to become self-contained. `libglycin.patchVendorHook` will do that. This is not needed for projects using the ELF library from `libglycin` package.

## Example code snippet

```nix
{
  lib,
  rustPlatform,
  libglycin,
  glycin-loaders,
  wrapGAppsHook4,
}:

rustPlatform.buildRustPackage {
  # ...

  cargoHash = "...";

  nativeBuildInputs = [
    wrapGAppsHook4
    libglycin.patchVendorHook
  ];

  buildInputs = [
    libglycin.setupHook
    glycin-loaders
  ];

  # ...
}
```

## Variables controlling glycin-loaders

### `glycinCargoDepsPath`

Path to a directory containing the `glycin` crate to patch. Defaults to the crate directory created by `cargoSetupHook`, or `./vendor/`.

### `dontWrapGlycinLoaders`

Disable adding the Glycin loaders path `XDG_DATA_DIRS` with `wrapGAppsHook`.

# libiconv, libintl

A few libraries automatically add to `NIX_LDFLAGS` their library, making their symbols automatically available to the linker. This includes libiconv and libintl (gettext). This is done to provide compatibility between GNU Linux, where libiconv and libintl are bundled in, and other systems where that might not be the case. Sometimes, this behavior is not desired. To disable this behavior, set `dontAddExtraLibs`.

# libxml2

Adds every file named `catalog.xml` found under the `xml/dtd` and `xml/xsl` subdirectories of each build input to the `XML_CATALOG_FILES` environment variable.

# `memcachedTestHook`

This hook starts a Memcached server during `checkPhase`. Example:

```nix
{ stdenv, memcachedTestHook }:
stdenv.mkDerivation {

  # ...

  nativeCheckInputs = [ memcachedTestHook ];
}
```

If you use a custom `checkPhase`, remember to add the `runHook` calls:

```nix
{
  checkPhase = ''
    runHook preCheck

    # ... your tests

    runHook postCheck
  '';
}
```

## Variables

Bash-only variables:

- `memcachedTestPort`: Port to use by Memcached. Defaults to `11211`

Example usage:

```nix
{ stdenv, memcachedTestHook }:
stdenv.mkDerivation {

  # ...

  nativeCheckInputs = [ memcachedTestHook ];

  preCheck = ''
    memcachedTestPort=1234;
  '';
}
```

# Meson

[Meson](https://mesonbuild.com/) is an open source meta build system meant to be
fast and user-friendly.

In Nixpkgs, meson comes with a setup hook that overrides the configure, check,
and install phases.

Being a meta build system, meson needs an accompanying backend. In the context
of Nixpkgs, the typical companion backend is [Ninja](#ninja), that provides a
setup hook registering ninja-based build and install phases.

## Variables controlling Meson

### Meson Exclusive Variables

#### `mesonFlags`

Controls the flags passed to `meson setup` during configure phase.

#### `mesonBuildDir`

Directory where Meson will put intermediate files.

Setting this can be useful for debugging multiple Meson builds while in the same source directory, for example, when building for different platforms.
Different values for each build will prevent build artifacts from interfering with each other.
This setting has no tangible effect when running the build in a sandboxed derivation.

The default value is `build`.

#### `mesonWrapMode`

Which value is passed as
[`-Dwrap_mode=`](https://mesonbuild.com/Builtin-options.html#core-options).
In Nixpkgs, the default value is `nodownload`, so that no subproject will be
downloaded (since network access is already disabled during deployment in
Nixpkgs).

Note: Meson allows pre-population of subprojects that would otherwise be
downloaded.

#### `mesonBuildType`

Which value is passed as
[`--buildtype`](https://mesonbuild.com/Builtin-options.html#core-options) to
`meson setup` during configure phase. In Nixpkgs, the default value is `plain`.

#### `mesonAutoFeatures`

Which value is passed as
[`-Dauto_features=`](https://mesonbuild.com/Builtin-options.html#core-options)
to `meson setup` during configure phase. In Nixpkgs, the default value is
`enabled`, meaning that every feature declared as "auto" by the meson scripts
will be enabled.

#### `mesonCheckFlags`

Controls the flags passed to `meson test` during check phase.

#### `mesonInstallFlags`

Controls the flags passed to `meson install` during install phase.

#### `mesonInstallTags`

A list of installation tags passed to Meson's commandline option
[`--tags`](https://mesonbuild.com/Installing.html#installation-tags) during
install phase.

Note: `mesonInstallTags` should be a list of strings that will be converted to
a comma-separated string that is recognized to `--tags`.
Example: `mesonInstallTags = [ "emulator" "assembler" ];` will be converted to
`--tags emulator,assembler`.

#### `dontUseMesonConfigure`

When set to true, don't use the predefined `mesonConfigurePhase`.

#### `dontUseMesonCheck`

When set to true, don't use the predefined `mesonCheckPhase`.

#### `dontUseMesonInstall`

When set to true, don't use the predefined `mesonInstallPhase`.

### Honored variables

The following variables commonly used by `stdenv.mkDerivation` are honored by
Meson setup hook.

- `prefixKey`
- `enableParallelBuilding`
- `enableParallelChecking`

# mpiCheckPhaseHook

This hook can be used to setup a check phase that
requires running a MPI application. It detects the
present MPI implementation type and exports
the necessary environment variables to use
`mpirun` and `mpiexec` in a Nix sandbox.

Example:

```nix
{ mpiCheckPhaseHook, mpi, ... }:
{
  # ...

  nativeCheckInputs = [
    openssh
    mpiCheckPhaseHook
  ];
}
```

# ninja

Overrides the build, install, and check phase to run ninja instead of make. You can disable this behavior with `dontUseNinjaBuild`, `dontUseNinjaInstall`, and `dontUseNinjaCheck`, respectively. Parallel building is enabled by default in Ninja.

Note that if the [Meson setup hook](#meson) is also active, Ninja's install and check phases will be disabled in favor of Meson's.

# nodejsInstallExecutables

Hook for wrapping Node.js executables.
Primarily created for a multi-language environment.

## Examples

[](#npm-build-hook-example-snippet)

## Variables controlling `nodejsInstallExecutables`

### `nodejsInstallExecutables` Exclusive Variables

#### `makeWrapperArgs`

Flags to pass to the call to [`makeWrapper`](#fun-makeWrapper).
To avoid double-wrapping, this flag can also be accessed in Bash.

```nix
stdenv.mkDerivation (finalAttrs: {
  #...
  dontWrapGApps = true;

  postInstall = ''
    makeWrapperArgs+=("''${gappsWrapperArgs[@]}")
  '';
  #...
})
```

# nodejsInstallManuals

Detects manuals in Node.js packages, and attempts to install them in standard locations.
This detection is done by inspecting the package.json of the project and finding any entries
with type `man`.

There are no ways currently to configure this hook.

## Examples

[](#npm-build-hook-example-snippet)

# npmHooks.npmBuildHook

Hook for building packages that use npm. Can be used in multi-language environments.

## Examples

:::{.example #npm-build-hook-example-snippet}

# Using `npmHooks`

```nix
{
  stdenv,
  fetchFromGitHub,
  fetchNpmDeps,
  npmHooks,
  nodejsInstallExecutables,
  nodejsInstallManuals,
  nodejs,
}:
stdenv.mkDerivation (finalAttrs: {
  pname = "some-npm-project";
  version = "1.0";

  src = fetchFromGitHub {
    owner = "JohnNpm";
    repo = "SomeProject";
    tag = finalAttrs.version;
    hash = "...";
  };

  strictDeps = true;

  nativeBuildInputs = [
    nodejs
    nodejsInstallExecutables
    nodejsInstallManuals
    npmHooks.npmConfigHook
    npmHooks.npmBuildHook
    npmHooks.npmInstallHook
  ];

  npmBuildScript = "build";

  npmBuildFlags = [
    "--prod"
  ];

  npmFlags = [
    "--ignore-scripts"
  ];

  npmDeps = fetchNpmDeps {
    inherit (finalAttrs) src;
    hash = "...";
  };

  makeWrapperArgs = [
    "--set"
    "NODE_ENV"
    "production"
  ];

  meta = {
    description = "npm project";
  };
})
```

:::

## Variables controlling `npmBuildHook`

### `npmBuildHook` Exclusive Variables

#### `npmBuildScript`

Controls the script ran to build the npm package within the `package.json` file.
Required to be set, usually to `build`, but can vary between packages.

#### `npmBuildFlags`

Controls the arguments to the {command}`npm run $npmBuildScript` command.

#### `dontNpmBuild`

Disables `npmBuildHook` when enabled

### Honored Variables

The following variables are honored by the `npmBuildHook`.

- [`npmWorkspace`](#javascript-buildNpmPackage-npmWorkspace)
- [`npmFlags`](#javascript-buildNpmPackage-npmFlags)

# npmHooks.npmConfigHook

Hook for configuring packages that use npm.
Primarily made for a multi-language environment.

## Examples

[](#npm-build-hook-example-snippet)

## Variables controlling `npmConfigHook`

### `npmConfigHook` Exclusive Variables

#### `npmDeps`

Derivation that contains the npm package dependencies.
Usually built with `fetchNpmDeps`.
This attribute is required or the hook will abort the build.

#### `makeCacheWritable`

Whether to make the dependency cache writable prior to installing the dependencies.
Don't set this unless npm tries to write to the cache directory.

#### `npmInstallFlags`

Flags to pass to the {command}`npm ci` call for installing the dependencies to the build environment.
Defaults to `--ignore-scripts`, which cannot be removed.
This does not control anything with the `npmInstallHook`.

#### `npmRebuildFlags`

Flags to pass to the {command}`npm rebuild` command after the dependencies are installed to the environment.

### Honored Variables

The following variables are honored by the `npmConfigHook`.

- [`npmWorkspace`](#javascript-buildNpmPackage-npmWorkspace)
- [`npmFlags`](#javascript-buildNpmPackage-npmFlags)
- `npmRoot`

# npmHooks.npmInstallHook

Hook to install node_modules for npm packages.
Does not create wrappers for executable npm projects
Primarily made for a multi-language environment.

## Examples

[](#npm-build-hook-example-snippet)

## Variables controlling `npmInstallHook`

### `npmInstallHook` Exclusive Variables

#### `dontNpmPrune`

Whether to run {command}`npm prune` on the `node_modules` or not.
Defaults to `true`.

#### `npmInstallFlags`

Flags to pass to the {command}`npm prune` call for the `node_modules` of the package.
Defaults to `--omit=dev --no-save` which cannot be modified.

#### `dontNpmInstall`

Controls whether `npmInstallHook` is enabled or not.
Defaults to `true`, so the hook will run.

### Honored Variables

The following variables are honored by the `npmInstallHook`.

- [`npmWorkspace`](#javascript-buildNpmPackage-npmWorkspace)
- [`npmFlags`](#javascript-buildNpmPackage-npmFlags)

# openmpCheckPhaseHook

This hook can be used to setup a check phase that
requires running a OpenMP application. It mostly
serves to limit `OMP_NUM_THREADS` to avoid overloading
build machines.

This hook will not attempt to override an already existing
definition of `OMP_NUM_THREADS` in the environment.

# `patchRcPath` hooks

These hooks provide shell-specific utilities (with the same name as the hook) to patch shell scripts meant to be sourced by software users.

The typical usage is to patch initialisation or [rc](https://unix.stackexchange.com/questions/3467/what-does-rc-in-bashrc-stand-for) scripts inside `$out/bin` or `$out/etc`.
Such scripts, when being sourced, would insert the binary locations of certain commands into `PATH`, modify other environment variables or run a series of start-up commands.
When shipped from the upstream, they sometimes use commands that might not be available in the environment they are getting sourced in.

The compatible shells for each hook are:

- `patchRcPathBash`: [Bash](https://www.gnu.org/software/bash/), [ksh](http://www.kornshell.org/), [zsh](https://www.zsh.org/) and other shells supporting the Bash-like parameter expansions.
- `patchRcPathCsh`: Csh scripts, such as those targeting [tcsh](https://www.tcsh.org/).
- `patchRcPathFish`: [Fish](https://fishshell.com/) scripts.
- `patchRcPathPosix`: POSIX-conformant shells supporting the limited parameter expansions specified by the POSIX standard. The current implementation uses the parameter expansion `${foo-}` only.

For each supported shell, it modifies the script with a `PATH` prefix that is later removed when the script ends.
It allows nested patching, which guarantees that a patched script may source another patched script.

Syntax to apply the utility to a script:

```sh
patchRcPath<shell> <file> <PATH-prefix>
```

Example usage:

Given a package `foo` containing an init script `this-foo.fish` that depends on `coreutils`, `man` and `which`,
patch the init script for users to source without having the above dependencies in their `PATH`:

```nix
{
  lib,
  stdenv,
  patchRcPathFish,
}:
stdenv.mkDerivation {

  # ...

  nativeBuildInputs = [ patchRcPathFish ];

  postFixup = ''
    patchRcPathFish $out/bin/this-foo.fish ${
      lib.makeBinPath [
        coreutils
        man
        which
      ]
    }
  '';
}
```

::: {.note}
`patchRcPathCsh` and `patchRcPathPosix` implementation depends on `sed` to do the string processing.
The others are in vanilla shell and have no third-party dependencies.
:::

# Perl

Adds the `lib/site_perl` subdirectory of each build input to the `PERL5LIB` environment variable. For instance, if `buildInputs` contains Perl, then the `lib/site_perl` subdirectory of each input is added to the `PERL5LIB` environment variable.

# pkg-config

Adds the `lib/pkgconfig` and `share/pkgconfig` subdirectories of each build input to the `PKG_CONFIG_PATH` environment variable.

# pnpmBuildHook

[pnpm](https://pnpm.io/) is a an NPM-compatible package manager focused on increasing managment speeds, and reducing disk space.

The `pnpmBuildHook` in Nixpkgs overrides the default build phase for building packages that use pnpm.

:::{.example #ex-pnpm-build-hook}

## pnpmBuildHook example code snippet

```nix
{
  lib,
  stdenv,
  fetchFromGitHub,
  fetchPnpmDeps,
  pnpmConfigHook,
  pnpmBuildHook,
  makeBinaryWrapper,
  pnpm_10,
}:
let
  pnpm = pnpm_10;
in
stdenv.mkDerivation (finalAttrs: {
  pname = "coolPackages";
  version = "1.0";

  src = fetchFromGitHub {
    owner = "JaneCool";
    repo = "coolpackage";
    tag = finalAttrs.version;
    hash = lib.fakeHash;
  };

  __structuredAttrs = true;
  strictDeps = true;

  pnpmDeps = fetchPnpmDeps {
    inherit (finalAttrs) pname version src;
    inherit pnpm;
    fetcherversion = 4;
    hash = lib.fakeHash;
  };

  nativeBuildInputs = [
    pnpmConfigHook
    pnpmBuildHook
    makeBinaryWrapper
  ];

  pnpmBuildScript = "build";
  pnpmBuildFlags = [
    "--mode"
    "production"
  ];
  pnpmWorkspaces = [
    "test"
  ];

  installPhase = ''
    runHook preInstall

    mkdir "$out"
    cp -r dist/. "$out"

    runHook postInstall
  '';

  meta = {
    description = "very cool package that does cool things";
    mainProgram = "cool";
  };
})
```

:::

## Variables controlling pnpmBuildHook

### pnpm Exclusive Variables

#### `pnpmBuildScript`

Controls the script ran to build the package, by default the script is `build`.

#### `pnpmFlags`

Controls flags used for all invocations of pnpm across all hooks local to this derivation.

#### `pnpmBuildFlags`

Controls the flags pass only to the pnpm build script invocation.

#### `dontPnpmBuild`

Disables automatically running `pnpmBuildHook`. The build can still be run manually if needed, for example:

```nix
{
  lib,
  rustPlatform,
  pnpmBuildHook,
  pnpmConfigHook,
  fetchPnpmDeps,
  emptyDirectory,
  pnpm_10,
}:
let
  pnpm = pnpm_10;
in
rustPlatform.buildRustPackage (finalAttrs: {
  pname = "super-fast-application";
  version = "1.0";

  src = emptyDirectory;

  cargoHash = lib.fakeHash;

  nativeBuildInputs = [
    pnpmBuildHook
    pnpmConfigHook
  ];

  pnpmDeps = fetchPnpmDeps {
    inherit (finalAttrs) pname version src;
    inherit pnpm;
    fetcherversion = 4;
    hash = lib.fakeHash;
  };

  dontPnpmBuild = true;
  postBuild = ''
    pnpmBuildHook
  '';
})
```

### Honored Variables

The following variables are honored by `pnpmBuildHook`.

- [`pnpmRoot`](#javascript-pnpm-sourceRoot)
- [`pnpmWorkspaces`](#javascript-pnpm-workspaces)

# `postgresqlTestHook`

This hook starts a PostgreSQL server during the `checkPhase`. Example:

```nix
{
  stdenv,
  postgresql,
  postgresqlTestHook,
}:
stdenv.mkDerivation {

  # ...

  nativeCheckInputs = [
    postgresql
    postgresqlTestHook
  ];
}
```

If you use a custom `checkPhase`, remember to add the `runHook` calls:

```nix
checkPhase ''
  runHook preCheck

  # ... your tests

  runHook postCheck
''
```

## Variables

The hook logic will read a number of variables and set them to a default value if unset or empty.

Exported variables:

- `PGDATA`: location of server files.
- `PGHOST`: location of UNIX domain socket directory; the default `host` in a connection string.
- `PGUSER`: user to create / log in with, default: `test_user`.
- `PGDATABASE`: database name, default: `test_db`.

Bash-only variables:

- `postgresqlTestUserOptions`: SQL options to use when creating the `$PGUSER` role, default: `"LOGIN"`. Example: `"LOGIN SUPERUSER"`
- `postgresqlTestSetupSQL`: SQL commands to run as database administrator after startup, default: statements that create `$PGUSER` and `$PGDATABASE`.
- `postgresqlTestSetupCommands`: bash commands to run after database start, defaults to running `$postgresqlTestSetupSQL` as database administrator.
- `postgresqlEnableTCP`: set to `1` to enable TCP listening. Flaky; not recommended.
- `postgresqlStartCommands`: defaults to `pg_ctl start`.
- `postgresqlExtraSettings`: Additional configuration to add to `postgresql.conf`

## Hooks

A number of additional hooks are run in postgresqlTestHook

- `postgresqlTestSetupPost`: run after postgresql has been set up.

## TCP and the Nix sandbox

`postgresqlEnableTCP` relies on network sandboxing, which is not available on macOS and some custom Nix installations, resulting in flaky tests.
For this reason, it is disabled by default.

The preferred solution is to make the test suite use a UNIX domain socket connection. This is the default behavior when no `host` connection parameter is provided.
Some test suites hardcode a value for `host` though, so a patch may be required. If you can upstream the patch, you can make `host` default to the `PGHOST` environment variable when set. Otherwise, you can patch it locally to omit the `host` connection string parameter altogether.

::: {.note}
The error `libpq: failed (could not receive data from server: Connection refused` is generally an indication that the test suite is trying to connect through TCP.
:::

# Premake

This setup hook attempts to configure the package using [the Premake build configuration system](https://premake.github.io/). It overrides the `configurePhase` by default, if none exists.

[]{#premake-hook-premakefile} The Premakefile to use can be specified by setting `premakefile` in the derivation.

[]{#premake-hook-premakeFlagsArray} The flags passed to Premake can be configured by adding strings to the `premakeFlags` list.

# Python

Adds the `python.sitePackages` subdirectory (i.e. `lib/pythonX.Y/site-packages`) of each build input to the `PYTHONPATH` environment variable.

# `redisTestHook`

This hook starts a Redis server during `checkPhase`. Example:

```nix
{
  stdenv,
  redis,
  redisTestHook,
}:
stdenv.mkDerivation {

  # ...

  nativeCheckInputs = [ redisTestHook ];
}
```

If you use a custom `checkPhase`, remember to add the `runHook` calls:

```nix
{
  checkPhase = ''
    runHook preCheck

    # ... your tests

    runHook postCheck
  '';
}
```

## Variables

The hook logic will read the following variables and set them to a default value if unset or empty.

Exported variables:

- `REDIS_SOCKET`: UNIX domain socket path

Bash-only variables:

- `redisTestPort`: Port to use by Redis. Defaults to `6379`

Example usage:

```nix
{
  stdenv,
  redis,
  redisTestHook,
}:
stdenv.mkDerivation {

  # ...

  nativeCheckInputs = [ redisTestHook ];

  preCheck = ''
    redisTestPort=6390;
  '';
}
```

# scons

Overrides the build, install, and check phases. This uses the scons build system as a replacement for make. scons does not provide a configure phase, so everything is managed at build and install time.

# cargo-tauri.hook

[Tauri](https://tauri.app/) is a framework for building smaller, faster, and
more secure desktop applications with a web frontend.

In Nixpkgs, `cargo-tauri.hook` overrides the default build and install phases.

## Example code snippet

```nix
{
  lib,
  stdenv,
  rustPlatform,
  fetchNpmDeps,
  cargo-tauri,
  glib-networking,
  nodejs,
  npmHooks,
  openssl,
  pkg-config,
  webkitgtk_4_1,
  wrapGAppsHook4,
}:

rustPlatform.buildRustPackage (finalAttrs: {
  # ...

  cargoHash = "...";

  # Assuming our app's frontend uses `npm` as a package manager
  npmDeps = fetchNpmDeps {
    name = "${finalAttrs.pname}-${finalAttrs.version}-npm-deps";
    inherit (finalAttrs) src;
    hash = "...";
  };

  nativeBuildInputs = [
    # Pull in our main hook
    cargo-tauri.hook

    # Setup npm
    nodejs
    npmHooks.npmConfigHook

    # Make sure we can find our libraries
    pkg-config
  ]
  ++ lib.optionals stdenv.hostPlatform.isLinux [ wrapGAppsHook4 ];

  buildInputs = lib.optionals stdenv.hostPlatform.isLinux [
    glib-networking # Most Tauri apps need networking
    openssl
    webkitgtk_4_1
  ];

  # Set our Tauri source directory
  cargoRoot = "src-tauri";
  # And make sure we build there too
  buildAndTestSubdir = finalAttrs.cargoRoot;

  # ...
})
```

## Variables controlling cargo-tauri

### Tauri Exclusive Variables

#### `tauriBuildFlags`

Controls the flags passed to `cargo tauri build`.

#### `tauriBundleType`

The [bundle type](https://tauri.app/v1/guides/building/) to build.

#### `dontTauriBuild`

Disables using `tauriBuildHook`.

#### `dontTauriFixup`

Disables the `tauriFixupHook` pre fixup phase.

#### `dontTauriInstall`

Disables using `tauriInstallPostBuildHook` and `tauriInstallHook`.

### Honored Variables

Along with those found in [](#compiling-rust-applications-with-cargo), the
following variables used by `cargoBuildHook` and `cargoInstallHook` are honored
by the cargo-tauri setup hook.

- `buildAndTestSubdir`
- `cargoBuildType`
- `cargoBuildNoDefaultFeatures`
- `cargoBuildFeatures`

# teTeX / TeX Live

Adds the `share/texmf-nix` subdirectory of each build input to the `TEXINPUTS` environment variable.

# udevCheckHook

The `udevCheckHook` derivation adds `udevCheckPhase` to the [`preInstallCheckHooks`](#ssec-installCheck-phase),
which finds all udev rules in all outputs and verifies them using `udevadm verify --resolve-names=never --no-style`.
It should be used in any package that has udev rules outputs to ensure the rules are and remain valid.

The hook runs in `installCheckPhase`, requiring `doInstallCheck` is enabled for the hook to take effect:

```nix
{
  lib,
  stdenv,
  udevCheckHook,
  # ...
}:

stdenv.mkDerivation (finalAttrs: {
  # ...

  nativeInstallCheckInputs = [ udevCheckHook ];
  doInstallCheck = true;

  # ...
})
```

Note that for [`buildPythonPackage`](#buildpythonpackage-function) and [`buildPythonApplication`](#buildpythonapplication-function), `doInstallCheck` is enabled by default.

All outputs are scanned for their `/{etc,lib}/udev/rules.d` paths.
If no rule output is found, the hook is basically a no-op.

The `udevCheckHook` adds a dependency on `systemdMinimal`.
It is internally guarded behind `hostPlatform` supporting udev and `buildPlatform` being able to execute `udevadm`.
The hook does not need explicit platform checks in the places where it is used.

The hook can be disabled using `dontUdevCheck`, which is necessary if you want to run some different task in `installCheckPhase` on a package with broken udev rule outputs.

# unzip

This setup hook will allow you to unzip .zip files specified in `$src`. There are many similar packages like `unrar`, `undmg`, etc.

# validatePkgConfig

The `validatePkgConfig` hook validates all pkg-config (`.pc`) files in a package. This helps catch some common errors in pkg-config files, such as undefined variables.

# versionCheckHook

This hook adds a `versionCheckPhase` to the [`preInstallCheckHooks`](#ssec-installCheck-phase) that runs the main program of the derivation with a `--help` or `--version` argument, and checks that the `${version}` string is found in that output. If this check fails then the whole build will fail. _(A softer option is [`testers.testVersion`](#tester-testVersion).)_

You use it like this:

```nix
{
  lib,
  stdenv,
  versionCheckHook,
  # ...
}:

stdenv.mkDerivation (finalAttrs: {
  # ...

  nativeInstallCheckInputs = [ versionCheckHook ];
  doInstallCheck = true;

  # ...
})
```

Note that for [`buildPythonPackage`](#buildpythonpackage-function) and [`buildPythonApplication`](#buildpythonapplication-function), `doInstallCheck` is enabled by default.

It does so in a clean environment (using `env --ignore-environment`), and it checks for the `${version}` string in both the `stdout` and the `stderr` of the command. It will report to you in the build log the output it received and it will fail the build if it failed to find `${version}`.

The variables that this phase control are:

- `dontVersionCheck`: Disable adding this hook to the [`preInstallCheckHooks`](#ssec-installCheck-phase). Useful if you do want to load the bash functions of the hook, but run them differently.
- `versionCheckProgram`: The full path to the program that should print the `${version}` string. Defaults to using the first non-empty value `$binary` out of `${NIX_MAIN_PROGRAM}` and `${pname}`, in that order, to build roughly `${placeholder "out"}/bin/$binary`. `${NIX_MAIN_PROGRAM}`'s value comes from `meta.mainProgram`, and does not normally need to be set explicitly. When setting `versionCheckProgram`, using `$out` directly won't work, as environment variables from this variable are not expanded by the hook. Hence using `placeholder "out"` is unavoidable.
- `versionCheckProgramArg`: The argument that needs to be passed to `versionCheckProgram`. If undefined the hook tries first `--version` and then `--help`. Examples: `version`, `-V`, `-v`.
- `versionCheckKeepEnvironment`: A list of environment variables to keep and pass to the command. Only those variables should be added to this list that are actually required for the version command to work. If it is not feasible to explicitly list all these environment variables you can set this parameter to the special value `"*"` to disable the `--ignore-environment` flag and thus keep all environment variables.
- `preVersionCheck`: A hook to run before the check is done.
- `postVersionCheck`: A hook to run after the check is done.

This check assumes the executable is _hermetic_. If environment variables such as `PATH` or `HOME` are required for the program to function, then [`testers.testVersion`](#tester-testVersion) is currently the better alternative.

# wafHook

[Waf](https://waf.io) is a Python-based software building system.

In Nixpkgs, `wafHook` overrides the default configure, build, and install phases.

## Variables controlling wafHook

### `wafHook` Exclusive Variables

The variables below are exclusive of `wafHook`.

#### `wafPath`

Location of the `waf` tool. It defaults to `./waf`, to honor software projects that include it directly inside their source trees.

If the file pointed by `wafPath` doesn't exist, then `waf` provided by Nixpkgs will be used.

#### `wafFlags`

Controls the flags passed to waf tool during build and install phases. For settings specific to build or install phases, use `wafBuildFlags` or `wafInstallFlags`, respectively.

#### `dontUseWafConfigure`

When set to true, don't use the predefined `wafConfigurePhase`.

#### `dontUseWafBuild`

When set to true, don't use the predefined `wafBuildPhase`.

#### `dontUseWafInstall`

When set to true, don't use the predefined `wafInstallPhase`.

### Similar variables

The following variables are similar to their `stdenv.mkDerivation` counterparts.

| `wafHook` Variable    | `stdenv.mkDerivation` Counterpart |
| --------------------- | --------------------------------- |
| `wafConfigureFlags`   | `configureFlags`                  |
| `wafConfigureTargets` | `configureTargets`                |
| `wafBuildFlags`       | `buildFlags`                      |
| `wafBuildTargets`     | `buildTargets`                    |
| `wafInstallFlags`     | `installFlags`                    |
| `wafInstallTargets`   | `installTargets`                  |

### Honored variables

The following variables commonly used by `stdenv.mkDerivation` are honored by `wafHook`.

- `prefixKey`
- `enableParallelBuilding`
- `enableParallelInstalling`

# writableTmpDirAsHomeHook

This setup hook provides a writable home directory for packages that require it.

To use, just add the hook to the `nativeBuildInputs` of the package.

# xcbuildHook

Overrides the build and install phases to run the "xcbuild" command. This hook is needed when a project only comes with build files for the XCode build system. You can disable this behavior by setting buildPhase and configurePhase to a custom value. xcbuildFlags controls flags passed only to xcbuild.

# `xfce4-dev-tools`

This setup hook attempts to run `xdt-autogen` in `xdtAutogenPhase`, which is part of `preConfigurePhases`.

[]{#dontUseXdtAutogenPhase} This behavior can be disabled by setting `dontUseXdtAutogenPhase` to `true`.

# Zig

[Zig](https://ziglang.org/) is a general-purpose programming language and toolchain for maintaining robust, optimal and reusable software.

In Nixpkgs, `zig` overrides the default build, check and install phases.

## Example code snippet

```nix
{
  lib,
  stdenv,
  zig,
}:

stdenv.mkDerivation {
  # . . .

  nativeBuildInputs = [ zig ];

  zigBuildFlags = [ "-Dman-pages=true" ];

  dontUseZigCheck = true;

  # . . .
}
```

## Variables controlling zig

### `zig` Exclusive Variables

The variables below are exclusive to `zig`.

#### `dontUseZigConfigure`

Disables using `zigConfigurePhase`.

#### `dontUseZigBuild`

Disables using `zigBuildPhase`.

#### `dontUseZigCheck`

Disables using `zigCheckPhase`.

#### `dontUseZigInstall`

Disables using `zigInstallPhase`.

#### `dontSetZigDefaultFlags`

Disables using a set of default flags when performing zig builds.

### Similar variables

The following variables are similar to their `stdenv.mkDerivation` counterparts.

| `zig` Variable    | `stdenv.mkDerivation` Counterpart |
| ----------------- | --------------------------------- |
| `zigBuildFlags`   | `buildFlags`                      |
| `zigCheckFlags`   | `checkFlags`                      |
| `zigInstallFlags` | `installFlags`                    |

### Variables honored by zig

The following variables commonly used by `stdenv.mkDerivation` are honored by `zig`.

- `prefixKey`
- `dontAddPrefix`
