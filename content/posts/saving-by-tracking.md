+++
draft = false
date = 2026-02-18T09:13:42-05:00
title = "Saving By Tracking"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

It's easy to over-spend if you don't track what you're spending money on. The cynic in me bets that cloud providers count on this. It's very easy to deploy ten (or a hundred) Linux instances. You can create recurring backups and spin up production-scale integration environments. Then you forget to clean up or scale back something until someone asks why the cloud bill is so high and can you please do something about it. I'd like to think I've only been responsible for saving money, but you can miss things if you aren't paying close attention.

Once, I was working with a team supporting the continued development of a digital trading card game when we received the dreaded message: our AWS bill was too high. We immediately began auditing and decommissioning. We pruned backups of our CI server, now managed by another team, that we knew we would never use. We consolidated and shrunk development Redis and Kafka clusters meant to simulate production. After all, the game had been live for over a year and we didn't need to prove we could handle the traffic. We made our service auto-scaling more aggressive and ran fewer instances when traffic was low. That wasn't all that we did, but I forget some of the specifics.

This effort added up to significant savings, though the exact number eludes me. We were confident that we could do more. Since we'd already pruned the obvious places, the next place to dig was the project itself. It was time to locate architectural inefficiencies.

I chose to look at our automated testing. We had a cool testing setup that would start a game client, connect to a game server, and play games at 10x speed. The client could be launched in a test mode, enabling an HTTP endpoint to drive the UI. This allowed python to orchestrate preset matches and make assertions about the final game state. Within the past year, we were forced to rework this setup after it fell over.

The short version is that we were trying to run too many game servers. Each test file spun up and tore down its own game server container in our Docker swarm. We ran out of resources when we reached a critical mass of concurrent builds and number of test files. The fix was to update the test helper code to only start a new game server for each game version once. Most builds only built a single game version and would now need a single game server. Larger builds would spin up five servers, one for each game version. I felt we could do better.

The game servers were game version agnostic, just like the client. Supply them with data and assets and you could play whatever you wanted. Game servers could even handle multiple game versions at once. In fact, we relied on this to minimize the number of game servers we deployed to development infrastructure. Unfortunately, the test command to load data on a game server used a hard-coded game version. This effectively limited a game server to testing a single game version.

After tracing all the calls, I threaded a new game version argument through two paths between the test helpers and game server. The first place was the code to load game data onto the server so that the data was paired with the game version. The second was the code to start a match, telling the game server which version to use.

The net effect of this change was that each build only required a single game server, regardless of game versions involved. This came at the cost of slight increase in memory usage. But it allowed the team to cut the Docker swarm from eight nodes to three. Our game server test logic now felt like it was finally operating as intended.

This work underscores two things to me. This first is that it's vital to track resource usage and compare it against costs. If you don't need something anymore, you save more money by turning it off today instead of next week or next year. The second bit is the value of understanding how your systems use resources. An opaque system may appear justified in how many resources it consumes but the only way to prove it is by digging in and validating. What lurks beneath may surprise you.
