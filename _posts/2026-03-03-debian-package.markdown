---
layout: post
title:  "Debian Packaging Support"
date:   2026-03-03 12:13:01 +0530
categories: jekyll update
---

I have been working on adding `.deb` package support to RTEMS Deployment tool.
I have had some initial success doing so.

After carefully studying how `.rpm` packages were being built, I found the following:

- `./waf --target=targets-required` built `.tar` packages with compiled binaries and place it in `tar/` directory.
- `./waf rpmspec` generated `rpmspec` files for each of the supported hardwares using `pkg/rpm.spec.in` and stored them into `out/hardware-name` directories.
- User then manually invoked the command `rpmbuild -bb out/required/hardware/hardware-name.spec`, which built the `.rpm` package in `out/buildroot` directory.

I have tried to mimick the exact same workflow for debian packages.

First I have added required functions in `pkg/linux.py` to add `debspec` command so that we can do `./waf debspec` and generate control file for all the supported hardwares same as rpmspec files.

Made a template `debian.control.in` file:
```
Source: rtems-@RSB_PKG_NAME@
Section: utils
Priority: optional
Maintainer: RTEMS Deployment <rtems@rtems.org>
Standards-Version: @RSB_VERSION@
Build-Depends: debhelper (>= 13)

Package: rtems-@RSB_PKG_NAME@
Version: @RSB_VERSION@
Section: utils
Priority: optional
Architecture: all
Maintainer: RTEMS Deployment <rtems@rtems.org>
Description: RTEMS tools and board support package
 This package contains development tools and libraries for RTEMS.

X-RSB-TAR: @TARFILE@
X-RSB_BUILDROOT : @RSB_BUILDROOT@

@USER_DEB_CONFIG@

```

Also a `debian.rules.in` file:
```
#!/usr/bin/make -f
%:
	dh $@

PACKAGE := $(shell awk '/^Package:/ {print $$2; exit}' debian/control)
TARFILE ?= $(shell awk '/^X-RSB-TAR:/ {print $$2; exit}' debian/control)
BUILDROOT ?= $(shell awk '/^X-RSB_BUILDROOT:/ {print $$2; exit}' debian/control)

override_dh_auto_install:
	@mkdir -p debian/$(PACKAGE)
	# fail early if TARFILE not specified
	@test -n "$(TARFILE)" || (echo "TARFILE not set (provide TARFILE env or X-RSB-TAR in debian/control)" >&2; false)
	@tar xjf $(TARFILE) -C $(BUILDROOT) --strip-components=1

```
This helps the `dpkg-buildpackage` setup the buildroot tree by extracting required tar into it.

A typical flow looks like the following:
```bash
cd out/deb_buildroot
mkdir -p debian
cp ../required/hardware/name.control debian/control
cp ../../pkg/debian.rules.in debian/rules
cp ../../pkg/debian.changelog.in debian/changelog
chmod +x debian/rules
fakeroot dpkg-buildpackage -us -uc -b
```
These steps will produce a `.deb` package in `out/` directory.
