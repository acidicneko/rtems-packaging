---
layout: post
title:  "Coding period starts"
date:   2026-05-27 22:00:00 +0530
categories: jekyll update
---

After almost a month of community bonding period, coding officially starts today.
I went through my proposal once again to make sure I am in sync with what I have to 
actually build.
After going through it once, I started working on debian packaging support.
I laid most of the ground work for it during the proposal period itself.

I have started to setup and get my Ubuntu and FreeBSD VMs up and working, for
future testing and development.

Today was the weekly RTEMS meeting as well. We talked about how things are going to be
during this period. Gedare asked us to record our weekly progress on blogpost.

I also found a bug in RSB during community period. Some distributions (i.e Fedora 44 Workstation)
ship with GCC 16+. It introduces C23 and C++23 as the default language standards. 
Which alas breaks a lot of old code in a lot of packages which RSB builds.
The bug was reported at the users forum which can be viewed [here](https://users.rtems.org/t/error-while-building-rtems-tools-suite-from-source/822).
After identifying the bug, Chris introduced a new infrastructure in RSB which allowed for
setting specific language standard for any package.
This in turn solved the compilation errors being raised.

After all this bugfixing adventure, I am really excited to start working on this project officially.
