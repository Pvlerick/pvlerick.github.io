---
layout: post
type : post
title : How to Get Rid of a Singleton
date: 2017-03-01
tags : [C#, Design Patterns]
published: true
---
Singleton have been qualified as [evil](https://blogs.msdn.microsoft.com/scottdensmore/2004/05/25/why-singletons-are-evil/), [stupid](https://sites.google.com/site/steveyegge2/singleton-considered-stupid), [and so on](http://stackoverflow.com/questions/137975/what-is-so-bad-about-singletons), for quite some time now.

I'm just to add my two cents to the fray (years later) by saying that I disagree with the principle that singletons are _bad_ per say and try to untangle the issue. In short, the problem lies with their implementation, more specifically the way it is implemented in the [GoF book](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612/ref=sr_1_1?ie=UTF8&qid=1488379218&sr=8-1&keywords=design+patterns).

Why am I doing this? Because I still encounter _Singletons_ in the wild and some developers still think that it is a good idea it its raw form...

### TL;DR

**A class being a singleton is be an implementation detail. _Singletons_ are not bad, but the callers should not be aware that they are calling a _Singleton_.**

How do we achieve this? By using _Dependency Injection_, of course!

It's actually telling that all the DI containers have a _Singleton_ life-time implementation, which is the recommended usage for all dependencies that are stateless as it will be very performant. It seems that we don't like _Singletons_ but still we like _Singletons_.
Whoever is using these _Singletons_ that have been registered in the container will receive them through an abstraction and will not be aware that this instance is being shared by god-knows how many callers.

And that is the way to get "rid" of a dependency on a _Singleton_: get the calling code to use an _abstraction_ instead.

The rest of the post will focus on the analysing the issue in depth and solving it in C#, but it probably more or less applies to other languages.

### The _static_ Problem

If you take the classical implementation of the _Singleton_ pattern ([courtesy of Jon Skeet, so we know it's correct](http://csharpindepth.com/Articles/General/Singleton.aspx)), it uses _static_:

```csharp
public sealed class Singleton
{
    private static readonly Lazy<Singleton> lazy =
        new Lazy<Singleton>(() => new Singleton());
    
    public static Singleton Instance => lazy.Value;

    private Singleton() { }
}
```

In general, _static_ is **bad**. I allow myself to use _static_ only when the method or property has no side effect whatsoever. A method performing an addition would be a good example of something that can be _static_, while accessing a database or logging is not.

**Side note**: although Jon wrote that _Singleton_, it is more of a demonstration on how to do it in C# than a recommendation to use it. I'm fairly certain Jon would not recommend you to use the _Singleton_ in the calling code but instead to have it injected behind an _abstraction_, which is exactly what he has done with NodaTime's [IClock](https://github.com/nodatime/nodatime/blob/master/src/NodaTime/IClock.cs) and [SystemClock](https://github.com/nodatime/nodatime/blob/master/src/NodaTime/SystemClock.cs).

And why is _static_ bad, you may ask?

I see two main reasons:

 1. You have a dependency on another concrete class;
 2. You render your code un-testable.

The first one is easy: most of the times, a singleton is a central access to a resource - a database, some logging tool, the network... That is the **definition** of an external dependency, which you definitely should not depend on directly but use through an _abstraction_ (_interface_ or _abstract class_).

The second one is easy too: by not using an _abstraction_, you cast in concrete the behaviour of your code and make it impossible to change it for testing purposes. If the _Singleton_ writes to a database, your test code will do so too.

Let's take another simple but compelling example: _DateTime.Now_, the nemesis of testable code.

Leaving apart the gazillions issues that arise from using this as is (time zones etc), if any logic of your code depends on the current time and you got it from _DateTime.Now_, you cannot test it (yes I know that you can use [Fakes](https://msdn.microsoft.com/en-us/library/hh549175.aspx) which are super cool but you should use them for testing in legacy code, not new code that you are writing). I hope this heps making the point accress that _static_ is generally a bad idea, unless previously stated conditions and maybe some other exceptions that I didn't think of.

### Getting Rid of the Singleton

Let's get down to it: how to refactor your code to get rid of that hard cast concrete dependency on a _Singleton_?

I'm going to propose two ways of doing this, depending on whether or not you own the code of the _Singleton_.

Both examples are going to use the same starting code.

Here is our _Singleton_:

```csharp
public sealed class Singleton
{
    private static readonly Lazy<Singleton> lazy =
        new Lazy<Singleton>(() => new Singleton());

    public static Singleton Instance => lazy.Value;

    private Singleton() { }

    public string CreateMessage() => "Message";
}
```

And here is our calling application:

```csharp
public sealed class Foo
{
    public void Bar()
    {
        var message = Singleton.Instance.CreateMessage();

        Console.WriteLine(message);
    }
}
```

#### When You Own the Singleton Code

That's the easiest scenario. The plan is very simple: extract an interface from the _Singleton_, make the calling code depend on that _interface_ instead of the _Singleton_, give the _Singleton_ as a parameter (ideally in the constructor of the caller, if you are doing _DI_).

The first thing is to extract an interface from the _Singleton_, writing it by hand or using Visual Studio to do it for you. Whichever way you pick, here is the resulting interface:

```csharp
public interface ISingleton
{
    string CreateMessage();
}
```

**Side note**: if you want to do proper DI & [DDD](https://en.wikipedia.org/wiki/Domain-driven_design), the interface should belong to the caller, especially if the _Singleton_ is some kind of data access or infrastructure concern. Let's leave this for a moment, more on that in the second solution.

Now that we have this interface next to our ```Singleton```, our ```Singleton``` should implement it (which is done automatically if you used Visual Studio's extract interface feature):

```csharp
public sealed class Singleton : ISingleton
```

Now, our caller can take a dependency on ```ISingleton``` instead of ```Singleton``` and get it trough its constructor (_DI_ again) and keep the reference in a private field and call it from there:

```csharp
public sealed class Foo
 {
     private readonly ISingleton singleton;

     public Foo(ISingleton singleton)
     {
         this.singleton = singleton;
     }

     public void Bar()
     {
         var message = this.singleton.CreateMessage();

         Console.WriteLine(message);
     }
 }
```

Now, this is all well and good, this class suddenly became testable and does not depend on the the concrete ```Singleton``` anymore. This is the ideal scenario.

However, there is a caveat, which is that adding this constructor is a breaking change. Users of this class that were using the (default) parameterless constructor will all be broken.

We can fix this by adding an extra parameterless constructor that preserves the current behaviour:

```csharp
public Foo() : this(Singleton.Instance) { }
```

Now in DDD this would be a sin as the class now has again a dependency on ```Singleton```. However, since we are refactoring, this class already had that dependency, and if a lot of code is calling the ```Foo``` class, you might not want to spend 2 days refactoring all the callers. This is a judgement call.

#### When You **Don't** Own the Singleton Code

Starting from the initial code, this case is slightly more complicated, but only very slightly.

What we are going to do is create our own _abstraction_, on the caller side, and then write an [_Adapter_](https://en.wikipedia.org/wiki/Adapter_pattern) to the ```Singleton``` class. 

Let's first build our own _abstraction_ which I'll call ```IMessageCreator``` (after all, the only method on ```Singleton``` is ```CreateMessage```, who's the idiot who called it ```Singleton```?)

```csharp
public interface IMessageCreator
{
    string CreateMessage();
}
```

This is actually the way you would do it according to [APPP](https://www.amazon.com/Agile-Principles-Patterns-Practices-C/dp/0131857258) since we know that the interface belongs in the caller.

Now, we can make our code depend on it instead of depending on ```Singleton```:

```csharp
public sealed class Foo
{
    private readonly IMessageCreator messageCreator;

    public Foo(IMessageCreator messageCreator)
    {
        this.messageCreator = messageCreator;
    }

    public void Bar()
    {
        var message = this.messageCreator.CreateMessage();

        Console.WriteLine(message);
    }
}
```

As in the previous example, this simple change instantly made our class testable (that is, except for the call to ```Console.WriteLine```, probably the same idiot who called the _Singleton_ ```Singleton```... Wait, was that me a few years ago?)

In any event, what we need now is an _Adapter_ that makes ```Singleton``` tha shap of a ```IMessageCreator```, and nothing is simpler than this:

```csharp
public class MessageCreatorAdapter : IMessageCreator
{
    public string CreateMessage() => Singleton.Instance.CreateMessage();
}
```

And we can now inject that in our ```Foo``` class' constructor and everything will work again.

As in the previous example, if you want to not break all the callers, simply add a parameterless constructor that creates the _Adapter_:

```csharp
public Foo() : this(new MessageCreatorAdapter()) { }
```

This will bring back the (hidden) dependency on _Singleton_, again a judgement call which depends on how many callers you would have to modify (which [if you are doing DI the right way](https://www.manning.com/books/dependency-injection-in-dot-net) using a container, is 0).

### Conclusion

Getting rid of a singleton is extremely easy: create an abstraction and depend on that instead of the concrete _Singleton_. Once this is done, you'll have to decide if you want to fix all the callers of your class or if you only want to do this locally because you don't have time for that or there are to many callers.

In any event, your code will be more testable and better after this refactoring. No more excuses!

I wrote this article after giving a talk called "Writing Testable Code" (in French) at the [DevDay 2017 conference in Belgium](http://www.devday.be/).