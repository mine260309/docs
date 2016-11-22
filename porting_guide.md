# OpenBMC porting guide

This document is intended to provide a guide to porting OpenBMC on a new machine.

In this document, the machine name `m1` of manufacturer `fac` is added to OpenBMC as an example.

## Create a new project

To support a new machine, the machine related layer and config files shall be creatd for bitbake.

Refer to `meta-openbmc-machines/meta-openpower/meta-ibm/meta-palmetto`:

1. Create `meta-fac` under `meta-openbmc-machines/meta-openpower` as a directory of the manufacture
2. Create `meta-m1` under `meta-fac` as a directory of the machine
3. Create a conf dir in `meta-fac`, following `meta-ibm/conf`
4. Use machine name `m1` in config files


Below directory tree shows the structure under `meta-openbmc-machines/meta-openpower`.

```
meta-fac
├── conf
│   ├── layer.conf
│   └── machine
│       └── include
│           └── fac.inc
└── meta-m1
    ├── conf
    │   ├── bblayers.conf.sample
    │   ├── conf-notes.txt
    │   ├── layer.conf
    │   ├── local.conf.sample
    │   └── machine
    │       └── m1.conf
    ├── recipes-kernel
    │   └── linux
    │       ├── linux-obmc
    │       │   └── m1.cfg
    │       └── linux-obmc_%.bbappend
    ├── recipes-phosphor
    │   └── workbook
    │       └── m1-config.bb
    └── recipes.txt
```

## Linux kernel

### Device tree

The machine's device tree shall be added to linux kernel.

OpenBMC's linux is at a separated repo https://github.com/openbmc/linux

Refer to `arch/arm/boot/dts/aspeed-bmc-opp-palmetto.dts`, add machine's device tree `aspeed-bmc-opp-m1.dts`

### Device driver

If the machine uses new drivers, the device driver shall be contributed to `linux` repo.

If the machine uses drivers existed in linux, the kernel config shall be set and defined in `m1.cfg` (See [Create a new project](#create-a-new-project))

## Uboot
TODO

## Power on sequence

Power on sequence logic is implmemented by OpenBMC, a new machine only needs to define related GPIOs to make it work.

The config file is in skeleton which is in a separated repo https://github.com/openbmc/skeleton

Refer to `configs/Palmetto.py`, create `configs/M1.py` and define the correct GPIO pins.

See below table for the power on related GPIOs.

| Name           | Direction | Description   |
| -------        | --------- | ------------- |
| BMC_POWER_UP   | out       | Write 1 to power on the system |
| SYS_PWROK_BUFF | in        | The pgood signal from power sequencer |
| FSI_DATA       | out       | The data line of FSI |
| FSI_CLK        | out       | The clock line of FSI |
| FSI_ENABLE     | out       | Enable or disable FSI |
| CRONUS_SEL     | out       | Select whether the Cronus is connected with BMC or FSP |


## Put it together

The above steps involve changes in different repos, and the changes may not be pushed to github yet, so usually you need to build with local repos.

Assume the local repos are:
* OpenBMC: `/home/user/openbmc`
* Linux: `/home/user/linux`
* Skeleton: `/home/user/skeleton`

To build with local repos, Linux and Skeleton's `bitbake` files need to be changed.

1. For Linux, in `/home/user/openbmc/meta-phosphor/common/recipes-kernel/linux/`
   * Change `linux-obmc.inc` for kernel source's URI:

      ```bash
      KSRC ?= "git:///home/user/linux/;protocol=file;branch=${KBRANCH}"
      ```

   * Change `linux-obmc_4.7.bb` for kernel revisioin:

      ```bash
      SRCREV="your-linux-kernel-git-revision"
      ```

2. For Skeleton, change `/home/user/openbmc/meta-phosphor/classes/skeleton-rev.bbclass` for skeleton's URI and revision:

   ```bash
   SRCREV ?= "your-skeleton-git-revision"
   SKELETON_URI ?= "git:///home/user/skeleton/;branch=your-branch;protocol=file"
   ```

Then do the OpenBMC build:

```bash
TEMPLATECONF=meta-openbmc-machines/meta-openpower/meta-fac/meta-m1/conf . oe-init-build-env
bitbake obmc-phosphor-image
```

## How to contribute

After the changes are tested, it's ready for push the changes to the community.

OpenBMC project does not use Github's pull-request flow. Instead, it consists of two parts:
1. The OpenBMC code except for Linux kernel source uses Gerrit, follow [contributing.md](contributing.md#submitting-changes) for how to submit changes for Gerrit review.
2. The Linux kernel source uses traditional mailing-list review.
   Contributors can subscribe mailing-list on https://lists.ozlabs.org/listinfo/openbmc, and use `git format-patch` and `git send-email` to submit kernel source review.

Tips: Use `--to=openbmc@lists.ozlabs.org --subject-prefix="PATCH linux dev-4.7"` during `format-patch`, so that the email contains the linux patch subject.

## Miscs

### Feature list
OpenBMC contains several features implemented in different repos.

TODOs:

#### IPMI

#### Chassis cotrol

#### LAN function

#### Sensors reading

#### FRU

#### Watchdog

#### Network

#### Event log

#### FAN control

#### SSH

#### SOL

#### KVM

#### Web-GUI

#### REST

#### Firmware update

BMC firmware can be updated via REST API. See [code-update.md](code-update.md) for details.
The code locates in `skeleton`'s `pyflashbmc/bmc_update.py`

### Add OEM IPMI command
TODO

### Devtool
`devtool` is a development tool provided by Yocto to modify source files that are external to OpenBMC.
See [Modifying Source Code](https://www.yoctoproject.org/docs/1.8/dev-manual/dev-manual.html#dev-modifying-source-code) for details.

Here is an example to modify linux kernel source.

```bash
devtool modify linux-obmc  # Create a local linux repo under build/workspace/sources
pushd /home/user/openbmc/build/workspace/sources/linux-obmc
# modify some code...
bitbake obmc-phosphor-image # Do the build with the modified code
```

When the local changes are completed and pushed to github, it can be removed:

```bash
devtool reset linux-obmc  # Remove the local repo from the build, but not delete local source
rm -rf /home/user/openbmc/build/workspace/sources/linux-obmc  # Delete the local repo
```

### Compile a single module
During testing and modifying the local changes, it is usually not necessary to build the whole image of OpenBMC, a single module can be re-build and tested.

This can be done by either using bitbake or SDK.

#### Bitbake build
Use `bitbake <recipe> -c do_build` to build the single recipe.

For example, if `obmc-op-control-host` code is changed, use below command to build:

```bash
bitbake obmc-op-control-host -c do_build
```

The built binary is at `build/tmp/work/armv6-openbmc-linux-gnueabi/obmc-op-control-host/1.0-r1/image/usr/sbin/`

If `obmc-op-control-host` is on local repo generated from `devtool`, the built binary locates `build/workspace/sources/obmc-op-control-host/op-hostctl` as well.


#### SDK build
TODO

Note: Maybe "Devtool" and "Compile a single module" sections can be moved to [cheatsheet.md](cheatsheet.md)?
