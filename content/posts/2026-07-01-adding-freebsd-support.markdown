---
layout: post
title:  "Adding FreeBSD Packaging Support"
date:   2026-07-01 09:30:00 +0530
categories: jekyll update
---

In my last post we added a shared utility, `rtems-pkg`.

Now that I am ahead of the proposed schedule, I decided to also
add FreeBSD support as well.

## The Port system
FreeBSD port is a framework for compiling and installing third-party 
software from its original source code. Part of the FreeBSD Ports 
Collection, a port is essentially a set of instructions containing 
the source's download location, required patches, build parameters, 
and installation steps.

For our task of adding freBSD support a port was the thing we needed.

A port consists of a directory, with following files inside it:

1. `Makefile`: contains rules and steps for building
2. `pkg-descr`: the description of the package
3. `pkg-plist`: files contained in the package
4. `distinfo`: to verify integrity of downloaded sources

The next step was to write template files for these.

## Writing templates

The first step was writing the Makefile.

File: `freebsd.Makefile.in`
```
# SPDX-License-Identifier: BSD-2-Clause
#
# RTEMS Deployment FreeBSD Port Makefile template
#

PORTNAME=	rtems-@RSB_PKG_NAME@
DISTVERSION=	@RSB_VERSION@
CATEGORIES=	devel

MAINTAINER=	builder@rtems.org
COMMENT=	RTEMS tools and board support package
WWW=		https://www.rtems.org

LICENSE=	BSD2CLAUSE

PREFIX= @PREFIX@
RSB_BUILDROOT=	@RSB_BUILDROOT@
RSB_WORK_PATH=	@RSB_WORK_PATH@
RSB_SET_BUILDER= @RSB_SET_BUILDER@
RSB_SET_BUILDER_ARGS= @RSB_SET_BUILDER_ARGS@
RSB_PREFIX= @PREFIX@
RSB_TARFILE= @TARFILE@
# HOST_ARCH is handled by ports infrastructure
DISTFILES=
NO_MANCOMPRESS= yes
NO_WRKSUBDIR= yes

do-fetch:
	@${DO_NADA}

do-build:
	(cd ${RSB_WORK_PATH} && env -i PATH="${PATH}" HOME="${HOME}" \
		${RSB_SET_BUILDER} ${RSB_SET_BUILDER_ARGS})

compress-man:
	@${DO_NADA}

do-install:
	${MKDIR} ${STAGEDIR}${PREFIX}
	tar jxf ${RSB_TARFILE} -C ${STAGEDIR}${PREFIX} --strip-components=3

post-install:
	@cd ${STAGEDIR}${PREFIX} && ${FIND} . \( -type f -o -type l \) | ${SED} 's|^\./||' >> ${TMPPLIST}


.include <bsd.port.mk>
```

I faced this issue where timing for generating the plist file
was causing issues.
The compress-man step compresses all `.1` man files to `.1.gz`.
The plist was generated before `compress-man` step and during the 
packaging step the files in staging dirs were compared against plist 
entries and the man files caused mismatch which lead to errors.
I disabled the entire `compress-man` step with `@${DO_NADA}` directive.


As we don't know the files that RSB is going to generate, we leave the plist
populating step to the ports system. During the post install step we use 
the find command to dynamically search for all files, resolve links and put everything
into the TMPPLIST, which would be used in place of `pkg-plist` file.



File: `pkg-descr`
```
RTEMS tools and board support package.

This package contains development cross-compiler toolchains, compilation utilities,
and architectural libraries optimized for RTEMS runtime deployment targets.
```

File: `distinfo`
```
#as we leave source verification to the RTEMS set builder itself.
```

With these templates in place, the next step was to add target to waf 
and complete the `pkg/freebsd.py` file.

## Adding ports target to waf
The ports infrastructure need every file to be in 
a directory.
I decided to go with the following structure:
```
out/
└── amd
    └── amd-kria-k26.port 
    │   ├── Makefile
    │   ├── pkg-descr
    │   └── distinfo
    └── amd-kria-k26-next.port
    └── amd-zcu102-lwip.port
```

