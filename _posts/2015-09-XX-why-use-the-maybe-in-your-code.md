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

---

How to avoid propagating null in the code
Have you ever wondered how to avoid propagating null? If you are writing a function that can potentionally return no result, how to make it so without returning null?
 
Read on :-)
Problem Statement
First, lLet's look at the semantics: null is a value that indicates the absence of a value (how ironic).
In C#, it can be assigned to any reference type variable.
 
However, it is very ofter wrongly used to mean much more than the absence of a value. When returning null, no information is conveyed as the reason for not having an object, apart from the technical fact that there is no object.
The other issue is that this forces the caller to check for null at various level to make sure there is an object, which makes the code fragile: you are always one line away from a NullReferenceException :-)
Empty Sequence
One first good habit is to return empty sequences when a method should return a sequence but there is nothing to returnare no elements to return. This makes a lot of sense: if I'm looking for merchants that match a certain criteria, if no merchant are matching I should be returned en empty list, not null.

This is very kind to users of your code, since they can safely foreach on the result of the call without having to worry about a null being returned. This can be easily achieved by returning a new empty list; or better yet, use Enumerable.Empty<T> method, e.g: Enumerable.Empty<string>() is an empty sequence of strings.

However, this only applies to sequences. But there is a solution for single values: the Maybe!
Introducting Maybe<T>
Maybe<T> is a small utility struct located in Ogone.Core.Utility that allows to stop null propagation. In essence, the Maybe is a value that indicates if the desired value was returned or not.

Let's take a simple example: say we have method that retuns a string but that for some reasons might not have a value to return. In this scenario, most programmer will return null or en empty string to "indicate" that there is no value to be returned. However, there are issues with this approach: what should the called do? Check for null and empty, using String.IsNullOrWhitespaces? What if an empty string is a valid value in that context? The fundamental issue when goig down this road is that you force the caller to look at implementation details to find out what can be returned. And we all know by now that you should program agains abstractions and not implementations :-) What if that implementation changes later on?

Instead of returning a string, you can mark the method as retuning a Maybe<string> instead. This first change guarantees the called that something will be returned (no more null since this is a value type). The returned Maybe itself indicates if there is a value or not.

Now that we have a value to work with, we can check if it contains the type we initially expected using the HasValue property. If it is true, then it is safe to use the Value property that has the type T; if not, then it means no value was returned.

You might say that this does not change much from returning null and then checking for null at the caller; you are correct in some way: the Maybe will have to be checked for value just as the null has. However, there are two advantages here:

1.a Maybe<T> is explicit to say that a function can or cannot return a value, the caller doesn't have to check inside the implementation to see if a null can be returned and in which conditions.
2.a Maybe<T> in an interface tells you that a function can return or not a value, so it means that in the same interface if a non Maybe is returned, it is guaranteed to have a value - no more null checks ;-)
Going Further
In essence, the Maybe is a Monad. Although theory migth be daunting, it's a realitvely simple concept that you are using every day with LINQ - it is a "general pattern for “amplifying” types while maintaining the ability to apply functions to instances of the underlying types."

A good introduction is is Brian Beckman's introduction on Monads on channel9, and Eric Lipert's series on the topic.
