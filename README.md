0.0 Submitting Patches
======================

Patches to the kernel meta data should be submitted to the linux-yocto
mailing list (subscripion only) and should cc' the maintainer.

  list: https://lists.yoctoproject.org/g/linux-yocto
  Maintainers: Bruce Ashfield <bruce.ashfield@gmail.com>

When sending single patches, please using something like:

  $ git send-email -1 --to=linux-yocto@lists.yoctoproject.org --subject-prefix='kernel-cache][PATCH'

1.0 Overview
============

The linux-yocto kernel is composed of additions/modifications to the
kernel.org source, plus configuration/control data to manage and use those
changes.

Source code changes are seen as git commits to the kernel source tree, are
arranged into features (sometimes) separated by branches and marked by tags.

The configuration and control data is contained within a separate branch from
source changes called the meta branch. The configuration data is contained
within the kernel-cache directory structure, and represents the instructions
to modify the source tree and the configuration policies required to configure
and build the kernel.

While changes to the source code have already been applied to the tree, the
control and configuration data is used before and during the kernel build
process to generate a valid kernel config.

This README explains the configuration data and policies around the
organization of this information, it is not a guide to tree construction, scc
file syntax or linux-yocto architecture.

2.0 Configuration Policy
========================

The configuration data contained within the meta branch has the following
purposes:

  - Documents and defines hardware, non-hardware, required and optional
    configuration data that are used to keep software configuration policy
    and board support configuration separate. It also tags configuration data
    in a manner that an audit can be performed to ensure that polices make it
    to the final .config and that required options are not overridden or
    dropped from the final .config.

  - Creates a baseline configuration that can be inherited/included to result
    in consistent configuration across all derived kernel builds

  - Groups patches and their configuration data into documented features. The
    proper configuration and enablement of a kernel change is coupled with the
    patches that make the change to the source.

  - Creates named feature fragments that when included enable the required
    options to implement a specific behaviour (i.e. USB boot)

  - Defines BSPs (Board Support Packages) (machines) that select a policy
    (features + config) and hardware options to form a buildable, bootable
    configuration.

The policies that are contained within the meta branch can be overridden by
external descriptions using the same description format as the meta branch
configuration. This allows for flexible modification and extension of the
base policy. Also, if a previously defined BSP configuration is modified, it
can be audited against the software policy to generate a compliance report.

2.1 Kernel Types (ktypes)
-------------------------

Kernel types (ktypes) are the highest level policy containers and represent
a significant set of kernel functionality that has been grouped (and named)
or partitioned.

When functionality is partitioned it indicates that the features kept apart
since they won't work together (eg: schedulers (BFS vs CFS), or security
methods (grsec vs another LSM)). Grouped functionality means that there are
many items (features, configuration) that you want to collectively call a
"kernel type" and validate that they work together, but there's no
fundamental incompatibility between these features and others in the system.

Note: ktypes or KTYPES are seen as "define KTYPE <name>" in .scc files, and
are part of a BSP definition.
 
There are often significant differences between kernel types in the following
ways:

  - source code: large or invasive features that cannot be cleanly disabled, 
    or that cannot co-exist with other features at a source code level are 
    separated by kernel type. The preempt-rt patches, alternate schedulers,
    grsecurity, are some examples of patches that are important parts of 
    kernel type definition.

  - behaviour: A kernel type defines a default behaviour, which is often a
    trade off against other options. 

       - performance vs. determinism
       - security vs. flexibility
       - size vs features
       - ...

    are all common parts of behavioural differences between kernel types.

  - feature support: different kernel types support different sets of features,
    such as XIP or different block schedulers, tracers, network devices and 
    power management.

  - board support: due to the source, behaviour and feature differences between
    kernel types, they often dictate hardware/board support. A BSP
    definition declares which kernel types it supports by providing
    descriptions that include a kernel type and add board support configuration
    data.

Kernel types can be inherited and extended. An example inheritance tree is
below:

  base: common/basic functionality, upstream features and bug fixes
    |
    +--- standard: selected functionality and performance profile.
    |       |
    |       +--- preempt-rt: real time extensions for the base + standard
    |
    +--- tiny: base functionality + few additional features with a small footprint


2.2 Kernel Features
--------------------

