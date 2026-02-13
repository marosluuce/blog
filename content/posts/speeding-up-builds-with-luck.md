+++
draft = false
date = 2026-02-10T08:49:09-05:00
title = "Speeding Up Builds With Luck"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Sometimes you try to solve a problem and the solution's hiding in plain sight. It's easy to miss what you don't measure.

To set the scene, I was working on a digital trading card game for a large studio and my team wanted to reduce build times. The project was large, with hundreds of thousands of commits and over 60 GB of assets in git-lfs. A full pipeline build could take almost 90 minutes, with a lot of parallel work. This included many stages like building clients for Windows and mobile, building assets for each platform and version of the game, running suites of tests, packaging mobile clients, and more. At that time, the longest piece was building assets, taking around 35 minutes. This wasn't great but, thanks to parallelization, the duration was bounded by the longest asset build. We wanted it to be faster.

Getting asset builds down to 35 minutes had already taken a lot of work. Cloning or checking out a branch in the large git repository was slow and done frequently due to the parallel build structure. So the team reused the game patching tech to load the data on each build node. We managed a fleet of persistent EC2 instances allowing each action to only patch down the difference between the current build and the last run. We identified a cpu bottleneck and upgraded all the EC2 instances for more cpu power. This work helped, but building assets was still the slowest part of the build.

One day, I was investigating a build failure and remoted into a build node to read some leftover logs. I opened up My Computer and found a C drive and an unexpected D drive. The C drive was the normal ELB volume we'd used since before I joined the team, but the D drive was blank and intriguing. Turns out, when we upgraded all the EC2 instances we'd moved to a tier that included ~300 GB of SSD storage. That gave me an idea, could we go even faster if we used the SSD?

All build workspaces were created under a root folder on the C drive using custom code for deterministic names, based on target platform. It was a quick change to add an optional "useSSD" parameter to the name function. If it was true and the D drive existed, then the path constructed would be on the D drive. Otherwise it would fall back to the C drive. Testing this was also easy but time consuming. I temporarily stole a subset of build nodes, seeded the D drive with workspace data, queued up a bunch of builds, and waited.

The results were good. Build times for asset bundles averaged 27 minutes for all platforms. Importantly, building assets was no longer the slowest stage in the whole build. I submitted the change and we stopped worrying about build times for asset bundles. Now the team could focus on the new longest build stage.

After the change was merged, some other parts of the build were slightly shorter too. The workspace path code was used in almost every build stage, so they all benefited from the SSD improvement. The results in other stages weren't quite as dramatic, but I'll take free performance upgrades any day.

What strikes me most about this was how I found it by accident. Curiosity and chance led me to saving 8 minutes of build time. In hindsight, builds lacked sufficient metrics about why they were slow. No one could see the IO bottleneck. If I had the opportunity to try this again, I'd start by investing in a lot more metrics and graphs to better understand the problem. Maybe then I would have proactively made this change instead of lucking into it.
