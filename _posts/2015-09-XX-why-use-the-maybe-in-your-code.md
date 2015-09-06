---
layout: post
type : post
date: 2015-09-XX
tagline: "Supporting tagline"
tags : [C#]
published: false
---
{% include JB/setup %}

# Why Use the Maybe<T> in Your Code?

The Maybe monad in C# has been around for quite some time in various forms, with some recommandations to use it in your code. I've always liked the idea to have some kind of having a _cleaner_ way to check if an object has a value and I found the calling code more compelling that then usual _null_ check.

However, I was struggling to explain to others why they should use it instead of simply making null checks, which vast amounts of programmers aren't bothered with. It's only recently that I gathered my thought around it and started to make a reasoning why it's simply better than other ways to go about.

Let's go over some examples.

1. Forces the caller to check (if he doesn't, he is nuts)
2. Implicitly makes all other members of that type that return ref types safe if the retun value is not a Maybe
3. Null can be returned and it can be explained in the comments, but comments might lie whereas code cannot lie