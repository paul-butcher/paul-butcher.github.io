---
layout: post
title: Ansible Smoke Testing
categories: ansible testing
---

An advantage of declarative code like (most of) Ansible, is that it is it's own test suite.  If the state of the host can't be made to match your definitions, it's a failure.  But what if your definitions are wrong?

A disadvantage is that, being its own test suite, it lacks the [double-entry bookkeeping](http://www.butunclebob.com/ArticleS.UncleBob.TheSensitivityProblem) aspect.

There are still things that could be forgotten or misconfigured when defining an environment, particularly when templating arcane configuration files.  Maybe you put an SSL certificate on the wrong box, or mistyped a port number in a firewall rule.

A lot of what might go wrong would be easily detected on the first human visit, but some things 
may not.  Maybe the firewall blocks the load balancer on one of your application servers, or 
maybe the production load balancer points to the staging estate; perhaps
half of your database cluster has split off into a cluster of its own.  Maybe you have a 
condition that stops your application being upgraded if it's already there.

One way to check this sort of thing is with smoke test plays.

Examples of things to check for:

- Firewall
    - Is everything that should be allowed, allowed?
    - Are hosts outside the environment (e.g. the machine running Ansible) forbidden from 
    accessing certain ports?
- Web application
    - Does visiting the home page work?
    - Does it contain some expected piece of text (e.g. a version id string)?
    - Do all application servers work page, does the load balancer?
    
 These tests don't prove that the application works properly and fulfils the expected 
 requirements, that's for your test suite and manual test plan to judge.  They just give a quick 
 indication that it's not completely broken.
 
 I've experimented with various ways of achieving this, with handlers and roles, and decided 
 that it is best handled by separate plays, with the test tasks defined within them.  Possibly as
  a separate playbook which can be included at the end of your main playbook (though tags work just as well).   This separation means that:
  
 - it can easily be run as part of a diagnostic effort if something goes awry independent of an 
  Ansible run;
 - the whole smoke test suite is together, so you can see at-a-glance what is being tested, and 
 in the results, what was tested.
 - tests can be added that are unrelated to specific individual roles.
 - you don't have to worry about timing and flushing handlers. By placing a test play at the end, 
 you know that everything else has already been done.
