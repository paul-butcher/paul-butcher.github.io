---
layout: post
title: Verifying Ansible Inventories
categories: ansible
---

All your roles do exactly what they are supposed to, all your vars are correct.  What could go 
wrong?  Your inventory may contain incorrect entries.

This is particularly true if the your organisation's naming convention for servers is somewhat 
arcane, making errors difficult to spot by eye.  You may have added a staging host to production or vice versa.

If your different estates have different deployment keys, then you'll thankfully fail to deploy
 to that one production server when trying to deploy to staging.  However, if it's the other way 
 around, your production deploy is going to abort half way through when it tries to modify the 
 staging one. Even if they both use the same key and the deploy is "successful", it will be incomplete, and your application may fail in 
 unexpected ways because there is no "foo" server or it's out of date, having been missed by the last deploy.

What you need is a test that knows how that naming convention works, and can tell you before you 
try to do anything.  This would preferably be run as part of your regular [check_vars]({% post_url 2017-08-29-checking-ansible-vars %}) tests - e.g. by a commit hook or CI server, ad possibly as a 
guard clause on deployment.

For this, you can use [inventory_hostname](http://docs.ansible.com/ansible/latest/playbooks_variables.html#magic-variables-and-how-to-access-information-about-other-hosts), and the [match filter](http://docs.ansible.com/ansible/latest/playbooks_tests.html#testing-strings).

Let's say you have a naming convention that contains information such as country, environment, purpose and a number that yields names like `gbrpdb001` for a production database server in the UK, or `sjmsws002` for a staging webserver in Svalbard.  It can be rather tricky to spot when `gbrpbd001` or `sjsmws002` find their way in. 

One solution that could make mistakes more apparent could be to make the naming convention a bit less dense.  However, that's not always under your control.  In any case, this is exactly the kind of job that computers are really good at.

A task such as the following will check each hostname to which it is applied, against a given pattern

```yaml
- name: fail if this is a bad host
  fail: 
    msg: This is a bad host
  when: not inventory_hostname | match (hostname_pattern)
```

At its simplest, ensuring that the 4th letter is 'p' in production hosts
would catch that the prod_webserver, below, does not belong in production.

```
[production:vars]
hostname_pattern=^\w{3}p.*$

[production:children]
prod_database
prod_webserver

[prod_database]
gbrpdb001

[prod_webserver]
gbrsws002
```
