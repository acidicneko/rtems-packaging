---
layout: post
title:  "RPMBuild Failure"
date:   2026-06-12 22:00:00 +0530
categories: jekyll update
---

Ladies and gentlemen. Turns out there were a few bugs in the existing `rpmbuild`
packaging pipeline.

On 6th of June, I decided to test out the existing rpmbuild packaging on my
freshly installed Fedora 44 system.
I `cd`'d into rtems-deployment directory. I ran `./waf rpmspec` to generate spec files
from the provided template i.e `pkg/rpm.spec.in`. Waf successfully generated those in the
`out/` directory.
After generating spec files, I decided to build for the `amd/amd-kria-k26` board.
The command to build a rpm package from spec file is:
```
rpmbuild -bb out/amd/amd-kria-k26.spec
```
The build usually takes around 30 minutes on my hardware so I decided to grab a cup
of coffee while my machine was compiling all the different packages.

I returned after a while only to see:
```
make[4]: Entering directory '/home/me/GSOC_Work/rtems-deployment/build/aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1/build/isl/doc'
make[4]: Nothing to be done for 'all'.
make[4]: Leaving directory '/home/me/GSOC_Work/rtems-deployment/build/aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1/build/isl/doc'
make[3]: Leaving directory '/home/me/GSOC_Work/rtems-deployment/build/aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1/build/isl'
make[2]: Leaving directory '/home/me/GSOC_Work/rtems-deployment/build/aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1/build/isl'
make[1]: Leaving directory '/home/me/GSOC_Work/rtems-deployment/build/aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1/build'
make: *** [Makefile:1064: all] Error 2
make: Leaving directory '/home/me/GSOC_Work/rtems-deployment/build/aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1/build'
shell cmd failed: /bin/sh -ex  /home/me/GSOC_Work/rtems-deployment/build/aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1/do-build
error: building aarch64-rtems7-gcc-15.2.0-newlib-a7c61498-x86_64-linux-gnu-1
```

The build failed for some reason.

I decided to investigate the said log file to find out the reason.
I found the main source of error:
```
../../../gcc-15.2.0/libcpp/expr.cc:882:35: error: format not a string literal and no format arguments [-Werror=format-security]
  882 |             cpp_warning_with_line (pfile, CPP_W_LONG_LONG, virtual_location,
      |             ~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  883 |                                    0, message);
      |                                    ~~~~~~~~~~~
../../../gcc-15.2.0/libcpp/expr.cc:885:38: error: format not a string literal and no format arguments [-Werror=format-security]
  885 |             cpp_pedwarning_with_line (pfile, CPP_W_LONG_LONG,
      |             ~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~
  886 |                                       virtual_location, 0, message);
      |                                       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
../../../gcc-15.2.0/libcpp/expr.cc:896:33: error: format not a string literal and no format arguments [-Werror=format-security]
  896 |           cpp_warning_with_line (pfile, CPP_W_SIZE_T_LITERALS,
      |           ~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  897 |                                  virtual_location, 0, message);
      |                                  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Turns out there were some warnings being treated as errors which in turn failed the whole compilation
process.

That was somewhat unexpected to me. During the proposal period I had definitely built the tar archive for this board,
and it worked at the time. What could possibly cause it to fail now?
I tried building it using `waf`; like I built it during proposal period.
With the command `./waf --target=amd/amd-kria-k26`. 
Again I left it to compile and returned after 30 minutes. A good reading session of "A Knight of Seven Kingdoms" helped a lot :)
To my surprise this time it built the archive successfully.
That was a bit odd. I tried to extract how `rsb` was being called both times, arguments it ran with, etc.

I checked the waf log for the command line and I found this: 
```
Command Line: /home/me/Code/RTEMS/src/rsb/source-builder/sb-set-builder --prefix=/opt/rtems/7 \
                --bset-tar-file --trace --log=out/amd/amd-kria-k26.txt \
                --no-install amd/amd-kria-k26
