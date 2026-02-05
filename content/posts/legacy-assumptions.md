+++
draft = false
date = 2026-02-05T15:30:24-05:00
title = "Assumptions In Legacy Code"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Legacy code can be difficult to work with for surprising reasons. It isn't always true that the code is bad; usually it does the job it was written to do. I just wasn't there when the original problem was discussed so my lack of context is a barrier I need to overcome.

Sometimes trust is the problem. Once, I joined a project and asked someone experienced to explain it to me. They showed that the code handled three things: creating an invitation, accepting an invitation, and canceling an invitation. This was an asynchronous system using Redis and Kafka, with each action broken into multiple steps. Most steps required talking to other, older systems. If something failed during accepting or canceling, a user could retry and resume from the first failed step.

This behavior matched the code I read. My mistake was not reading all of the code.

Though things usually ran smoothly, there were sometimes production issues. After a root cause was determined and resolved, there almost always followed by a phase of manual data fixes. Some invitations were stuck and required complete removal. Others were half-completed and needed the remaining steps triggered. This didn't happen frequently but after a particularly unlucky week something didn't feel right. Shouldn't users be retrying?

I went back and read the code, all the code. The retry mechanism was flawed.

At a high level, it seemed ok. When a request came in, the system would search Redis for a matching key and retrieve that data. If the data indicated the request was finished and a failure, it would re-use the found data, restarting work from the failed step. What I'd missed was buried in the Redis data retrieval. The retrieval logic filtered matching entries excluding finished ones and failures were considered finished. Thus the system assumed the request was new. It started processing from the beginning and immediately failed because the steps were not idempotent.

The fix for this was easy to implement--stop filtering out finished entries.

After deploying the fix, production issues still happened but there was no data to fix. The team was suspicious and multiple verifications proved that users were simply retrying failures on their own. I'd like to write that this solved everything and that retries were now perfect. Instead, it was the beginning of discussions about retry nuances as we discovered more edge-cases. I enjoyed those discussions.

I don't attribute the broken retries to an individual. The system was explained by someone who thought they understood it and I repeated it to other people. The problem had to be some other part of the system we all told ourselves. In long-lived systems, it's risky to base your understanding on what someone says. When something doesn't feel right, go read the code. You might be surprised (horrified?) by what you find.
