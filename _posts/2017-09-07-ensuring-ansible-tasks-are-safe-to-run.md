---
layout: post
title: Ensuring Ansible Tasks are Safe to Run
categories: ansible
---

An important factor in testing definitions before deploying, is that
they do not require a connection to the hosts they would deploy to.  However, there is no 
guarantee that that is the case.  Someone could simply add a ['check_vars']({% post_url 2017-08-29-checking-ansible-vars %}) tag to a task that 
modifies the host and the tests are worthless.  Something is needed to guard against such an eventuality.

As long as you [keep deployment keys out of the vaults]({% post_url 
2017-08-30-keep-deployment-keys-out-of-the-codebase %}), you won't be able to do any harm by this,  
but it may be desirable for some environments to have deployment keys in the vault 
(particularly those environments that developers actively work on).  In that situation, if a 
developer erroneously marks some tasks as safe, then when they eventually come to try it out
on a protected environment, everything will fail for the wrong reason (i.e. that it couldn't 
connect, rather than because the definitions are incorrect).

A [callback_plugin](http://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html#callback-plugins) like the one below can be used to [fail fast](https://martinfowler.com/ieeeSoftware/failFast.pdf), and provide a much tighter 
feedback loop to warn the developer that their code is potentially dangerous and will fail the CI 
tests or commit hooks, and never make it to production.

Place it in a callback_plugins directory alongside the playbooks, and it will block any
tasks that are tagged as 'check_vars' but run on the target host.

```python

from ansible.plugins.callback import CallbackBase
from ansible.errors import AnsibleParserError


def is_marked_safe(task):
    # check_vars should always only run locally, so is (for this purpose) "safe"
    return 'check_vars' in task.tags


def is_local(task):
    """
    Test whether a task will be run locally.
    
    A task is run locally if it is either:
    
    - An action that runs locally
    - delegated to localhost

    :param task: An instance of ~ansible.playbook.task.Task
    :return: True if the task is to be executed on the local machine, False otherwise
    """
    # Certain actions are run locally anyway.
    # There may be more, but these are the ones that I have encountered in our safe tasks.
    if task.action in ['debug', 'assert', 'pause', 'fail', 'set_fact', 'include', 'include_vars']:
        return True

    if task.delegate_to in ['127.0.0.1', 'localhost']:
        return True

    return False


class CallbackModule(CallbackBase):
    """
    Ansible Callback plugin to check whether a task that has been tagged with check_vars
    is safe to call.
    """

    def v2_playbook_on_task_start(self, task, is_conditional):

        # A task that the author has marked safe, but actually requires a connection to the host
        # is bad, and must be blocked.
        if is_marked_safe(task) and not is_local(task):
            # Convert the task to a "fail".
            # Unfortunately, raising an exception simply presents a warning, and carries on.
            # It's essential that this is not permitted to continue.
            task.action = 'fail'
            msg_format = "'{}' is tagged with check_vars, but needs to connect to a remote host"
            task.args['msg'] = msg_format.format(task.name)
        return super(self.__class__, self).v2_playbook_on_task_start(task, is_conditional)

```

It is important to note that this is only a guard against accidental mistagging.  
A nefarious user (or one who is not nefarious, but sufficiently determined to make a mistake)
can easily bypass this by deleting this file.

It also does not prevent any action that goes in “through the front door“ of a web application
by visiting a URL from the control host.