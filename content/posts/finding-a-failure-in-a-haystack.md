+++
draft = true
date = 2026-02-27T14:40:09-05:00
title = "Finding A Failure In A Haystack"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Picture a Jenkins build pipeline.

It has seven stages, each with up to a dozen or so steps running in parallel. All of these steps run on different build nodes and call out to tools in Python and C#. Jenkins produces logs, as do these tools, and by the end there are hundreds of thousands log lines across dozens of files.

Now imagine you're an developer for a card game and you're updating the art for a card. Everything looks good locally. You commit your changes and trigger a build. Forty minutes later, you receive a Slack message that Jenkins failed while building assets. The message contains a generic error that assets did not build properly with a link to the build and logs. This isn't your first build failure and you message the build team for help, instead of digging through a mountain of logs.

Several years ago, I found myself on a build team responding to frequent queries as to why a build failed. Over time I'd looked up enough failures that I could scan through the logs pretty quickly. I'd gained an intuition for which file contained the relevant information for a given failure. Being able to quickly unblock other developers was a useful skill but it wasn't fun work. As a team, we reviewed our recent help requests and the dominant reason was unclear build failures. We decided something needed to change.

There was a lot to like about our current setup. The existing Slack messages included helpful failure details like build node, branch, and commit. They were sent to the branch owner and a shared builds channel. The unclear part was the message itself, lacking actionable feedback. We brainstormed ways to make failures more self-serviceable and reduce our own toil. One idea was a scanner to look through the logs and find relevant output. We decided against this because a team member had attempted this in the past and deemed it expensive to build and fiddly to get right.

We reviewed our tools on hand and concluded the problem wasn't knowing or finding the failure but extracting it. The build system and associated scripts were a combination of Groovy (Jenkins), Python (build/deploy scripts and tests), and C# (Unity build logic) that grew organically over many years. One aspect common to all three was that failures were driven by exceptions. Many failures used custom exceptions, containing specific details about the root cause. The trick was convincing Groovy to essentially re-throw a C# or Python exception.

One team member hit on the idea of storing a failure as a json file. If a tool call in C# threw an exception, it would write all that data to a json file (including the detailed error message). Then failure handling in Jenkins would verify whether a failure file existed. When it did, the system would throw a new exception with the specifics from the file. Otherwise, it would do a best effort job, indicating the failed stage with a generic message. We wrote this failure file exception logic three times, once for each language, and it was tremendously helpful.

In C#, we implemented this as a try-catch around the main tool entry point, only writing the failure file for calls within Jenkins. This was minimally invasive, allowing the existing custom exceptions to continue to transport failure details. Python was a little hairier because there wasn't a single entry-point due to the many scripts involved. We updated all the explicit failures with new exceptions and better messages. For the rest, we were thankful Python provides `sys.excepthook`, allowing a global error handler to catch the remaining ways things could fail.

Jenkins and Groovy were their own problem. All the failure coordination logic was written in Groovy. It captured exceptions, looked for failure files, and sent Slack messages. Catching exceptions properly took several tries to get right. A blanket try-catch for all exceptions turned out to be a mistake because it interfered with the control flow in Jenkins, making a failed build look like a success. This necessitated a custom job failure exception. If the caught exception was a job failure, it was safe to assume it was already handled and to allow it to bubble up.

This work dramatically clarified error messages. Compilation failures included the problematic files and line numbers. Asset builds listed the offending assets. Test failures printed a summary of failed tests, including a link to specific archived reports. Most importantly the number of times someone asked me to to comb through logs dropped dramatically. It wasn't zero, but it only happened when something new went wrong. In addition, resolving a build failure now included making improvements to the error message with the intent that the failure would be self-serviceable next time.

A surprise benefit to this overhaul was finally being able to turn on fast fail in Jenkins. When fast fail is off, parallel work must wait until all parallel actions finish before reporting a failure. We'd been unable to turn this on previously because our failure handling couldn't distinguish a legitimate failure from Jenkins terminating a process when something else failed. The last time we tried turning this on had result in confusion over where failures occurred. Now that we could safely turn on fast failure, it significantly improved feedback time for failed builds and freed up build nodes faster.

This effort highlighted the value of tracking the kinds of requests we got from other teams. The repetition of log searching started as a sense that something was wrong. Data turned it into a clear target for improvement. It allowed us to carve out time to make the necessary changes and ultimately save everyone time.

Clear error messages aren't glamorous or highly visible but they're valuable. Generally, you don't hear about it when things are running smoothly. Developers do notice when an error message is bad.
