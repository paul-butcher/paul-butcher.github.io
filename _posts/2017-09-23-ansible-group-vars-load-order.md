---
layout: post
title: Ansible group_vars load order
categories: ansible
---

The order of precedence for Ansible vars is defined in the Ansible documentation under [Variable precedence](http://docs.ansible.com/ansible/latest/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable), but what if two vars have the same level of precedence according to the list?  

Later it mentions that "last one wins" where they are equal, but says nothing about the order of loading.

> Within any section, redefining a var will overwrite the previous instance. If multiple groups have the same variable, the last one loaded wins. If you define a variable twice in a play’s vars: section, the 2nd one wins.

I turns out that groups are loaded alphabetically (though child groups win: See [Inventory](http://docs.ansible.com/ansible/latest/intro_inventory.html#groups-of-groups-and-group-variables)).

Ideally, this wouldn't matter.  You should ideally avoid overriding vars even when the precedence is obvious.  (As stated in the [Ansible documentation](http://docs.ansible.com/ansible/latest/playbooks_variables.html): "Ultimately it’s Ansible’s philosophy that it’s better you know where to put a variable, and then you have to think about it a lot less.")

However, it does happen, and can cause problems if you start renaming your groups.  This is a good reason to follow the advice to define your vars in one place.

Recently, I was reorganising some Ansible code, and discovered a var, buried in a group_vars file,  that was accidentally relying on this fact.  Fortunately, I noticed this before any problems arose.

To isolate this for a demonstration, imagine this scenario.  

You have an environment where you install a product and its backing services on the same host:
```
[backing_service]
myhost

[product]
myhost
```

Somewhere buried in the vars files for `product` and `backing_service` is a different value for the same var.

product.yml
```yaml
myvar: 1
```

backing_service.yml
```yaml
myvar: 2
```

That's OK, because the value you want for `myvar` on `myhost` is `1`.

```
$ ansible -i inventory myhost -m debug -a 'var=myvar'
myhost | SUCCESS => {
    "myvar": 1
}
````


You want to add a new instance of `backing_service` to the environment, and you realise that some of the configuration of that service is specific for the first product, so you change the group name to reflect that, and rename `backing_service.yml` accordingly.

```
[product_backing_service]
myhost

[product]
myhost
```

Unfortunately, this change makes the vars for `product_backing_service` override the vars for `product`, and now `myvar` is `2`

```
$ ansible -i inventory myhost -m debug -a 'var=myvar'
myhost | SUCCESS => {
    "myvar": 2
}
```