```

I checked the error log to see the command line and guess what it was the SAME!
How could that possibly be right? The same command line yielding two very different results!

My first instinct was to contact my friendly neighbourhood (6700 miles away btw) mentor Chris. After informing him 
about this anamoly, I created a topic on [users forum](users.rtems.org).

I suspected something was changing the build environment. After Chris confirmed he felt the same,
I started to dig more into it.

The first hint was `-Werror=format-security`. I ran a fuzzy search on the `waf` build log using
the best editor in the world neovim (sorry VSCode). I found no occurrence of it.
Something was injecting this flag into build environment to use.

I also tried invoking `rsb` with same command line from debian rule file as well and it yielded same errors.

Clearly both the build environments were being infected with extraneous build flags.

A quick documentation probing revealed that `rpmbuild` injects flags via environment variables
in the build environment. These flags are set by the distribution, which was Fedora in my case.


Running `rpmbuild --eval '%{optflags}'` revealed all the default build Optional flags injected by rpmbuild.
```
-O2 -flto=auto -ffat-lto-objects -fexceptions -g -grecord-gcc-switches -pipe -Wall -Wno-complain-wrong-lang
-Werror=format-security -Wp,-U_FORTIFY_SOURCE,-D_FORTIFY_SOURCE=3 -Wp,-D_GLIBCXX_ASSERTIONS 
-specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -fstack-protector-strong 
-specs=/usr/lib/rpm/redhat/redhat-annobin-cc1  -m64 -march=x86-64 -mtune=generic -fasynchronous-unwind-tables 
-fstack-clash-protection -fcf-protection -mtls-dialect=gnu2 -fno-omit-frame-pointer -mno-omit-leaf-frame-pointer
```
There it was!

My first instinct was to clear `RPM_OPT_FLAGS` which resulted into the build process failing with errors related
to `-fPIE`. Clearly clearing just `RPM_OPT_FLAGS` wasn't enough. Or maybe I cleared some important flag?

My next step was to just remove the flag affecting the build i.e `-Werror=format-security`.

I decided to strip it by:
```
%build
export RPM_OPT_FLAGS=$(echo "$RPM_OPT_FLAGS" | sed -e 's/-Werror=format-security //')
```

Surely that would solve the issue, right? RIGHT?

No!
The compilation did move forward but again was interrupted at another step.

Turns out there was another flag which was causing issues.
RPMBuild's was pushing build flags too hard.
RSB first builds the cross compiler which runs on the host machine.
The cross compiler then builds programs which would run on the target board.
RPMBuild set `CC=gcc` and `CXX=g++` which confused RSB and forced it to use host's compiler
for compiling another architecture's code with the flags meant for cross-compiled GCC.
`-march=x86-64` and `-mtune=generic` were also enabled by rpmbuild.

This lead to the following errors:
```
cc1: error: unrecognized command-line option ‘-mno-outline-atomics’
cc1: error: unrecognized command-line option ‘-mfix-cortex-a53-835769’
cc1: error: unrecognized command-line option ‘-mfix-cortex-a53-843419’
cc1: error: bad value ‘cortex-a53’ for ‘-mtune=’ switch
cc1: note: valid arguments to ‘-mtune=’ switch are: nocona core2 nehalem corei7 westmere sandybridge corei7-avx ivybridge core-avx-i haswell core-avx2 broadwell skylake skylake-avx512 cannonlake icelake-client rocketlake icelake-server cascadelake tigerlake cooperlake sapphirerapids emeraldrapids alderlake raptorlake meteorlake graniterapids graniterapids-d arrowlake arrowlake-s lunarlake pantherlake diamondrapids wildcatlake novalake bonnell atom silvermont slm goldmont goldmont-plus tremont gracemont sierraforest grandridge clearwaterforest intel x86-64 eden-x2 nano nano-1000 nano-2000 nano-3000 nano-x2 eden-x4 nano-x4 lujiazui yongfeng shijidadao k8 k8-sse3 opteron opteron-sse3 athlon64 athlon64-sse3 athlon-fx amdfam10 barcelona bdver1 bdver2 bdver3 bdver4 znver1 znver2 znver3 znver4 znver5 znver6 btver1 btver2 generic native
```

The obvious next step was to clear the `CC` and `CXX` flags as well. And so I did.
However even after doing that some of the code meant for target board's architecture was being compiled
with the `march=x86-64` flag.

After consulting with Chris, I tried to unset all the related build environment variables.
Chris also suggested reviewing the output of `rpmbuild --showrc`. This revealed an important macro
`_auto_set_build_flags`. A quick search on internet revealed that this flag control whether or not 
rpmbuild injects Distro-default build flags into the environment. Combined this with unsetting all the
build flags before invoking `rsb` actually solved the issue.
```
...
%global _auto_set_build_flags 0
...
%build
unset CC
unset CXX
unset CFLAGS
unset CXXFLAGS
unset CPPFLAGS
unset LDFLAGS
unset RPM_OPT_FLAGS
unset FFLAGS
unset FCFLAGS
unset AR
unset AS
unset LD

# The RSB deployment build command
cd %{rsb_work_path}
%{rsb_set_builder} %{rsb_set_builder_args}
...
```

This wasn't the end though. After fixing all the cross-build environment related error, 
another error occurred.
```
lto-wrapper: fatal error: write: Disk quota exceeded
compilation terminated.
/usr/bin/ld.bfd: error: lto-wrapper failed
collect2: error: ld returned 1 exit status
make[2]: *** [../../gcc-15.2.0/gcc/cp/Make-lang.in:145: cc1plus] Error 1
```

Link-time-optimization (lto) defers optimization to linker.
When GCC compiles with `-flto`, it defers optimization to link time. A backend tool called lto-wrapper 
then analyzes the entire codebase at once and streams several gigabytes of intermediate data to `TMPDIR` while doing it.
On Fedora (and most modern systemd-based distros), 

`/tmp` is mounted on `tmpfs` which is a RAM-backed filesystem
with a kernel-enforced size quota, which in my case was around 6 GB.
```
❯ quota -s
Disk quotas for user me (uid 1000): 
     Filesystem   space   quota   limit   grace   files   quota   limit   grace
          tmpfs     88K   6275M   6275M               2       0       0        
          tmpfs    888M   6275M   6275M             522       0       0
```
That quota exists to prevent runaway processes from exhausting host memory. 

On some systems though(i.e Debian based) `/tmp` points to disk-storage.

This issue was easily solved by temporarily changing `/tmp` dir by modifying
`TMP`, `TMPDIR` and `TEMP` variables.

My initial thought was to modify these vars in the spec file itself before calling `rsb`.
```
export TMPDIR=%{rsb_work_path}/build/tmp
export TMP=%{rsb_work_path}/build/tmp
export TEMP=%{rsb_work_path}/build/tmp
```

But Chris suggested this should be left to the host system admin; and warning the
users about this potential issue via documentation would suffice.

Regardless of the method, pointing `/tmp` to disk-storage solved the issue.


The code compiled successfully and it was able to build the tar package and in turn the RPM package.
Here is the build [log](https://gist.github.com/acidicneko/72d8f47306c866d963e28b6a65ddc18c).

![RPM Package](https://raw.githubusercontent.com/acidicneko/rtems-packaging/refs/heads/main/assets/images/term_rpm_pkg.png)


Here is the entire users forum [thread](https://users.rtems.org/t/rsb-fails-during-rpm-build-on-fedora-44/835).
