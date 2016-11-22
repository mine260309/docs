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


Below directory tree shows the structure.
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

The above steps invovlve changes in different repos, and the changes may not be pushed to github.

To build with local repos, some bitbake config files need to be changed.

Assume the local repos are:
* OpenBMC at `/home/user/openbmc`
* Linux at `/home/user/linux`
* Skeleton at `/home/user/skeleton`

The Linux and Skeleton's `bbclass` files need to be changed.
1. For Linux, in `/home/user/openbmc/meta-phosphor/common/recipes-kernel/linux/`
   * Change `linux-obmc.inc` for kernel source's URI:
      ```
      KSRC ?= "git:///home/user/linux/;protocol=file;branch=${KBRANCH}"
      ```
   * Change `linux-obmc_4.7.bb` for kernel revisioin:
      ```
      SRCREV="your-linux-kernel-git-revision"
      ```
2. For Skeleton, change `/home/user/openbmc/meta-phosphor/classes/skeleton-rev.bbclass` for skeleton's URI and revision:
   ```
   SRCREV ?= "your-skeleton-git-revision"
   SKELETON_URI ?= "git:///home/user/skeleton/;branch=your-branch;protocol=file"
   ```

Then do the OpenBMC build:
```
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
TODO:

### Add OEM IPMI command
TODO

### Devtool
TODO

### Compile a single module
TODO

Maybe "Devtool" and "Compile a single module" sections can be moved to [cheatsheet.md](cheatsheet.md)?
