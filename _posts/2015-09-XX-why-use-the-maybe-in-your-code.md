---
layout: post
type : post
date: 2015-09-20
tags : [C#]
published: false
---
# Why Use the Maybe<T> in Your Code?

The Maybe monad in C# has been around for quite some time in various forms, with some recommandations to use it in your code.
I've always liked the idea of having a _cleaner_ way to check if an variable holds a value and I found the calling code more compelling that the usual _null_ check.

However, I have been  struggling to explain to others why they should use it instead of simply making null checks, which vast amounts of programmers aren't bothered with. It's only recently that I gathered my thought around it and started to make a reasoning why it's simply better than other ways to go about.

## Short Introduction to the Maybe

> If you know what the Maybe monad is, skip ahead. Otherwise, I'll introduce it very briefly.

You may have head of the maybe monad in articles such as [this one](http://blogs.msdn.com/b/wesdyer/archive/2008/01/11/the-marvels-of-monads.aspx) and [that one](http://ericlippert.com/2013/04/02/monads-part-twelve/).

The Maybe is a simple generic type that can hold a value or not, as simple as that. The Monad will hold the value if there is one and the object can be queried to check if there is a value or not.

Here is a very simple implementation:

```CSharp
using System;

public struct Maybe<T>
{
    public static Maybe<T> Empty = default(Maybe<T>);
    
    private readonly T _value;
    
    public bool HasValue { get; }
    
    public Maybe(T value)
    {
        this._value = value;
        this.HasValue = true;
    }
    
    public T Value
    {
        get
        {
            if (!HasValue)
                throw new InvalidOperationException("No value");
                
            return _value;    
        }
    }
}

public static class MaybeExtensions
{
    public static Maybe<T> ToMaybe(T value)
    {
        return new Maybe(value);
    }
}
```

You can get this version [on one of my GitHub repository](https://github.com/Pvlerick/Maybe), but I would suggest [using this one in your code](https://github.com/AndreyTsvetkov/Functional.Maybe) instead.

Side note: in F#, an pre existing `Option` type is available and works like a charm with pattern matching. Unfortunately, C# doesn't have such facility, but the actual implementation is trivial.

## Advantages of Using Maybe<T> in Your Code

Let's go over an example to have code we can discuss.

Let's say that we have an interface to retreive some `Customer` object.

```CSharp
interface ICustomerRepository
{
    Customer GetById(string id);
}
```

When calling the `GetById` method, assuming that `Customer` is a reference type and there is no documenation on the interface, you'll have to check for `null` and will end up with code probably simillar to this:

```CSharp
var customer = customerRepository.GetById("ABC");

if (customer != null)
{
    //Meaningful code
}
else
{
    //Recovery
}
```

Quite standard code that you see dozen times a day.

Let's inspect what happens when the interface uses the Maybe type:

```CSharp
interface ICustomerRepository
{
    Maybe<Customer> GetById(string id);
}
```

And the calling code will look like this:

```CSharp
var customer = customerRepository.GetById("ABC");

if (customer.HasValue)
{
    var c = customer.Value
    //Meaningful code
}
else
{
    //Recovery
}
```

As you can see, the code is similar apart that you have to extract the real value out of the Maybe in the second case.

So, why use the Maybe then?

If you come to think about it, I believe that it's better than a null check: because the Maybe is a value type, you know that you will be returned something. That something (the Maybe object) will tell you if a value was returned or not, forcing the caller to make the check before calling `.Value` (admitely the caller could call it directly but then he would get what he deserves). A null check can always be forgotten and brind its dreaded `NullReferenceException`, while if you work with Maybe you don't have a choice.

But wait you say, the comments on the interface/class mention that if no customer is found, null will be returned.

That's a good point. However, it means two things:

1. This is somewhat an implementation detail, and it's sad to see that in an interface comments; if that service is injected then the comments on the class are useless.
2. What if the implementation changes and suddenly an exception is thrown instead? Or a default value is returned, or what no? The great thing about Maybe is that the contract is expressed _in code_ and _code doesn't get out of sync_.

But let's see another more advanced scenario. Suppose that the interface takes a second member as such:

```CSharp
interface ICustomerRepository
{
    Maybe<Customer> GetById(string id);
    Customer GetBest();
}
```

(This is a very silly example, I know, but I'm out of inspiration at this time)

Now, something magic happens: if you call `GetBest`, because it returns a `Customer` directly, you have an implicit contract telling you that the returned `Customer` will not be null!

A last example: imagine a function that returns a `string`. That is very frequent, and it is also very frequent to have this kind of method returning `null` or an empty `string` when the operation failed for some reason. The caller will then check the value of the string, often by using the popular `String.IsNullOrEmpty` that serves as a catch-all.
The problem with this kind of code is that it relies on implemenation details again; how ofter do you see programmers checking the called method to check what they should test for in the caller? The Maybe avoids all those issues by making it explicit if the returned value can we worked with or not.

## Typical Comment and Questions

### It's The Same As Nullable<T>

Not really :smile:

Despite the similarity in the code, it's actually the complete inverse. Nullable types are used to give the opportunity to be null to a value type. The Maybe does the oposite: it gives any type the option to always have a value.

### I'm Not Bothered by `null`

There is nothing fundamentaly wrong with `null` itself, but the issue lies with the way it is commonly used. Methods tend to return null when they don't know what to do but don't want to take the responsibility to throw an exception. It is fine to let the caller to apreciate if recovery is possible or not, but then it should be made explicit: that's exactly what the Maybe does.