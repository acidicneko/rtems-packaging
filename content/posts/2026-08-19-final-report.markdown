---
layout: post
title:  "Final Report"
date:   2026-08-19 10:37:01 +0530
categories: jekyll update
---

## About me
Hey everyone! I’m Ayush Yadav, aka `acidicneko`, from India.
I’m currently a pre-final-year student at the Indian Institute of Technology Jammu,
majoring in Materials Science and Engineering.
As much as I enjoy spending my time working with Fourier transforms of crystal
lattice diffraction patterns, I also love programming and building things with code.

## Project Summary
Over the last two months, I worked on adding packaging support for Debian
and FreeBSD, as proposed in my original proposal. However, the scope of
the project grew considerably in the second half of the program. Most of
the deliverables promised in the proposal were completed even before the
midterm evaluation. After talking with Chris, the scope was later expanded
to replace `waf` with a custom tool, `rtems-pkg`, which now acts as the
central API for Deployment.

## Code written during coding period
Relevant links for the midterm evaluation are below:
- [Merge Request !35](https://gitlab.rtems.org/rtems/tools/rtems-deployment/-/merge_requests/35) — fixing build environment pollution
- [Merge Request !36](https://gitlab.rtems.org/rtems/tools/rtems-deployment/-/merge_requests/36) — adding Debian support
- [Merge Request !37](https://gitlab.rtems.org/rtems/tools/rtems-deployment/-/merge_requests/37) — adding FreeBSD support

Relevant links for the end-term evaluation are below:
- [Merge Request !38](https://gitlab.rtems.org/rtems/tools/rtems-deployment/-/merge_requests/38) — replacing `waf` with `rtems-pkg`


Other Merge Requests:
- [Merge Request !32](https://gitlab.rtems.org/rtems/tools/rtems-deployment/-/merge_requests/32) — adding HOST_ARCH to rpmbuild
- [Merge Request !34](https://gitlab.rtems.org/rtems/tools/rtems-deployment/-/merge_requests/34) — handling unknown arch

## Goals
### Midterm Goals: Adding Debian and FreeBSD Packaging Support

Although my proposed timeline listed FreeBSD support as an end-term
deliverable, I was able to complete it relatively easily, well before the
midterm evaluation.

A detailed walkthrough of the entire process has been documented on this
blog. It involved writing new template files, researching packaging
metadata, and fixing bugs introduced along the way.

The biggest challenge I faced during this period was injecting
distro-specific flags into the build environment. My mentor, Chris,
helped me a great deal in getting past this hurdle.

A new command-line utility, `rtems-pkg`, was also added to Deployment to
simplify the packaging process. It started out as a simple wrapper but
eventually grew into something much bigger.


### End-Term Goals: Replacing `waf` with `rtems-pkg`

This wasn't originally planned, but it happened anyway. After finishing
all my midterm and end-term goals early, my mentor asked whether it would
be possible to develop `rtems-pkg` further and swap out `waf` entirely.
After carefully researching what `waf` was doing under the hood, I
confirmed it was possible.

Going forward, `rtems-pkg` serves as the central API for RTEMS Deployment.
It provides a single CLI to package, generate package metadata files, and
build different BSPs, considerably simplifying the user experience.

Previously, the `waf` workflow required manually invoking a
packager-specific command to build the package after `waf` had generated
the package metadata files from templates. `rtems-pkg` combines these two
separate steps into one. Now, a user can simply provide the list of
targets to build, the packager, the install prefix, and any additional
build options — `rtems-pkg` takes care of generating the metadata files,
invoking the appropriate packager binary, and producing the final package,
all on its own.


## What was done?
- Debian and FreeBSD packaging support was successfully added to
  Deployment (as stated in the proposal).
- `waf` was completely replaced by `rtems-pkg` (extended scope).

## What wasn't? Future Scope?
Though I achieved all the proposed goals, there are still a few gaps in
the current `rtems-pkg` implementation.
One being the user configuration stuff. With the `waf` approach it was
possible to give custom config and change a few fields. Though totally
possible even with the new `rtems-pkg`, it hasn't been done due to time
constraints.
It would be nice to support user supplied configuration.

Other than that, it would be nice to have more packagers in `rtems-pkg`.
A tutorial on adding new packagers was added to the deployment repository.
Currently supported packagers are:
- FreeBSD
- Debian
- RPM

Extending the list would be pretty awesome!

## Other useful links
Discourse threads:
- [Discussing issue #82](https://users.rtems.org/t/discussing-issue-82-add-packaging-options-to-rtems-deployment/577)
- [RSB fails on FreeBSD 15.1](https://users.rtems.org/t/rsb-fails-on-freebsd-15-1/864)
- [RSB failed during RPM Build on Fedora 44](https://users.rtems.org/t/rsb-fails-during-rpm-build-on-fedora-44/835/25)
- [Error while building RTEMS Tools suite from source](https://users.rtems.org/t/error-while-building-rtems-tools-suite-from-source/822)
- [GSOC 2026 - Intro : Packaging options for RTEMS Deployment](https://users.rtems.org/t/gsoc-2026-intro-packaging-options-for-rtems-deployment/549)

## Final Words

Getting selected for Google Summer of Code 2026 was unexpected. I was
genuinely surprised. It's been a wonderful experience to work on something
real, something that will actually be used by real people. I've always
been fascinated by open source, and I'm grateful I got to give something
back to it. Working with RTEMS has been an excellent experience overall.

I'd like to thank my mentor, Chris Johns. Even though he was strict at
times, and scolded me more than once, and pushed me to rethink my
implementations, I learned a great deal from him. He always nudged me
back in the right direction whenever I drifted off course.

I'd also like to thank Gedare, who always listened patiently during our
weekly meetings. Those meetings felt warm and welcoming. I was nervous to
speak up at first and probably fumbled through my updates, but he always
gave me the time and space to collect my thoughts and share them.

I'd definitely recommend that any student reading this blog consider
contributing to RTEMS.
