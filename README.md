# Build

Yocto build for the Lightspeed Rogue board. In addition
to `yocto/poky`, the following layers are required:

- meta-rogue
- meta-st-openstlinux
- meta-st-stm32mp
- meta-openembedded
- meta-qt5
- meta-mender

The build system is assumed to be a Debian-based Linux system.

The canonical system image is `rogue-image-som-evkit`. This includes
`st-image-resize-initrd` as a dependency for use as a ramdisk.


### Yocto Resources

For those unfamiliar with the Yocto build system, consult the official upstream
Yocto documentation. Of relevant interest is the conceptual overview, and
reference manual for both Yocto and bitbake. Documentation can be found at:

- [Yocto Overview & Concepts](https://docs.yoctoproject.org/overview-manual/index.html)
- [Yocto Reference Manual](https://docs.yoctoproject.org/ref-manual/index.html)
- [Bitbake User Manual](https://docs.yoctoproject.org/bitbake/index.html)

Further potentially useful portions of the documentation are:

- [Yocto BSP Guide](https://docs.yoctoproject.org/bsp-guide/index.html)
- [Development Task Manual](https://docs.yoctoproject.org/dev-manual/index.html)
- [SDK and eSDK Manual](https://docs.yoctoproject.org/sdk-manual/index.html)


### Installation of base utilities

The following utilities/packages must be available on the
build system. As such, these instructions need only be performed
once.

```
apt install bash build-essential chrpath cpio curl diffstat gawk rsync wget xz-utils
apt install python3-pip python3-pexpect
apt install libsdl2-dev
apt install repo
```


### Setup and Checkout

The `repo` utility is used to check out and manage versions
of the layers according to the definitions in the manifest XML file.
Once checked out, there is no further technical requirement for this tool.

First, create a new directory and clone this build repository as a subdirectory.
```
# Clone this repository
git clone ... build

cd build
```

Next, use `repo` to clone the appropriate versions of
the public metadata layers according to (one of) the manifest
XML files in this repository.
```
repo init -u "$PWD" -m manifest.xml
repo sync
```

Next, clone the `meta-rogue` repository to the `meta-rogue` subdirectory.
```
# Clone meta-rogue
git clone .../meta-rogue.git meta-rogue
```

And last, clone the rogue-base-fw-som repository beneath its recipe.
```
# Clone rogue app
cd meta-rogue/recipes-rogueapp/rogueapp/rogueapp
git clone .../rogue-base-fw-som.git src
```


### Set yocto Build Environment
Yocto comes with a source-able script to calculate and
export the necessary build environment variables. This script can be
sourced directly from within the `yocto` directory that was created during
the `repo sync` process. It takes a single argument of a path to this
repository.

A `buildenv.sourceme` convenience wrapper that does not require any arguments
is provided in this repository and can be used instead. Simply sourcing this
file from within this repository's directory will automatically handle
everything.

Either of these two options will ensure the required bitbake variables get
correctly calculated and exported into the environment of the currently-running
shell process.

Source the environment script by running either of the command blocks
below (both are equivalent):

```
. buildenv.sourceme
```

Or

```
# Since yocto is cloned to a directory within this build directory, the
# build directory argument should be yocto's parent directory (..).

cd yocto
. oe-init-build-env ..

# Here, the current directory will have automatically been changed back to
# the build directory -- the path of this repository -- so there is no need
# to cd back.
```


### Board Selection and Build Settings
All builds will use the settings from the `conf/local.conf`
configuration file. The most important settings are:

Variable              | Description
----------------------|---------------
`PARALLEL_MAKE`       | Options to pass to make commands. Each bitbake job builds with this many threads.
`BB_NUMBER_THREADS`   | Number of bitbake tasks to perform simultaneously.
`MACHINE`             | Embedded machine to build image for.


Take care to note that `PARALLEL_MAKE` is not a number, but rather the option
that gets passed to each invocation of the `make` command; this typically
begins with `-j`, which is the mnemonic for "job server". Since recipes
typically spend the majority of time doing compilation, parallelization
of this process greatly reduces build times.

`PARALLEL_MAKE` is independent of `BB_NUMBER_THREADS`: due to certain
inherently serial portions of all build procedures, CPU core usage will
approach -- but can never reach -- the product of these two numbers. A
good rule of thumb is to have the product equal to roughly 5/4 the number
of CPU cores, but these should be tuned on a per-machine basis.


```
# Example.
# (See comments in `conf/local.conf` for more).

PARALLEL_MAKE      = "-j8"
BB_NUMBER_THREADS  = 3
MACHINE           ?= 'stm32mp15-ya157c-v2'
```


### Run bitbake
The `bitbake` command can be used to start and monitor the
build. The build can be run in a single shot, or performed
in stages.
The canonical system image is `rogue-image-som-evkit`. This includes

```
# Just do checkout of all source trees.
bitbake rogue-image-som-evkit --runonly=fetch

# Build bootable image.
bitbake rogue-image-som-evkit

# Build redistributable development SDK.
bitbake rogue-image-som-evkit -c populate_sdk

# Clean all, without removing initial sources.
bitbake rogue-image-som-evkit --runall=cleansstate
```

While builds generally compile only what has changed since
the last build, the very first build can take a considerable
amount of time. A machine with at least 16 GB of RAM is recommended,
though 8GB is probably also fine.


------------

**Developer Note**

If, during development, the message
`WARNING: ...package.bb:do_compile is tainted from a forced run`
appears, you can clean the package's current state via

```
bitbake package -c cleansstate
bitbake package
```


### Build Outputs
The build places outputs in the `deploy` subdirectory, particularly
`deploy/images`.


### Flashing to eMMC
TODO



### Build Management
Periodically, as the build changes over time, the below command can be
issued in order to trim outdated sstate cache objects and reclaim disk
space:
```
sstate-cache-management.sh -d -v --cache-dir=cache/sstate
```



## Other Considerations (Docker)
The BSP code and bitbake scripts have all been developed on a native Linux
system. While it is **not recommended to build images on a non-Linux
device**, doing so is still possible via Docker container or a Virtual
Machine (VM). Both are viable options with different strengths and
weaknesses, but due to VMs generally having a much higher relative overhead,
it is likely that most developers would prefer Docker.

Following the instructions above, one can construct a Dockerfile and
corresponding image suitable for the Yocto build environment. However,
there are some additional considerations to be taken into account. Some
of these are outside the scope of this README, but some high-level
guidelines are enumerated below.


### Memory
Ensure the Docker environment has enough RAM available to successfully
compile the image. Insufficient RAM will likely result in unexpected build
failures.

### Disk
Build the image requires a large amount of available disk space. There
are high amounts of random access to small files, so some sort of solid
state disk is highly recommended.

### CPU
CPU performance is less important, though naturally it does directly impact
overall build time. Of particular importance for portable systems is that
the container's CPU allocation bounds impacts concurrent use of the
system while the build runs in the background.

Once Docker has been configured to use the desired number of CPU cores,
the [config file variables](#board-selection-and-build-settings) should
be appropriately modified to match.

### Git
Repositories will very likely need to be cloned inside of the Docker
container due to limitations inherent in the host's filesystem --
particularly HFS (the standard OS X filesystem). To avoid having to
import SSH credentials into the Docker container, one option to consider
is to create and register a read-only OAuth access token for access and
use that.

### System Locale
The build will fail if the container's system locale is not set correctly.
The locale should be set to `en_US.UTF-8`. It is recommended to set this
within the Dockerfile itself.

### Non-privileged User
The build will fail when run as the root user. It is recommended to create
a second, non-privileged user within the Dockerfile where appropriate,
e.g. via `adduser` commands or other similar mechanism.



# Release Process (and technical descriptions)

The release process involves multiple repositories, and at the moment depends
on versioning both the `meta-rogue` and the `base-fw-som` repositories.

### Versioning

The important repository Yocto-wise is of course `meta-rogue`.

Its versioning at the moment depends on annotated Git tags and a string used by
the Mender updater system.

Examples of the Git tags are in the `meta-rogue` repository history. Once a set
of changes has been merged into the main branch, the merge commit can be tagged.
For example, to tag a version v7.1.00, the following command should be run:
`git tag -a v7.1.00`, after which a title and release notes can be added in your
configured text editor.

This version string must also be reflected in a Mender configuration variable,
`MENDER_ARTIFACT_NAME`. With Mender by default, this resides in local.conf, but
since the primary repository reflected by this version is `meta-rogue`, it is
defined in the top level Rogue machine configuration file,
`conf/machine/stm32mp15-ya157c-v2.conf`. At the moment, this must be changed
separately from the git tag. This does allow for easier changing of the artifact
name for testing updates, but might be best automated to match the latest Git
tag, or even the output of a `git describe` invocation.

The `recipes-rogueapp/rogueapp` recipe in `meta-rogue` also requires attention
to versioning. While the Mender-based update bundle does not include the
persistent data partition where the rogue application is stored, the full-device
image used to flash devices on the production line, includes that partition,
with the rogue app already installed thanks to the `rogueapp` recipe. Because
the recipe doesn't yet presume a specific repository location for the rogue app,
you must manually clone the rogue application source to the directory:
`[meta-rogue]/recipes-rogueapp/rogueapp/rogueapp/src/` (see the final step in
"Setup and Checkout" for initial cloning). Once it's cloned, you can check out
the appropriate tag which should be included in the factory image for this
release.

***NOTE:*** At the moment, the rogueapp recipe can have incremental build issues
after its initial fetch and compile steps. Unless this is fixed, it's necessary,
if the recipe or `rogueapp/src/` has been updated, to run
`bitbake cleanall rogueapp` before building the image.

### Bundling

So far it's at least somewhat a manual process. The particular files of interest
didn't shake out until Mender-based updates were fully in place in late 2022.

**Full device (Factory) image**

The factory image comprises several binary images relative to a manifest file,
`FlashLayout_emmc.tsv`, to be used with the STM32CubeProgrammer utility (GUI or
command line).

***STM32CubeProg manifest edits:*** There are two side effects of the Mender
integration that we ported:
1. The U-boot configuration needs to live in a raw flash location rather than
in the root or boot filesystems. ST's normal ways of calculating partition
offsets don't accommodate that. Since we're putting the configurations between
the U-boot partition and the `rootfs` (Rootfs A) partition, the `rootfs`
partition offset needs to be higher.
2. The offsets for the subsequent partitons (rootfsb and data) are not correctly
calculated.

The upshot is that the offets of the partitions `rootfs`, `rootfsb`, and
`dataimg` need to be edited. This can probably be automated in the future, now
that we have all of the partition sizes and offsets nailed down.

The values below provide ample room for the u-boot environments, and around 100%
excess headroom for the rootfs partitions, in case other components are built in
down the road, while still leaving around 2GB for the persistent data partition,
which can flexibly acommodate larger media files or other yet-unplanned needs.

```
P	0x10	rootfs	FileSystem	mmc1	0x00580000
PED	0x11	rootfsb	FileSystem	mmc1	0x20580000
P	0x12	dataimg	FileSystem	mmc1	0x40580000
```
**MANUALLY EDIT YOUR FlashLayout_emmc.tsv FILE WITH THE ABOVE VALUES!**

As you can read in the FlashLayout_emmc.tsv manifest file, the following files
are indicated for STM32CubeProgrammer to flash to a device in USB DFU mode, by
relative path:
```
bootloaders/tf-a-rogue.stm32
bootloaders/u-boot-rogue.stm32
rogue-image-som-evkit-rogue-stm32mp15-ya157c-v2.ext4
rogue-image-som-evkit-rogue-stm32mp15-ya157c-v2-[date stamp].dataimg.ext4
```
And the `FlashLayout_emmc.tsv` file itself.

As the size of the entire image directory would be cumbersome, adding just these
necessary files is ideal.

Zipping just these files, preserving the relative paths from the FlashLayout
file, constitutes a flashable full image bundle.


**Mender update bundle**

This single file, which is only the Linux rootfs partition and Mender metadata,
is generated by the image build process thanks to the Mender integrations. In
the `dist-glibc/deploy/images/stm32mp15-ya157c-v2/` directorywill be a file
`rogue-image-som-evkit-rogue-stm32mp15-ya157c-v2.mender` linkedto a datestamped
.mender file. This can be used directly with the mender client in the C25's
Linux shell, or bundled with the `bundle.py` script to create an OTA bundle,
like with Rogue application and DCX firmware updates. Like the other artifacts,
naming is important. Linux rootfs .mender artifacts must be renamed to use the
prefix `C25_OS`.

Example invocation of `bundle.py` with an already-renamed .mender file:

`python3 ./bundle.py -name v7.1.03 -f OSrelease7.1.03.tgz ./C25_OSv7.1.03.mender`

This will produce `OSrelease7.1.03.tgz` which can be deployed from the Cascadia Web
Console to update the Linux rootfs of C25 devices.
