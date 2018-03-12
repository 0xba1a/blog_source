---
title: Raspberry Pi dishes from Yocto cuisine
date: 2018-03-11 17:10:50
thumbnail: "/images/raspberry_pi_3_yocto.jpg"
tags:
 - RaspberryPi
 - yocto
 - build
---

Recently I got some board bring-up work where I come across Yocto project. To complete that project, I had to understand a little more than the basics of Yocto. So I had some hands on like [Yocto Project Quick Start](https://www.yoctoproject.org/docs/2.4.1/yocto-project-qs/yocto-project-qs.html) and [Yocto Project Linux Kernel Development Manual](https://www.yoctoproject.org/docs/2.4.1/kernel-dev/kernel-dev.html). Seems Yocto is so powerful thus I thought of starting to use it for my Raspberry Pi hacking. As expected there are already people tried doing so and there are some good blogs about what they have accomplished. But to my surprise, there is already a meta layer for Raspberry Pi 3. It makes the work even simpler.

I created a directory in Desktop as base for the development. Inside it I created `build`, `downloads`, `sstate`, `tmp` and `yocto` layers directories.
```sh
$ ls
build  downloads  sstate  sstate-cache  tmp  yocto
```
With reference to the [meta-raspberry quick start](http://meta-raspberrypi.readthedocs.io/en/latest/readme.html#quick-start) page, I have cloned [poky](http://git.yoctoproject.org/cgit.cgi/poky), [meta-openembedded](http://git.openembedded.org/meta-openembedded) and [meta-raspberry](https://github.com/agherzan/meta-raspberrypi) inside the `yocto` directory.
```sh
$ cd yocto
$ git clone git://git.yoctoproject.org/poky.git
$ git clone git://git.openembedded.org/meta-openembedded
$ git clone git@github.com:agherzan/meta-raspberrypi.git
```
Sourced the `oe-init` script to setup build environment. And then configured the `conf/bblayers.conf` file to include `meta-layers` from meta-openembedded and meta-raspberry layers. The [meta-raspberry quick start](http://meta-raspberrypi.readthedocs.io/en/latest/readme.html#quick-start) specifies `meta-oe`, `meta-multimedia`, `meta-networking` and `meta-python` as dependencies for `meta-raspberry`. So I'm including all of the into my `conf/bblayers.conf` file.
```sh
$ source yocto/poky/oe-init-build-env build/rpi3/

### Shell environment set up for builds. ###

You can now run 'bitbake <target>'

Common targets are:
    core-image-minimal
    core-image-sato
    meta-toolchain
    meta-ide-support

You can also run generated qemu images with a command like 'runqemu qemux86'
$ pwd
/home/kaba/Desktop/yocto/build/rpi3
$ cat conf/bblayers.conf 
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  ${TOPDIR}/../../yocto/poky/meta \
  ${TOPDIR}/../../yocto/poky/meta-poky \
  ${TOPDIR}/../../yocto/poky/meta-yocto-bsp \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-oe \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-multimedia \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-networking \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-python \
  ${TOPDIR}/../../yocto/meta-raspberrypi \
  "
$
```
Then made following changes in `conf/local.conf`.
 * Set the `MACHINE` variable to point raspberry Pi 3 [raspberrypi3-64].
 * Set `SSTATE_DIR`, `TMPDIR` and `DL_DIR` to point to `sstate`, `tmp` and `downloads` directories respectively.
 * I usually run my Pi headless. So I enabled `ssh-server-openssh` **EXTRA_IMAGE_FEATURE**. Its up to your choice.
 * Left the other default options as it was.

```sh
$ ls ../../yocto/meta-raspberrypi/conf/machine/
include                 raspberrypi2.conf     raspberrypi-cm3.conf
raspberrypi0.conf       raspberrypi3-64.conf  raspberrypi-cm.conf
raspberrypi0-wifi.conf  raspberrypi3.conf     raspberrypi.conf
$
$ cat conf/local.conf 
MACHINE = "raspberrypi3-64"
DISTRO ?= "poky"
PACKAGE_CLASSES ?= "package_rpm"
EXTRA_IMAGE_FEATURES ?= "debug-tweaks ssh-server-openssh"
USER_CLASSES ?= "buildstats image-mklibs image-prelink"
PATCHRESOLVE = "noop"
CONF_VERSION = "1"

SSTATE_DIR ?= "${TOPDIR}/../../sstate-cache"
TMPDIR ?= "${TOPDIR}/../../tmp"
DL_DIR ?= "${TOPDIR}/../../downloads"

PACKAGECONFIG_append_pn-qemu-native = " sdl"
PACKAGECONFIG_append_pn-nativesdk-qemu = " sdl"
BB_DISKMON_DIRS ??= "\
    STOPTASKS,${TMPDIR},1G,100K \
    STOPTASKS,${DL_DIR},1G,100K \
    STOPTASKS,${SSTATE_DIR},1G,100K \
    STOPTASKS,/tmp,100M,100K \
    ABORT,${TMPDIR},100M,1K \
    ABORT,${DL_DIR},100M,1K \
    ABORT,${SSTATE_DIR},100M,1K \
    ABORT,/tmp,10M,1K"

#SDKMACHINE ?= "i686"
#ASSUME_PROVIDED += "libsdl-native"
$
```
The variable `${TOPDIR}` represents the build directory by default.

Check available image types for Raspberry Pi under the directory `../../yocto/meta-raspberry/recipes-core/images`. I was using the minimal `rpi-hwup-image`.
```sh
$ ls ../../yocto/meta-raspberrypi/recipes-core/images/
rpi-basic-image.bb  rpi-hwup-image.bb  rpi-test-image.bb
$
$ bitbake rpi-hwup-image
WARNING: Host distribution "ubuntu-17.10" has not been validated with this version of the build system; you may possibly experience unexpected failures. It is recommended that you use a tested distribution.
ERROR: ParseError at /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-raspberrypi/recipes-devtools/python/rpio_0.10.0.bb:9: Could not inherit file classes/pypi.bbclass

Summary: There was 1 WARNING message shown.
Summary: There was 1 ERROR message shown, returning a non-zero exit code.
$
```
Surprised? Nah! If it went through without a problem that would be a surprise. After all I wouldn't be sitting here writing this blog if it was a breeze. Of course its not a rocket science either. Initially I thought some variable I configured wrongly, so the path to `pypi.bbclass` couldn't be reached. I searched in the `yocto` directory for `pypi.class` file. To my surprise, the file is not found!
```
$ find ../../yocto/ -name 'pypi.class'
$
```
Ah! Something has changed in either `poky` or `meta-openembedded` that broke the dependency of `rpio_0.10.0.bb` of `meta-raspberry`. Some googling brought me to this [page](https://github.com/agherzan/meta-raspberrypi/issues/195). It seems `meta-openembedded` developers have removed the `pypi.class` as it is added to `oe-core` - *openembedded-core* - just a month back. I don't completely understand the difference between `meta-openembedded` and `openembedded-core` layers. So I simply cloned the `openembedded-core` repo and added it into the `bblayers.conf`.
```sh
$ git clone git@github.com:openembedded/openembedded-core.git ../../yocto/
$ ls ../../yocto/
meta-openembedded  meta-raspberrypi  openembedded-core  poky
$
$ head -n 13 conf/bblayers.conf | tail -n 3
${TOPDIR}/../../yocto/poky/meta-yocto-bsp \
${TOPDIR}/../../yocto/openembedded-core/meta \
${TOPDIR}/../../yocto/meta-openembedded/meta-oe \
$
```
Started the build again
```sh
$ bitbake rpi-hwup-image
WARNING: Host distribution "ubuntu-17.10" has not been validated with this version of the build system; you may possibly experience unexpected failures. It is recommended that you use a tested distribution.
WARNING: /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtevent/libtevent_0.9.33.bb: Exception during build_dependencies for do_compile
WARNING: /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtevent/libtevent_0.9.33.bb: Error during finalise of /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtevent/libtevent_0.9.33.bb
WARNING: /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtdb/libtdb_1.3.15.bb: Exception during build_dependencies for do_compile
WARNING: /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtdb/libtdb_1.3.15.bb: Error during finalise of /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtdb/libtdb_1.3.15.bb
ERROR: ExpansionError during parsing /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtevent/libtevent_0.9.33.bb
Traceback (most recent call last):
bb.data_smart.ExpansionError: Failure expanding variable do_compile, expression was     python ./buildtools/bin/waf ${@oe.utils.parallel_make_argument(d, '-j%d', limit=64)}
 which triggered exception AttributeError: module 'oe.utils' has no attribute 'parallel_make_argument'

WARNING: /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtalloc/libtalloc_2.1.10.bb: Exception during build_dependencies for do_compile
WARNING: /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtalloc/libtalloc_2.1.10.bb: Error during finalise of /home/kaba/Desktop/yocto/build/rpi3/../../yocto/meta-openembedded/meta-networking/recipes-support/libtalloc/libtalloc_2.1.10.bb

Summary: There were 7 WARNING messages shown.
Summary: There was 1 ERROR message shown, returning a non-zero exit code.
$
```
Failed again!
Here the catch is `module 'oe.utils' has no attribute 'parallel_make_argument'`. Lets see who defined the function `parallel_make_argument`.
```sh
$ grep -rn 'def parallel_make_argument' ../../yocto/
../../yocto/openembedded-core/meta/lib/oe/utils.py:182:def parallel_make_argument(d, fmt, limit=None):
$
```
Yup! The function `parallel_make_argument` is defined in the file `oe/utils.py`. So whats wrong. There is one more `oe/utils.py` file which is under `poky/meta` directory. Even though `poky` is mentioned as a dependency for `meta-raspberry`, lets just give a try removing `poky/meta` and see what happens as we are now using `openembedded-core/meta`.
```sh
$ cat conf/bblayers.conf 
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  ${TOPDIR}/../../yocto/openembedded-core/meta \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-oe \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-multimedia \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-networking \
  ${TOPDIR}/../../yocto/meta-openembedded/meta-python \
  ${TOPDIR}/../../yocto/meta-raspberrypi \
  "
$ bitbake rpi-hwup-image
ERROR:  OE-core's config sanity checker detected a potential misconfiguration.
    Either fix the cause of this error or at your own risk disable the checker (see sanity.conf).
    Following is the list of potential problems / advisories:

    DISTRO 'poky' not found. Please set a valid DISTRO in your local.conf


Summary: There was 1 ERROR message shown, returning a non-zero exit code.
$
```
Okay my bad. Lets change `DISTRO` from `poky` to `nodistro` and try.
```sh
$ head -n 4 conf/local.conf 
MACHINE = "raspberrypi3-64"
#DISTRO ?= "poky"
DISTRO ?= "nodistro"
PACKAGE_CLASSES ?= "package_rpm"
$ bitbake rpi-hwup-image
Parsing recipes: 100% |######################################################################################################################################################################| Time: 0:04:09
Parsing of 2077 .bb files complete (0 cached, 2077 parsed). 2933 targets, 146 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "1.36.0"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "ubuntu-17.10"
TARGET_SYS           = "aarch64-oe-linux"
MACHINE              = "raspberrypi3-64"
DISTRO               = "nodistro"
DISTRO_VERSION       = "nodistro.0"
TUNE_FEATURES        = "aarch64"
TARGET_FPU           = ""
meta                 = "master:e4da78229f0bd67fd34928eafe48dbdc9e8da050"
meta-oe              
meta-multimedia      
meta-networking      
meta-python          = "master:271ce3576be25d8012af15961f827aa2235e6434"
meta-raspberrypi     = "master:79ea44b997a6eb6e2f0d36b1d583c85fa9e37f36"

Initialising tasks: 100% |###################################################################################################################################################################| Time: 0:00:05
NOTE: Executing SetScene Tasks
NOTE: Executing RunQueue Tasks
WARNING: rpi-hwup-image-1.0-r0 do_image: The image 'rpi-hwup-image' is deprecated, please use 'core-image-minimal' instead
NOTE: Tasks Summary: Attempted 1966 tasks of which 5 didn't need to be rerun and all succeeded.

Summary: There was 1 WARNING message shown.
$
```
Tadaannn!!!

### Installation and booting
If everything goes well, you can see the final image `tmp/deploy/images/raspberrypi3-64/rpi-hwup-image-raspberrypi3-64.rpi-sdimg`. But in my case the path is `tmp-glibc` instead of `tmp` - *the final image path was `tmp-glibc/deploy/images/raspberrypi3-64/rpi-hwup-raspberrypi3-64.rpi-sdimg`*. The path is relative to your build directory - *`build/rpi3`*. It is a soft-link to your latest build image. `dd` it to your sd-card - *`/dev/sdb` in my case* - and boot the Raspberry Pi 3 with it.
```sh
$ sudo umount /dev/sdb*
$ sudo dd if=tmp-glibc/deploy/images/raspberrypi3-64/rpi-hwup-image-raspberrypi3-64.rpi-sdimg of=/dev/sdb bs=4M
$ sudo umount /dev/sdb*
```
As for as I remember , I didn't configure any IP address as default for the board. So I connected a display to the Pi and checked. The `sshd` daemon is running but I couldn't see `wifi` interfaces enabled. Let me figure that out and write in next post.

### References
 * [http://meta-raspberrypi.readthedocs.io/en/latest/readme.html]
 * [http://www.jumpnowtek.com/rpi/Raspberry-Pi-Systems-with-Yocto.html]
 * [https://raspinterest.wordpress.com/2016/11/30/yocto-project-on-raspberry-pi-3/]
 * [https://stackoverflow.com/questions/35904769/what-are-the-differences-between-open-embedded-core-and-meta-openembedded]
 * [https://github.com/agherzan/meta-raspberrypi/issues/195]
 * [https://patchwork.openembedded.org/patch/36139/]
 * [file://openembedded-core/meta/lib/oeqa/selftest/cases/imagefeatures.py]
