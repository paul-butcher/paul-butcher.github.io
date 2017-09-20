---
layout: post
title: Verifying Ansible inventories part two
---

You can build up the pattern for verifying a naming convention, using fragments specified in group_vars.

In [verifying ansible inventories]({% post_url 2017-09-05-verifying-ansible-inventories %}),
I showed a way to help ensure that a given host conforms to your naming convention.  I gave 
a simple example, whereby a dense naming convention has a 'p' as the fourth character to denote
a production host.  Checking for that character helps ensure that you don't accidentally include
staging hosts in your production inventory (or the other way around!)

There are several nuggets of information that can be encoded in a hostname.  
See [Picking Server Hostnames](https://blog.serverdensity.com/picking-server-hostnames/)
for an example.

All of these nuggets can be encoded and checked, to make sure that all of the members of your groups 
actually belong there.

The example I will use for this post encodes location (country), environment, purpose and an identifier
for the individual host, thus: froproddb01.  This is production database number 1 in the Faroe Islands.

As in the previous post, we have a task that fails if the host doesn't match a given pattern
```yaml
- name: fail if this is a bad host
  fail: 
    msg: This is a bad host
  when: not inventory_hostname | match (hostname_pattern)
```

In group_vars/all, we build hostname_pattern
```yaml
hostname_pattern: "^{{ location }}{{ environment }}{{ purpose }}\d\d$"
```

Then in the vars files for the various groups:

faroes.yml
```yaml
location: fro
```

production.yml
```yaml
environment: prod
```

database.yml
```yaml
purpose: db
```

And so on.  A production web server in the same place, "froprodws01", can be validated by adding
a purpose rule for webservers.

webserver.yml
```yaml
purpose: ws
```
