---
layout: post
title: Radon and the Jenkins warnings plugin
categories: jenkins, warnings
---

The [Warnings plugin](https://wiki.jenkins.io/display/JENKINS/Warnings+Plugin) for [Jenkins](https://jenkins.io/) 
is a very versatile tool for reporting on the health of a job.  [Radon](https://radon.readthedocs.io/en/latest/) 
is a useful tool for analysing a Python codebase. Here's how they can play together.

One of the great things about the Warnings plugin is that you can very easily write your own parsers for any
build or analysis tool.  All you need is a regular expression, and a small [Groovy](http://groovy-lang.org/) script.

The two metrics I am interested in are [Cyclomatic Complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity),
and Maintainability Index (bearing in mind some 
[caveats](https://avandeursen.com/2014/08/29/think-twice-before-using-the-maintainability-index/)).  Radon reports 
maintainability on a per-file basis, and so provides an indicator of where one might focus refactoring efforts.

[Parsers for the Warnings plugin](https://github.com/jenkinsci/warnings-plugin/blob/master/doc/Documentation.md#creating-a-new-tool-in-the-user-interface)
 can be added on the `/configure` page on your Jenkins instance.  Regular expressions
and mapping scripts for the `radon cc` and `radon mi` are given here.


Together, these two parsers show most of what you might need to do with Warnings -
 * Per-file messages
 * Per-line messages
 * Warnings spread over multiple lines
 * Prioritising warnings
 
## Radon CC

On Jenkins `/configure`, add a parser with a Regular Expression `(.*)\n\W+.\W(\d+):\d+\W(.*) - (A|B|C|D|E|F) (.*)`, 
and a Mapping Script:

```
import hudson.plugins.warnings.parser.Warning
String filename = matcher.group(1)
String line = matcher.group(2)
gradeMap = [
    A:'low - simple block',
    B:'low - well structured and stable block',
    C:'moderate - slightly complex block',
    D:'more than moderate - more complex block',
    E:'high - complex block, alarming',
    F:'very high - error-prone, unstable block'
]
String message = gradeMap.get(
    matcher.group(4))+ " " + matcher.group(4) + " " + matcher.group(5)
return new Warning(
    filename, Integer.parseInt(line), 
    'complexity', 'complexity', message
);
```

Add Radon CC to your build script - `radon cc -s myproject` will produce an output containing entries
that will be matched by the above CC parser.  Add `-n D` to only include blocks that are "more complex" or worse.

```
myproj/myfile.py
    M 185:4 MyClass.my_method - C (12)
```

## Radon MI

On Jenkins `/configure`, add a parser with a Regular Expression `(.*) - (B|C) (.*)`, and a Mapping Script:

```groovy
import hudson.plugins.warnings.parser.Warning
import hudson.plugins.analysis.util.model.Priority

String filename = matcher.group(1)
String type = 'maintainability'
String category = type
gradeMap = [B:'Medium', C:'Extremely Low']
String message = gradeMap.get(
    matcher.group(2)
    )+ " maintainability: " + matcher.group(2) + " " + matcher.group(3)
priorityMap = [B:Priority.NORMAL, C:Priority.HIGH]
return new Warning(
    filename, 0, type, category, 
    message, priorityMap.get(matcher.group(2))
);
```


Add Radon MI to your build script - `radon mi -s myproject` will produce an output containing lines
that will be matched by the above MI parser.  Add `-n B` to leave out files with "very high" maintainability.

```
myproject/setup.py - A (69.77)
myproject/main.py - B (18.32)
myproject/utils.py - C (8.51)
```
