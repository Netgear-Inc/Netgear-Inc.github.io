---
layout: default
title: Seal SDK README
---
# SeAL SDK

The NETGEAR Service Abstraction Layer (SeAL) is a framework that provides developers a stable API and system structure so they can create and run their own custom service packages on Netgear routers. 
Each custom SeAL Service runs inside an [LXC container](https://linuxcontainers.org/).

The SeAL SDK provides the tools and documentation to get started building a custom service package. Currently the RBR750 platform supports the use of the SeAL SDK.

## Documentation
For more detail about SeAL, its components, and other related procedures, see [SEALPARTNERS](https://ntgr.atlassian.net/wiki/spaces/SEALPARTNERS/overview)

## Quickstart
This section contains the minimum steps to get started creating your first SeAL service package.

### Prerequisites
+ Laptop running a recent Linux distribution (Ubuntu 20.04 recommended, and is what was tested with the below steps)
+ NETGEAR router (model RBR750)
+ USB/RS232 serial cable/adapter

### Prepare the Router

For complete documentation of this process, see [TBD](https://ntgr.atlassian.net/wiki/spaces/SEALPARTNERS/pages/536379435/Developer+Doc+Resources)

### Access the console

You can access the firmware on the router in one of these ways:
+ Serial connection
+ Telnet
+ SSH (after set up)

Certain router models will allow enabling telnet from `routerlogin.net/debug.htm`.
Alternately, a Windows program `telnetEnable.exe` can be used. If needed, contact your Netgear rep.

For complete documentation of this process, see [TBD](https://ntgr.atlassian.net/wiki/spaces/SEALPARTNERS/pages/536379435/Developer+Doc+Resources)

#### Determine IP address

The default router gateway address is `192.168.1.1`. However, if your router's WAN port is connected to an existing `192.168.1.X` subnet, the IP address will be different, such as `10.0.0.1`.

To determine the router's LAN-side IP address, copy and then paste this command:

    ifconfig br-lan

Note: In this README this value is known as `<router_ip>`.

#### SSH Setup

Setting up SSH is strongly recommended.

On the router using the serial console or telnet enter these commands (substituting your SSH public key):

    echo "/bin/sh" >> /etc/shells
    rm /etc/dropbear/dropbear_rsa_host_key
    echo "<your ssh public key>" > /etc/dropbear/authorized_keys
    dropbear -R

Note: `dropbear` does not remain enabled after rebooting the router.
After a reboot, you must run `dropbear` (no `-R` required this time) to reenable SSH.

#### Update the Configuration File of the Router

To use SSH, you must update the configuration file of the router with the IPv4 address of your
router and your RSA key. Put these lines in your `~/.ssh/config`.

Note: These examples use OpenSSH.

    Host orbi
      HostName <router_ip>
      port 22
      User root
      IdentityFile ~/.ssh/<your_rsa_key>
      IdentitiesOnly yes

Now you can type `ssh orbi` to access a faster, more functional console, use scp, and so on.

#### Prepare the Build System

The SeAL SDK was developed and tested on Ubuntu 20.04.

To install the minimal build environment on Ubuntu 20.04, run the following commands:

    sudo apt update
    sudo apt install build-essential cmake gawk git git-lfs
    git lfs install

### Build

On the build system:

    git clone https://github.com/Netgear-Inc/sealsdk.git
    cd sealsdk
    ./build.sh

If you encounter an error, such as:

    SDK extraction...
    tar: This does not look like a tar archive
    bzip2: (stdin) is not a bzip2 file.
    tar: Child returned status 2
    tar: Error is not recoverable: exiting now

You probably didn't install git-lfs. Install and re-pull using `git lfs pull`.

### Install the Example Service
To confirm successful setup, install the example SeAL service named "seal-service-goodbye_1.0.1-0_ipq.ipk"

**Special Requirement for 4.5.3.30 Firmware**

    # This updates 4.5.3.20 packages to include necessary files for SeAL Services. It only needs to be run once.

    # On the build system, copy the update.sh file from the SDK
    scp update.sh orbi:/root

    # On the router
    /root/update.sh

Next, install the service by entering these commands:

    # On the build system:
    scp /path/to/build/bin/ipq/packages/examples/seal-service-goodbye_1.0.1-0_ipq.ipk orbi:/root

    # On the router, install using ssvc, the 'seal service' utility
    ssvc install /root/seal-service-goodbye_1.0.1-0_ipq.ipk

### Run the Example Service

Installed services start automatically.
To test the `goodbye` service is running correctly, you can 'talk' to it through a shared pipe, or through the web interface.

Shared Pipe:

    # This path is within the host-space. Inside the container, the paths are: /usr/bin/seal-goodbye and /tmp/goodbye
    /opt/services/goodbye/rom/usr/bin/seal-goodbye c /tmp/services/goodbye/goodbye
    >

    # Type 'hello'.  The service will respond
    > hello
    read 6 bytes: hello

    # Try 'goodbye'
    > goodbye

Web interface:

    # In a web browser, visit:
    https://<router_ipr>/goodbye

    # For example:
    https://192.168.1.1/goodbye

If the browser forwards you to the main router administration page, clear your browser cache, cookies, and so on, and then try again.

Tip: Use 'incognito mode' in Chrome as a test. Otherwise, a complete wipe of stored data is the way to go for Chrome.
For Firefox, clearing the cache is simpler. See [clear data for a single site](https://support.mozilla.org/en-US/kb/clear-cookies-and-site-data-firefox)

Note: In the future, 'root' access to the host system will be disallowed. Service providers will be able to use SSH to directly contect to their service container.

For now, to access the container command line for testing:

    lxc-attach goodbye

After doing so, notice that the `hostname` in the command line prompt now shows `goodbye`.
Try running `ps` to see the running processes within the container. You should only see the container processes.
To return to the host space, type `exit`.


## Running Your SeAL Service

### Building

For the most part, the contents of a SeAL Makefile are up to the service provider,
based on the rules in the OpenWrt Developers Guide: https://openwrt.org/docs/guide-developer/packages

However, there are a few additional SeAL requirements (also exemplified in the 'goodbye' Makefile):

1. Use `SEAL_SERVICE` to define the package name (instead of `PKG_NAME`, as is standard OpenWrt procedure).
   The final package name will have `seal-service-` prepended to it.
2. This file must `include $(INCLUDE_DIR)/seal/service.mk`
   This OpenWRT makefile defines boilerplate required for SeAL services.
   Check the 'goodbye' Makefile for the order of calls.
3. All SeAL Services must include `+seal-service-catalog` in their `DEPENDS:=` definition.
4. SeAL Services must not contain `preinst`, `postinst`, `prerm`, or `postrm` sections.
   Any install requirements should be handled on first run of the service,
   and SeAL will handle cleaning up after package removal.
   Files that should persist across upgrades should be stored in the 'cache' directory
   (see below for details).
5. SeAL Services must not contain the `Package/<name>/install` directive.
   Instead, install into the desired structure in `Build/Install` with `PKG_INSTALL_DIR` as the destination.
6. A `service_hook` script must be installed in `/sbin/`
   This is the only way to start your service within the container.

During the build, the framework provided by service.mk generates the necessary pieces to create a SeAL-compatible package.

On installation, the service will be installed in a specified location for services and will be launched within a container.

### Runtime Environment

#### Service Hook

Every SeAL Service must install a file `/sbin/service_hook`.  This file must begin with:

    #!/bin/sh /etc/service_hook.common

The service_hook should define `run()`, `stop()`, `status()`, and `prepare_logs()` functions.
Only the `run` function is currently required; however, all of the functions are recommended.

* The `run` function should start up your service.
* The `stop` function should shut it down cleanly.
* `status` should return status information to stdout, and a return value of `0` for success, or non-zero for failure.
* `prepare_logs` is a hook to indicate that the user has begun a log collection process.
Any logs to be included should be put into the `log_dir` (defined in the `Directories` section, below)

#### Web URL

SeAL Services are automatically assigned a URL, equivalent to the service name, on the LAN side of the router.
For example, as noted above, the `goodbye` service can be accessed with

    https://<router_ip>/goodbye

This is implemented by setting up a reverse proxy from the service name to a port assigned at install-time.
Services can determine their assigned port by checking the `SEAL_PORT` environment variable within their container.

Note: When network namespaces are enforced, this process will change.

#### Directories

The file `/etc/netgear.conf` contains:

    [seal_service]
    cache_dir = /opt/cache
    tmp_dir   = /tmp
    log_dir   = /tmp/seal_logs

Use these locations when determining where to write files.  You can use the `ssvc` utility for simple parsing:

    # ssvc get_config seal_service tmp_dir
    /tmp

Note: This utility is not optimized. Do not run it in a loop. Cache the values.

##### cache_dir

Any files that are intended to remain static across version and firmware upgrades should be stored in the cache_dir.
For example, user configuration files would generally fit within this definition.

##### tmp_dir

tmp_dir is the location to write temporary files. The contents are not preserved across reboots.
Because routers are embedded devices without high-endurance flash, it is important to avoid filesystem writes whenever possible.
Therefore, tmp_dir should be used because this location is guaranteed to be writable and safe.

##### log_dir

log_dir should be used for logs. It can be used for logs at any time, but should
particularly be prepared with logfiles when `prepare_logs` is called on the `service_hook`.

In most cases the log dir will be within the tmp_dir directory and should be considered volatile (not static across reboots).
Importantly, only files within the log_dir will be available for retrieval using user log-retrieval tools.

##### Anywhere else

Because SeAL Services run within a container, the filesystem of your service see may not map one-to-one to the underlying system,
though in principle a service has 'normal' access to read/write the filesystem heirarchy (i.e. subject to filesystem
permissions). However, be advised that some portions may be read-only or inaccessible, to prevent undesirable interactions
between the service and the host's use of system resources. Examples might include the `/dev`, `/sys` and/or `/proc` filesystem
hierarchies.

While the containers are currently run as 'privileged' (with some privileges removed), we expect to migrate to running
services within 'unprivileged' containers over time, as we attain critical mass with the supporting service APIs, which
will remove the requirement to do things 'as root'. Please let us know about areas that would become problematic if you
do not have root-level access, and APIs you might require in such cases.

Your read-only assets will be bundled together in a compressed overlay (which is read-only by nature). This can
reduce the Flash storage required by 2x-3x, so it is strongly encouraged to use this wherever possible. Your overlay is
only visible to your service, within the container.

If you need additional libraries, we strongly recommend that you express those as dependencies of your package, and avoid bundling them in your package. Resources are in general limited, and such duplication may both:
a) limit the ability of your package to run on all target products, and
b) prevent NETGEAR from updating libraries system-wide with security fixes.
If you choose to do this, expect to answer questions on why it is necessary. NETGEAR provides
packages of current versions of open source projects in our repos, if they are not provided already. However, we cannot
do this for libraries which are not freely distributable. Some licenses might also conflict with the licenses of other components in the system.

Be aware that we reserve the right to have any modifications your service makes to the filesystem limited to only
affecting the filesystem visible within your container environment. Again, the intent is to prevent undesirable
interactions, either with the underlying system, or with other services attempting to do similar things.

Be aware of the amount of write activity your service generates. Flash memory has a finite
life, and elevated levels of write activity greatly shortens it. You are strongly encouraged to write data that changes
often into the temporary directories and to periodically copy the data to persistent storage much less frequently, if
the data does need to be persisted. NETFEAR reserves the right to refuse services that we believe could harm our
devices through creating excessive wear on the on-board Flash memory.

### Additional Notes

During the build process, when editing a Makefile that isn't a part of the feed itself, the build may not reflect the edit.
To force a particular package to rebuild from scratch, delete its build_dir. For example:

    rm -rf build_dir/target-arm_cortex-a7_musl-1.1.16_eabi/seal-service-goodbye-1.0.1/

If a change is made to the packages within a feed, reload that feed with

    # replace 'examples' with your feed name
    ./scripts/feeds update -i examples && ./scripts/feeds install -a -p examples

To build with verbose logging:

    # -j1 triggers a single-threaded build (which makes visually parsing the logfiles significantly simpler)
    make -j1 V=s

To build just one package, you can address it directly.  For example:

    make -j1 V=s package/feeds/examples/goodbye/compile

If you have a serial cable, you can wipe the read-write portion of the flash (to get to the default state)

    **WARNING** This is only known to be valid for the RBR750.  **Running this command on another system could brick it!**
    
    # Break into the bootloader when you see this during boot:
    Hit any key to stop autoboot:  0

    # Then run at the prompt
    IPQ807x# nand erase 0x3e80000 0xdd80000 && nand erase 0x2080000 0x1e00000

## Missing Pieces

Please let us know how we can improve this SDK!
The goal is to make this SDK quick and easy to use.

Watch for these missing pieces to be added in a later release:
1.	Network namespace isolation and an API for packet inspection/connection tracking. Currently services are on the host network namespace.
2.	External communication/authentication and subscription service handling.
3.	Related to (2), the ability to configure the reverse proxy dynamically.

For (3), to set up a path for testing purposes, you can edit /etc/tinyproxy_reverse_config.conf to add an
additional 'ReversePath' and then call 'monit restart tinyproxy'. This allows direct access to a service externally.