Kernel features are named containers for changes to the kernel (via patches
and/or configuration) that implement or enable a defined feature. A feature
can be small, or large, simple or complex, but it always represents
functionality or behaviour that can be included by other features or kernel
types.

Within the kernel-cache, kernel features are found as $FEATURE.scc files.

If a feature contains patches, it must only be included once by a given BSP
or kernel type, since including that feature applies a source change to the
tree. Including it more than once would result in the double application of
the same patches, which will fail.

If functionality is added via patches, is frequently extended by patches, or
periodically contains patches, it is typically classified as a "feature". It
should be noted, that this is only a logical distinction from Kernel
Configuration features, since the underlying mechanism is the same.

Features are often sub-categorized into a directory structure that groups
them by maintainer defined attributes such as architecture, debug, boot, etc.

Full kernel features are found under: kernel/* in the directory tree.

Patches are a feature subtype and are simply a grouping of changes into a
named category. These typically are included by kernel types, and are not
meant to implement a defined functionality or be included multiple times.

These often contain bug fixes, backports or other small changes to the kernel
tree, and do not typically contain any kernel configuration fragments.
Configuration fragments are not required, since they are fixing bugs, or
adding simple functionality that does not need Kconfig options to be
enabled. 

Patches are normally arranged into a directory structure that makes their
maintenance and carry forward easier and are found under "patches/*" in
the directory structure.

2.3 Config Features
--------------------

Config features are collections of configuration options that when included
enable a specific behaviour or functionality. Configuration features do not
contain patches, and can be included multiple times by any other feature or
kernel type. 

The impact of configuration groups is additive, and order matters, since the
last included config group can override the behaviour of previous includes.

Pure Config features are found under "cfg/*" in the directory structure, and
are named <$config_feature>.scc. 

Configuration fragments are the actual input to the linux kernel 
configuration subsystem and are included by config features. Configuration
fragments are found throughout the tree, and are ".cfg" files.

Note: Depending on the architecture of the meta data, configuration groups
can be complete or partitioned.

Example:

   complete.scc
      include complete.cfg

   complete.cfg
      CONFIG_A=y
      CONFIG_B=y

   partitioned.scc
      include partitioned_a.cfg
      include partitioned_b.cfg

   partitioned_a.cfg
     CONFIG_A=y

   partitioned_b.cfg
     CONFIG_B=y

Complete config groups contain all the options required to enable
functionality while partitioned configurations rely on multiple includes to
build up a set of non-overlapping options to enable functionality. In the
preceding example, including complete.scc or partitioned.scc will result
in the same kernel configuration.

Complete groups are simpler to include, but make it more difficult to remove
or disable an option (since it can appear multiple times), while partitioned
configuration only has a single option in a single config group, but make it
more difficult to determine the right set of groups to include for the
desired functionality.

2.4 BSPs (Board Support Package)
--------------------------------

The BSP .scc files combine the policy from the kernel type with the
hardware requirements of the machine into a single place. This file
describes all the source code changes from patches and features and the
configuration changes that are used to configure and build the kernel. 

There is one BSP description per kernel type that is located by a build
system when it starts the process of configuring and build a kernel for a
board.

To repeat an earlier point, one of the goals of the data in the meta branch
is the separation between software policy and board support configuration. As
such, BSP descriptions should set configuration options that are related
to physical devices (drivers, driver options, flash filesystems, error
checking, etc) and leave software policy to features or kernel types.

BSPs directly include kernel types to inherit their functionality.  They
include feature and config fragments to define non-hardware configuration and
functionality. New or local configuration values introduced by a BSP should
not override non-hardware (or policy) values unless absolutely necessary, but
always should define the hardware they support.

2.5 Staged Features
-------------------

It is often desirable to manage some features independently from other
features in the tree to allow clean upstream fix integration and to avoid
managing large numbers of patches and contributions. These branches are
called "staged" features, and are included by BSP definitions by merging the
topic branch in their board description.

    git merge emgd-1.14

is an example of merging a staged emgd feature into a BSP branch via a git
operation.

2.6 Licensing
-------------
All scripts contained in this directory are under GPL-2.0-or-later license,
all kernel configuration file .cfg and .scc are under MIT license, as stated
by the SPDX-License-Identifier.

2.7 References
--------------

  git://git.yoctoproject.org/yocto-kernel-cache

