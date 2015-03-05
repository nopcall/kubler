gentoo-bb
=========

Build framework to produce minimal root file systems based on [Gentoo][]. It's primarily intended for maintaining [LXC][] base image stacks,
but can probably fairly easy (ab)used for other use cases involving a custom root fs, cross compiling comes to mind.

Currently supported build engines:

* [Docker][]

Planned support:

* [Rocket][]

Adding new engines should be fairly easy. PR are always welcome. ;)

## Why?

* Containers should only contain the bare minimum to run
* Gentoo shines when it comes to control and optimization

## Features

* Supports 2 phase builds to produce a minimal rootfs via seperate build containers
* Maintain multiple image stacks with differing build engines
* Image/Builder dependencies, if supported by build implementation (i.e. Docker's layering)
* Automated image [documentation][nginx-packages] and history when utilizing a CVS

### Docker Features

* Tiny static busybox-uclibc image (~1.2mb) as base
* Full layering support for final images, images are not squashed
* [s6][] instead of [OpenRC][] as default supervisor (small footprint (<1mb) and proper docker SIGTERM handling)
* Images are available on [docker.io][gentoo-bb-docker]

## How much do I save?

* Quite a bit, the Nginx Docker image, for example, clocks in at ~20MB, compared to >1GB for a full Gentoo version or ~300MB for a similiar Ubuntu version

## Quick Start

    $ git clone https://github.com/edannenberg/gentoo-bb.git
    $ cd gentoo-bb
    $ ./build.sh

* If you don't have GPG available use `-s` to skip verification of downloaded files
* Check the directories in `dock/gentoobb/images/` for image specific documentation
* `bin/` contains a few scripts to start/stop container chains

## How does it work?

* `build.sh` reads global defaults from `build.conf`
* iterates over `dock/`
* reads `build.conf` in each `dock/<namespace>/` directory and imports defined `CONTAINER_ENGINE` from `inc/engine/`
* generates build order by iterating over `dock/<namespace>/images/`
* executes `build_core()` for each required engine to bootstrap the initial build container
* executes `build_builder()` for any other required build containers in `dock/<namespace>/builder/<repo>`
* executes `build_image()` to build each `dock/<namespace>/images/<repo>` 

Each implementation is allowed to only implement parts of the build process, if no build containers are required thats fine too.

### Docker specific build details

* `build_core()` builds a clean stage 3 image with portage snapshot and helper files from `./bob-core/`
* `build_image()` mounts each `dock/<namespace>/images/<repo>` directory into a fresh build container as `/config`
* executes `build-root.sh` inside build container
* `build-root.sh` reads `Buildconfig.sh` from the mounted `/config` directory
* `package.installed` file is generated which is used by depending images as `package.provided`
* if `configure_rootfs_build()` hook is defined in `Buildconfig.sh`, execute it
* `PACKAGES` defined in `Buildconfig.sh` are installed at a custom empty root directory
* if `finish_rootfs_build()` hook is defined in `Buildconfig.sh`, execute it
* resulting `rootfs.tar` is placed in `/config`, end of first build phase
* used build container gets committed as a new builder image which will be used by other builds depending on this image, this preserves exact build state
* `build.sh` then starts a normal docker build that uses `rootfs.tar` to create the final image

## Adding Docker images

 * All images must be located in `dock/<namespace>/images/`, directory name = image name
 * `Dockerfile.template` and `Buildconfig.sh` are the only required files
 * `build.sh` will pick up the image on the next run

Some useful options for `build.sh` while working on an image:

Start an interactive build container, same as used to create the rootfs.tar:

    $ ./bin/bob-interactive.sh mynamespace/myimage

Force rebuild of myimage and all images it depends on:

    $ ./build.sh -f build mynamespace/myimage

Same as above, but also rebuild all rootfs.tar files:

    $ ./build.sh -F build mynamespace/myimage

Only rebuild myimage1 and myimage2, ignore images they depend on:

    $ ./build.sh -nF build mynamespace/myimage1 mynamespace/myimage2

## Updating to a newer gentoo stage3 release

First check for a new release by running:

    $ ./build.sh update

If a new release was found simply rebuild the stack by running:

    $ ./build.sh -F

* Minor things might break, Oracle downloads, for example, may not work. You can always download them manually to `tmp/distfiles`.

## Thanks

[@wking][] for his [gentoo-docker][] repo which served as an excellent starting point

[@jbergstroem][] for tons of feedback, irc discussions and testing on [CoreOS][] <3

[LXC]: http://en.wikipedia.org/wiki/LXC
[gentoo-docker]: https://github.com/wking/dockerfile
[s6]: http://skarnet.org/software/s6/
[OpenRC]: http://wiki.gentoo.org/wiki/OpenRC
[Docker]: http://www.docker.io/
[Rocket]: https://github.com/coreos/rocket
[gentoo-bb-docker]: https://hub.docker.com/u/gentoobb/
[nginx-packages]: https://github.com/edannenberg/gentoo-bb/blob/master/bb-dock/nginx/PACKAGES.md
[Gentoo]: http://www.gentoo.org/
[CoreOS]: https://coreos.com/
[@wking]: https://github.com/wking
[@jbergstroem]: https://github.com/jbergstroem
