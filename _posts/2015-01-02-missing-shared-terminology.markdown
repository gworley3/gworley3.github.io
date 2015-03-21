---
layout: post
category: programming
title:  Missing Shared Terminology
date:   2015-01-02
listed: false
permalink: /2015/01/02/missing-shared-terminology.html
published: true
---

I wish as professional programmers we could agree a bit more on terminology. We all agree to use "object" and "class" and "function" and "closure" to each refer to a category of things that differ in various ways in implementation but look enough like the other implementations that we can form coherent clusters, but we don't seem to have widely agreed what to call some of the other clusters.

At the moment I'm thinking of what I frequently call the "[aspect](http://en.wikipedia.org/wiki/Aspect-oriented_programming)" cluster, which are those bits of cross-cutting functionality you include in multiple classes and closures. But we call these all sorts of things in different languages, often depending on details of the implementation: concerns, aspects, mix-ins, includes, templates, etc.. This makes it difficult to talk about "aspects" with other programmers because we first have to negotiate the terminology unless we are both using the same language.

Not having a shared abstraction is hurting us. It makes it difficult to generalize ideas and share them between implementations (languages). Consequently, each community has had to rediscover how to use aspects when we've actually had some good ideas about it stretching back to the early 1990s. Sure, not every lesson of C++ templates or Ruby includes is applicable to your implementation, but there is wisdom about aspects we could be sharing but largely aren't.

I sort of blame the heavy use of Java and C++ in universities for this problem because both of those languages make using aspects difficult, so it's something left off in school as "advanced stuff you don't really need". Thus professors haven't had to spend a lot of time educating their students about the theory of aspects, so millions of working programmers don't have a shared background knowledge on aspects they way they do on classes and functions.

So can we all agree to call these things "aspects" and use the terminology far and wide? I like "aspect" because it doesn't attempt to explain anything about a particular implementation, yet it conveys the basic concept of "thing I can add to another thing to give it extra functionality", but if not "aspect" then "concern" or "include" or something so we can consolidate this part of programming and make it easier to communicate about it and search for it.
