---
layout: post
title: Ansible Smoke Testing
categories: ansible testing
---

An advantage of declarative code like (most of) Ansible, is that it is it's own test suite.  
If the state of the host can't be made to match your definitions, it's a failure.  
But what if your definitions are wrong?

A disadvantage is that, being its own test suite, it lacks the 
[double-entry bookkeeping](http://www.butunclebob.com/ArticleS.UncleBob.TheSensitivityProblem) aspect.

There are still things that could be forgotten or misconfigured when defining an environment, 
particularly when templating arcane configuration files.  
Maybe you put an SSL certificate on the wrong box, or mistyped a port number in a firewall rule.

A lot of what might go wrong would be easily detected on the first human visit, but some things 
may not.  Maybe the firewall blocks the load balancer on one of your application servers, or 
maybe the production load balancer points to the staging estate; perhaps
half of your database cluster has split off into a cluster of its own.  Maybe you have a 
condition that stops your application being upgraded if it's already there.

One way to check this sort of thing is with smoke test plays.

# Examples of things to check for:

- Firewall
    - Is everything that should be allowed, allowed?
    - Are hosts outside the environment (e.g. the machine running Ansible) forbidden from 
    accessing certain ports?
- Web application
    - Does visiting the home page work?
    - Does it contain some expected piece of text (e.g. a version id string, the title)?
    - Do all application servers work, does the load balancer?
    
These tests don't prove that the application works properly and fulfils the expected 
requirements, that's for your test suite and manual test plan to judge.  They just give a quick indication 
that it's not completely broken.  

Crucially, adding tests like this means that a failed deployment can immediately raise a red flag, 
even if all of the tasks that should have configured your hosts appeared to work.  This allows you
to get on with diagnosing and fixing it immediately, rather than waiting for human testing or 
log-based reporting to notify you of a problem.

# How

I've experimented with various ways of achieving this, with handlers and roles, and decided 
that it is best handled by separate plays, with the test tasks defined within them.  Possibly as
a separate playbook which can be included at the end of your main playbook 
(though tags work just as well).   This separation means that:
  
 - It can easily be run as part of a diagnostic effort if something goes awry independent of an 
  Ansible run;
 - The whole smoke test suite is together, so you can see at-a-glance what is being tested, and 
 in the results, what was tested.
 - Tests can be added that are unrelated to specific individual roles.  
     - You are testing whether the estate is correctly configured.
 - You don't have to worry about timing and flushing handlers. By placing a test play at the end, 
 you know that everything else has already been done.

# Example plays

## Firewall

This firewall test play checks three things - 

1. That the ports used for intercommunication between the servers are accessible from each host
2. That those ports are not accessible from the machine running Ansible (this assumes that Ansible is not running on one of those hosts)
3. That the http and https ports are accessible from the machine running Ansible

{% raw %}
```yaml
    - name: smoke test firewall
      hosts: all
      gather_facts: no
    
      tasks:
        - name:  "Notify the firewall smoke tests"
          debug:
            msg: "Trigger smoke tests"
          notify: "smoke"
          changed_when: true
    
      handlers:
        - name: intercommunication ports
          wait_for:
            host: "{{ inventory_hostname }}"
            port: "{{ item[0] }}"
            timeout: 5
            state: started
          delegate_to: "{{ item[1] }}"
          with_nested:
            - [43594, 43595]
            - "{{ intercommunication_hosts }}"
          listen: "smoke"
    
        - name: local machine cannot reach intercommunication ports
          wait_for:
            host: "{{ inventory_hostname }}"
            port: "{{ item }}"
            timeout: 5
            state: stopped
          delegate_to: 127.0.0.1
          become: no
          with_items:
            - [43594, 43595]
          listen: "smoke"
          
          - name: local machine can reach web ports
          wait_for:
            host: "{{ inventory_hostname }}"
            port: "{{ item }}"
            timeout: 5
            state: started
          delegate_to: 127.0.0.1
          become: no
          with_items:
            - [80, 443]
          listen: "smoke"
```
 {% endraw %}
       
## Home Page

This play visits the home page of the load balancer and each application host.  The page is checked for the presence of the text 'My Home Page'.

This proves that the application "works" in the sense that it is wired up to the web 
server, and you don't just get a 500 or the default "Congratulations, you've just configured your webserver" page.

If you are installing an update to an application, adding some version-specific text to your homepage (either visible or hidden), and checking it here, will also give you confidence that your update has been applied.

By proxy, this might indicate correctness deeper in your configuration.  In Django, for example,
it won't serve anything unless it connects appropriately to the database.

You might have a status page served by your application that checks whether all appropriate services
are correctly configured.  You can use this technique to check that page too.

{% raw %}
```yaml
    - name: run client-side smoke test
      hosts:
      - load_balancer
      - application
      connection: local
      gather_facts: no
      tasks:
    
        - name:  "Notify the smoke tests"
          debug:
            msg: "Trigger smoke tests"
          notify: "smoke"
          changed_when: true
    
      handlers:
    
        - name: "Get homepages"
          environment:
            no_proxy: "{{ inventory_hostname }}"
          uri:
            url: "https://{{ inventory_hostname }}/"
            return_content: yes
            follow_redirects: all
            status_code: 200
            validate_certs: no
          register: homepage
          listen: "smoke"
    
        - name: "Ensure that the homepage serves real content"
          assert:
            that:
              - "'My Home Page' in homepage.content"
              - "{{ application_version }}" in homepage.content
          listen: "smoke"
 ```
{% endraw %}
