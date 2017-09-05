---
layout: post
title:  "Checking vars in Ansible."
date:   2017-08-29 18:00:59 +0100
categories: ansible testing
---

[Ansible](https://www.ansible.com/) is great, but as with anything to do with deployment configuration,  there are some things you can only test when you actually use it to deploy to a given environment. You can reduce the number of surprises awaiting a production deploy, by checking vars without needing to connect to the target host. 

In the case of Ansible, you can ensure that your playbook installs the right things, and puts configs in the right place by running them against a test environment.  However, there are, by necessity, differences between environments, and these are handled by group and host vars.  Given that, how can you know that your vars for production are correct, or even complete?

Our current Ansible codebase follows the advice that Frederic de Villamil attributes to his colleague [ggreg](https://github.com/ggreg) in [Documenting your Ansible roles interface](https://t37.net/documenting-your-ansible-roles-interface-and-making-other-people-s-life-easier.html).  

Each role has in it at least one task with a 'check_vars' tag.  This can be used, not only to document the interface, but also to ensure that the variables have been set for all relevant hosts in a given environment. Crucially, unlike [check mode](http://docs.ansible.com/ansible/latest/playbooks_checkmode.html) it can do this without even connecting to the host.

This is an exceptionally useful, simple, and potentially very rich method for increasing confidence in the configuration of production environments.

Fred's examples check for undefined or empty vars, and in one case, there's a check to ensure that a given username isn't 'root', but that merely scratches the surface of what can be achieved with this technique.

Some other checks include:

- Where one var requires another:

```
- name: assert that a key exists for a certificate
  assert:
    that:
    - my_key is defined
  when: my_cert is defined
```
* Where the form of entries in a dict is important 

```
{% raw %}
- name: All foos are properly defined
  assert:
    msg: "{{ item }} is not properly defined"
    that:
    - item.value.description is defined
    - item.value.size is defined
  with_dict: "{{ foos }}"
{% endraw %}
```

* Where the form of a string must conform to a given filter (in this case, a regex, but other examples might be length or numbers within a range)

```
{% raw %}
- name: my_user is a valid username
  assert:
    that:
    # The key is used for the username - it must match NAME_REGEX
    - my_user | match("[a-z][-a-z0-9]*")
  with_dict: "{{ site_users }}"
{% endraw %}
```

* Where a var refers to a host that should be in a certain group in the same inventory, (or any other 'value from a set of options' constraint)

```

  - name: db_host refers to a database host
    assert:
      that:
        - db_host in groups['databases']
```

The checks will be run as part of a run with no `tags` option, but that will only prevent a given role (and subsequent roles) being run, which could lead to a half-completed deployment (which would be bad).  

A better approach is to run the playbook with the check_vars tag, as a gatekeeper to a deployment.

`ansible-playbook -i your_inventory -t check_vars site.yml && ansible-playbook -i your_inventory site.yml`

As it requires no access to the servers in the inventory, it can also form part of a commit hook or Continuous Integration test suite.

These are just a few examples of the ways in which this technique can help keep your environment definitions complete, reducing the cognitive load and fear involved in making changes across multiple environments.