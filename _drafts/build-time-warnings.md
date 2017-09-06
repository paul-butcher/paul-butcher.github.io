---
layout: post
title: Build time warnings
---

I recently had a problem whereby a Jenkins job had been doing nothing for a few runs.  It was 
recording successes, but that's only because it was very successfully doing nothing.  Since there
 was a massive difference between the duration of a real successful run, and the NOOP successful 
 runs, it was easy to spot when I looked at it as a human.

It was a job that deployed an application to a test environment.  The problem was caught when 
someone found that they could not deploy a special version to the environment via their own 
command line, and although Jenkins was running successfully, the expected version wasn't there.  

Because the job didn't report on any test outcomes, it didn't break on discovering an out of date 
test report file or anything like that.

A plug in thar can fail or warn a bid if it's duration is longer or shorter than expected.

Pipeline stage 
Different to previous, or mean of previous n.
