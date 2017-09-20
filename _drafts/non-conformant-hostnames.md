---
layout: post
title: non-conformant hostnames
---

[verifying ansible inventories]({% post_url 2017-09-05-verifying-ansible-inventories %}), shows how 
to use the match filter to ensure that your hosts conform to the naming convention expected of them.
However, not all hosts in your inventory will necessarily conform to the same convention.

This isn't a problem, as you can simply define a different hostname_pattern for those hosts.  
However, if there are only a few, then it may be better just to list out the non-conformists.
Doing so in a separate list (outside of the inventory )helps with the double-entry-bookkeeping 
aspect of testing, ensuring that you have to make the same mistake twice for it to slip through 
the net.