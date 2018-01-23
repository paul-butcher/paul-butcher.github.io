---
layout: post
title: Jenkins connectivity check
categories: jenkins
---

One aspect of ensuring that a deployment will go smoothly, is ensuring that it can happen at all.
If you are not yet in a position to practice continuous deployment, the process around a failure 
can be unpleasantly heavy.

Even if you are practicing continuous deployment, you might still end up in hot water if you are 
unexpectedly rejected by one of the hosts.

You can test the behaviour and performance of an application at each stage, both automatically and manually, 
before moving on to the next.  You can ensure that 
[no required definitions are missing]({% post_url 2017-08-29-checking-ansible-vars %}).  
You can ensure that 
[the right hosts will be touched]({% post_url 2017-09-05-verifying-ansible-inventories %}). 
You can plan an appropriate time for a deployment, 
so that no one is badly affected by an outage or unexpected change.

When you finally come to deploy the application to the next environment, you suddenly find that
it is a non-starter.  The credentials used for deployment have gone stale, been revoked, or
otherwise fail.  

Now you are embarrassed, you have to run a post-mortem and go through the process of working out 
and negotiating the outage, and the customer has to make do without that important new feature for 
however long all of that takes.

This could be avoided.  One way is to run regular [drift tests](http://docs.ansible.com/ansible/latest/test_strategies.html#check-mode-as-a-drift-test).  Another is to exercise those credentials away from 
the process of actually deploying an application.

If you use Jenkins Pipelines to deploy, you can also use them to run these connectivity checks, with
a script like this:

```
def transformIntoStage(host_name, environment) {
    return {
        stage(environment + host_name.tokenize(".")[0]​) {
            sshagent(credentials: ['my_key']) {
              sh "ssh my_user@${host_name} 'whoami'"
            }
        }
    }
}

node(){
    
    STAGING_HOSTS = ['s0.example.com', 's1.example.com']
    
    def stepsForParallel = [:]

    for (int i = 0; i < STAGING_HOSTS.size(); i++) {
        stepsForParallel[STAGING_HOSTS[i]] = transformIntoStage(STAGING_HOSTS[i], 'staging: ')
    }

    parallel stepsForParallel
}
```

This assumes that you normally connect via ssh, using the Jenkins
[SSH Agent](https://wiki.jenkins.io/display/JENKINS/SSH+Agent+Plugin), and 
[Credentials](https://wiki.jenkins.io/display/JENKINS/credentials+Plugin) plugins.  However, a 
similar pattern could be used for other kinds of connection.  Simply change the content of the
stage returned by `transformIntoStage` to match.

It runs through each host in parallel, and will pass as long as `my_user` can log in to all of them.

If `my_user` is rejected by any of them, you can take remedial action before it becomes a problem.
