---
layout: post
title:  Keep Deployment Keys out of the Codebase.
categories: ansible
---

This may sound like obvious advice, but with [vaults](http://docs.ansible.com/ansible/latest/playbooks_vault.html), [Ansible](https://www.ansible.com/) provides an excellent system to allow you to keep sensitive data encrypted in your repository, so that they can be versioned alongside the rest of your definitons, and you don't have to find somewhere else to put them.

Ansible also makes it easy to run a playbook without requiring a user to authenticate on the remote host, by using the 
[behavioural inventory parameters](http://docs.ansible.com/ansible/latest/intro_inventory.html#list-of-behavioral-inventory-parameters) (`ansible_user`, `ansible_ssh_pass` and `ansible_ssh_private_key_file`).

As such, it is tempting to put these authentication parameters into a vault, so that your deployment process is silky smooth.  Unfortunately, doing so also bypasses any protections you may have that ensure that your Ansible code is reviewed and well tested, or that the software you are deploying has transitioned through whatever release process you have in place.

You can't protect your estate against someone who, in order to do their job, has the keys.  So you need to ensure that those keys are only needed (and, indeed available) for the final deployment push.

Those who are entrusted with the keys are obviously deemed trustworthy enough not to mount a malicious attack, and probably sensible enough not to make major mistakes every day.  But everyone makes mistakes, and some may be more serious than others.

Anyone working on your Ansible codebase needs some access to the vaults in order to modify and test definitions. If the deployment credentials are in your codebase, then those people are permanently only a few keystrokes away from making an unplanned and possibly broken deployment to your production estate.

Back in the olden days, in order to accidentally break your production estate, you would probably have log on to each machine individually and do it one machine at a time, with plenty of opportunities to realise that you're in the wrong place.  Hopefully, you would spot what you had done pretty quickly, and unless you've done something particularly knotty, you would probably be able to put it right without too much trouble.

As the saying goes: [To Err is Human; To Really Foul Things Up Requires a Computer](https://quoteinvestigator.com/2010/12/07/foul-computer/).  With IT Automation tools like Ansible, you can efficiently make a pretty unholy mess of your entire estate with just one mistake.

So, reserve vault files for secrets that are internal to the environment - database passwords, API keys and the like - Keep your deployment credentials elsewhere (e.g. on your CI server or other deployment tool).




