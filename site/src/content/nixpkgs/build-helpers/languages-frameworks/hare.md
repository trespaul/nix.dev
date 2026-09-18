---
title: Hare
---

# Hare

## Building Hare programs with `hareHook`

The `hareHook` package sets up the environment for building Hare programs by
doing the following:

1. Setting the `HARECACHE`, `HAREPATH` and `NIX_HAREFLAGS` environment variables;
1. Propagating `harec`, `qbe` and two wrapper scripts for the hare binary.

It is not a function as is the case for some other languages --- _e. g._, Go or
Rust ---, but a package to be added to `nativeBuildInputs`.

## Attributes of `hareHook`

The following attributes are accepted by `hareHook`:

1. `hareBuildType`: Either `release` (default) or `debug`. It controls if the
   `-R` flag is added to `NIX_HAREFLAGS`.

## Example for `hareHook`

```nix
{
  hareHook,
  lib,
  stdenv,
}:
stdenv.mkDerivation {
  pname = "<name>";
  version = "<version>";
  src = "<src>";

  nativeBuildInputs = [ hareHook ];

  meta = {
    description = "<description>";
    inherit (hareHook) badPlatforms platforms;
  };
}
```

## Cross Compilation

`hareHook` should handle cross compilation out of the box. This is the main
purpose of `NIX_HAREFLAGS`: In it, the `-a` flag is passed with the architecture
of the `hostPlatform`.

However, manual intervention may be needed when a binary compiled by the build
process must be run for the build to complete --- _e. g._, when using Hare's
`hare` module for code generation.

In those cases, `hareHook` provides the `hare-native` script, which is a wrapper
around the hare binary for using the native (`buildPlatform`) toolchain.
