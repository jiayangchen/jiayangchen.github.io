---
layout: post
title: "Recitation 3 Eraser: A Dynamic Data Race Detector for Multithreaded Programs "
subtitle: "论文阅读笔记 | Notes when reading papers"
date: 2016-10-18
author: "ChenJY"
header-img: "img/recitation.jpg"
catalog:    true
tags:
    - 论文阅读
    - Paper Reading
---

>Read the Eraser paper, and think about the following questions as you read the paper:
>
* According to the lockset algorithm, when does eraser signal a data race? Why is this condition chosen? 
* Under what conditions does Eraser report a false positive? What conditions does it produce false negatives? 
* Typically, instrumenting a program changes the intra-thread timing (the paper calls it interleaving). This can cause bugs to disappear when you start trying to find them. What aspect of the Eraser design mitigates this problem? 
* Please raise at least one question of your own for the discussion. 

### Question1：According to the lockset algorithm, when does eraser signal ...

* If C(v) becomes empty this indicates that there is no lock that consistently protects v. The reason is
that locks_held(t) is the set of locks held by thread t and C(v) is the set of all locks, if C(v) ∩
locks_held(t) == NULL then it means that no lock are protecting the variable.
* And when it comes to Initialization and ReadSharing,
it uses four states and race will only be reported
when a write access from a new thread changes the state from Exclusive or Shared to the SharedModified
state because simultaneous reads of a shared variable by multiple threads are not races, there
is also no need to protect a variable if it is readonly,
and it reports races only after an initialized variable
has become writeshared
by more than one thread.

### Question2：Under what conditions does Eraser report a false ...

* False Positive: The Eraser will do a good job only if the test case causes enough shared variable reads
to follow the corresponding writes.
* False Negatives: If a thread t1 reads v while holding lock m1, and a thread t2 writes v while holding l* ock
m2, the violation of the locking discipline will be reported only if the write precedes the read.

### Question3：Typically, instrumenting a program changes the intra ...

* Eraser’s testing is not very sensitive to the scheduler inter* leaving because because the paths violate
the locking discipline any variable regardless of the interleaving produced by the scheduler.

### Question4：Please raise at least one question of your own for ...

* How locks of different granularity are used in a computer sys* tem？
