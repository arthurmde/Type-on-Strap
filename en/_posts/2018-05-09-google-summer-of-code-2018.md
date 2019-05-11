---
layout: post
title: "Google Summer of Code 2018"
feature-img: "assets/img/pexels/debian-screen.png"
thumbnail: "assets/img/thumbnails/debian-screen.png"
lang-ref: gsoc-2018-1
lang: en
categories:
  - Hacking
  - FLOSS
tags:
  - gsoc
  - debian
  - floss
excerpt_separator: <!--more-->
---

During my graduation period, I became a free software enthusiast and contributor.
My early involvement with free software communities contributed significantly to
my education, both in the technical and social aspects, as well as bringing me
many opportunities as a Software Engineer. Not by chance, I have been applied
the knowledge acquired over the years with free software throughout my
master's degree.

Now, I have a great opportunity to expand my participation in the free
software world because [my proposal](https://summerofcode.withgoogle.com/projects/#5631070255448064)
to work on [Debian](https://www.debian.org/) project
was accepted in [Google Summer of Code 2018](https://summerofcode.withgoogle.com/) - **GSoC**.
<!--more-->
My project aims at designing and implementing new features in 
[Distro Tracker](http://tracker.debian.org/) to better support Debian teams to track
the health of their packages and to prioritize their work efforts.
[Lucas Kanashiro](http://blog.kanashiro.xyz/) is my mentor on this project.


This means that I will spend my winter (I live in Brazil =D) hacking to
solve issues and to contribute to an important
free software that is vastly used by both Debian's developers and users,
interacting with the amazing community of Debian, and learning a lot
technically. Besides, this is my big chance to start blogging for real.
Although I have been thinking about doing this for a long time, participating
in the GSoC was what really made me to get this blogging space off the drawing
board. So I will be using this space constantly to report on the progress being
made during the Summer of Code, as well as to share various ideas and discussions that
emerge.

## Project Details

Distro Tracker concentrates several useful features related to Debian packages,
allowing teams to group it packages of interest and to receive update
notifications from it. Although Distro Tracker provides a comprehensive page
with detailed data for each package, it lacks features for helping packaging
teams to have an overview of their packages.

Currently, the [PET (Package Entropy Tracker)](https://pet.debian.net)
project implements useful features for packaging teams tracking the status of
their packages, providing useful information about upstream projects, and
supporting QA tasks by highlighting packages with bugs, missing tags, and
outdated. However, PET is no longer active and is obsolete as it does not
integrate the data from [Salsa](https://salsa.debian.org/public) development server,
which is a [Gitlab](http://gitlab.com/) instance of the Debian project.

Therefore, the idea of this project is to migrate the most important
team-related features from PET to Distro Tracker, leveraging and improving
Distro Tracker current code base regarding teams.  Thus, as a final result of
SoC, I expect to incorporate to Distro Tracker a set of useful data to help
teams to see the health of multiple packages and better prioritize their efforts
where it is most needed. It is worthing noticing that Distro Tracker is a
general purpose service that is also used by 
[Kali community](https://pkg.kali.org/). Thus, they also
will be able to take advantage of the proposed improvements.

## Why Debian?

I have been using Debian for years. I admire the project for its technical
quality, for being a project that prioritizes its users, and for the philosophy
and organization of the project community. I am very keen to contribute back
to the project actively, I believe that there are several opportunities to apply
my technical skills to assist the project in multiple aspects, but I also
see it as a precious space of much learning and knowledge. Moreover,
during the latest years, I had the opportunity to work with some Debian Developers
and other official Debian members that are people I truly admire. Interacting
and working together consistently with them are great motivators to
contribute to Debian as well.

## What Else?
 
I would like to thank Lucas Kanashiro and 
[Raphael Hertzog](https://raphaelhertzog.com/)
for mentoring me and revising my Merge Requests so far. Now, we are planning and
discussing the initial steps towards integrating data from [Salsa](https://salsa.debian.org/public)
into Distro Tracker.

Finally, I am extremely happy to congratulate my Brazilian friends
[Athos Ribeiro](https://summerofcode.withgoogle.com/projects/#6144149196111872),
[Rodrigo Siqueira](https://summerofcode.withgoogle.com/projects/#5821994302439424),
and [Tallys Martins](https://summerofcode.withgoogle.com/projects/#5618610421104640)
for getting their projects accepted in GSoC as well.

**I'm really excited for SoC and I hope to make good contributions =)**