For each build set, we generate a `.port` directory, mirroring the
buildset file path.

```python
def ports_build(bld, build):
    bset = pkg.configs.buildset(bld, build, dry_run=False)
    pkg_name = bset['name']
    buildroot = ""
    board_path = _esc_name(build['buildset'])
    port_dir = board_path + '.port'

    if bld.env.RSB_RELEASED:
        rel = 'released'
    else:
        rel = 'not-released'

    subst_vars = {
        'RSB_BUILDROOT': '',
        'RSB_PKG_NAME': bset['name'],
        'PREFIX': bld.env.PREFIX,
        'RSB_VERSION': bld.env.RSB_VERSION,
        'RSB_REVISION': _esc_label(bld.env.RSB_REVISION),
        'RSB_RELEASED': rel,
        'TARFILE': bset['tar'],
        'RSB_SET_BUILDER': bset['cmd'],
        'RSB_SET_BUILDER_ARGS': ' '.join(bset['pkg-opts']),
        'RSB_WORK_PATH': bld.path.abspath(),
    }
    bld(name='port_makefile_' + bset['name'],
        features='subst',
        description='Generate port Makefile',
        target=bld.path.get_bld().find_or_declare(port_dir + '/Makefile'),
        source='pkg/freebsd.Makefile.in',
        chmod=0o755,
        **subst_vars)
    bld(name='port_distinfo_' + bset['name'],
        features='subst',
        description='Generate port distinfo file',
        target=bld.path.get_bld().find_or_declare(port_dir + '/distinfo'),
        source='pkg/freebsd.distinfo.in',
        **subst_vars)
    bld(name='port_pkg-descr_' + bset['name'],
        features='subst',
        description='Generate port pkg-descr file',
        target=bld.path.get_bld().find_or_declare(port_dir + '/pkg-descr'),
        source='pkg/freebsd.pkg-descr.in',
        **subst_vars)
```

Note that we explicitly set `0o755` flag to Makefile to mark it as executable.

We also add the register the `ports` as a valid target for waf.
```python
@TaskGen.feature('port')
class porter(Build.BuildContext):
    '''generate port files'''
    cmd = 'ports'
    fun = 'ports'

def ports(bld):
    for build in pkg.configs.find_buildsets(bld):
        ports_build(bld, build)
```

With this in place we can now just run `./waf ports` and it will generate 
`.port` dir for all valid buildsets.

Then to build the `.pkg` file we just cd into the required `.port` dir and
run `make package` there.

```
cd out/amd/amd-kria-k26
make package
```

## Registering freeBSD support into rtems-pkg
Now that we have our basic support added into waf, it's time to 
register it in `rtems-pkg`.

```python
class PORTS(Packager):

    def __init__(self):
        super().__init__("ports")

    def get_build_config(self, target: str) -> dict:
        port_path = os.path.join('out', f"{target}.port")
        return {
            'path': port_path,
            'cmd': ['make', 'package'],
            'cwd': port_path
        }
```
Due to the modular nature of `rtems-pkg`, adding a new packager is pretty easy.
We make a new `PORTS` class. And register it to the `PackagerFactory`.

```
class PackagerFactory:
    _packagers = {
        # format_type: (waf_feature, packager_class)
        'deb': ["deb", DEB],
        'rpm': ["rpmspec", RPM],
        'ports': ["ports", PORTS]
    }
```

Now building a freeBSD package is as easy as:
```
./rtems-pkg --packager ports --target amd/amd-kria-k26
```

## Summary
Till now we have added packaging support for both debian and freeBSD.
Solved bugs in the existing rpm pipeline. Introduced a shared abstraction layer
to make it easy for users.

Most of our end term goals are completed. Only adding tutorial on how
to add a new packager is left.

Here is the [merege request](https://gitlab.rtems.org/rtems/tools/rtems-deployment/-/merge_requests/37).

I would be talking to my mentor Chris to check what additional tasks I can take upon.
Untill then sayounara!!


