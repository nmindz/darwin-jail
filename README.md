# darwin-jail

[![Build Status](https://github.com/darwin-containers/darwin-jail/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/darwin-containers/darwin-jail/actions?query=branch:main)

## Prerequisites

* macOS 12 Monterey or newer
* Disable [System Integrity Protection](https://developer.apple.com/documentation/security/disabling_and_enabling_system_integrity_protection).
SIP [doesn't allow](https://github.com/containerd/containerd/discussions/5525#discussioncomment-2685649) to `chroot` (not needed for building though).

## Usage

```shell
cd "$repo_root"
sudo python3 -m darwinjail "$jail_dir" # prepare chroot dir contents
sudo chroot "$jail_dir" # enter chroot
```

In order to make DNS work in chroot, run:

```shell
sudo mkdir -p "$jail_dir/var/run"
sudo link -f /var/run/mDNSResponder "$jail_dir/var/run/mDNSResponder"
```

## A jail that can run xcodebuild

```shell
sudo python3 -m darwinjail -f darwinjail/xcode.files "$jail_dir"
```

This needs a host where Xcode has been installed and launched once. It copies
what `xcodebuild -runFirstLaunch` installs, because that command cannot run
inside the jail: it fails to install `MobileDeviceDevelopment.pkg` with
`PKInstallErrorDomain 105`, a signature check the jail cannot satisfy
([rund#16](https://github.com/darwin-containers/rund/issues/16)).

`darwinjail/xcode.files` documents why each path is needed. Verified on macOS
26.5.1 with Xcode 26.5: `clang`, `xcrun`, `swiftc`, `swift build`,
`xcodebuild -list`, `xcodebuild build` and `codesign` all work inside the jail.

Two things outside this repository are also required:

- `sudo defaults write com.apple.dt.Xcode DVTDisableCoreDeviceLocator -bool YES`
  for the user that runs the jail. Otherwise `xcodebuild` aborts with `SIGABRT`
  in `_coredevice_xpc_add_bundle`, called from `DVTCoreDeviceLocator.startLocating()`.
  `CFPreferences` talks to the host's `cfprefsd` even from inside a chroot, so a
  plist inside the jail is ignored.
- The empty directories listed at the top of `darwinjail/xcode.files`.

## `/AppleInternal/XBS/.isChrooted`

This marker is no longer created by default; pass `--apple-internal` to get it
back. Apple's build tools read it as "running inside Apple's internal build
system" (XBS), and with it present every Xcode build fails:

```text
xcodebuild: error: Building with an inferred run destination in XBS assumes
            `SDKROOT` being set in the process environment.
```

Setting `SDKROOT=macosx` then fails with `"macosx" is not a valid SDK`.

<!--
TODO: This should fix https://github.com/darwin-containers/darwin-jail/issues/11
TODO: Doesn't work, see https://github.com/moby/buildkit/issues/5544

## Uploading Darwin rootfs as Docker image (with Docker)

Create a simple Dockerfile:

```dockerfile
FROM scratch
COPY . .
```

Build and upload Docker image to registry

```shell
sudo docker buildx build -f Dockerfile $jail_dir --output type=image,name=$image_tag,compression=zstd,oci-mediatypes=true,platform=darwin/arm64
```

-->

## Uploading Darwin rootfs as Docker image (without Docker)

```shell
brew install crane

# You might first need to authenticate using
# sudo crane auth login "$registry" -u "$username" -p "$password"

sudo bash -c 'crane append --oci-empty-base -t "$image_tag" -f <(tar -f - -c -C "$jail_dir" .)'
```

If you want to run Darwin image using containerd or Docker, see [instructions](https://github.com/darwin-containers/homebrew-formula#darwin-native-containers).
