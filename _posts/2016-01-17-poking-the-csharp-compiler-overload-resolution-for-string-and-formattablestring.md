---
layout: post
type : post
title : Poking the C# Compiler's Overload Resolution for String and FormattableString
date: 2016-01-17
tags : [C#]
published: true
---
Watching Rob Conery's [Exploring C# 6 with Jon Skeet](http://www.pluralsight.com/courses/csharp-with-jon-skeet) chapter on `String`s, especialy the last part of the chapter "Skeet Creates an ORM" gave me some ideas on possible usages of `FormattableString`.

I went ahead and poked the compiler to see how it reacted, and the results are somewhat surprising.

### A Methods Expecting a FormattableString

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        Print("Hello");
    }

    static void Print(FormattableString fs)
    {
        Console.WriteLine(fs.ToString());
    }
}
{% endhighlight %}

This doesn't even compile; the compiler is not willing to convert a `String` to a `FormattableString` for you.

Adding a `$` in front for the string does the trick, though:

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        Print($"Hello");
    }

    static void Print(FormattableString fs)
    {
        Console.WriteLine(fs.ToString());
    }
}
{% endhighlight %}

The interesting bits are only revealed when looking at what the compiler generated for us (I use [ILSpy](http://ilspy.net/)):

{% highlight csharp linenos %}
using System;
using System.Runtime.CompilerServices;

internal class Program
{
	private static void Main(string[] args)
	{
		Program.Print(FormattableStringFactory.Create("Hello", Array.Empty<object>()));
	}

	private static void Print(FormattableString fs)
	{
		Console.WriteLine(fs.ToString());
	}
}
{% endhighlight %}

It's interesting to look at the whole code, because as pointed out in the Pluralsight course, the compiler looks for a `FormattableStringFactory.Create` static method in the `System.Runtime.CompilerServices`. This is how `FormattableString` can be back ported in earlier version than .NET 4.6.

### An Overload that Takes a String

Let's see what happens when there is an overload that takes a `String`:

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        Print($"Hello");
    }

    static void Print(FormattableString fs)
    {
        Console.WriteLine("Print(FormattableString)");
    }

    static void Print(string s)
    {
        Console.WriteLine("Print(string)");
    }
}
{% endhighlight %}

Surprisingly, this prints:

> Print(string)

Looking at the generated code once more reveals that the compiler simply removed the `$` that was in front of our string.

{% highlight csharp linenos %}
using System;

internal class Program
{
	private static void Main(string[] args)
	{
		Program.Print("Hello");
	}

	private static void Print(FormattableString fs)
	{
		Console.WriteLine("Print(FormattableString)");
	}

	private static void Print(string s)
	{
		Console.WriteLine("Print(string)");
	}
}
{% endhighlight %}

So, it seems that we should us a real [interpolated string](https://msdn.microsoft.com/en-us/library/dn961160.aspx) to force the compiler to make a choice:

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        Print($"Hello today {DateTime.Today}");
    }

    static void Print(FormattableString fs)
    {
        Console.WriteLine("Print(FormattableString)");
    }

    static void Print(string s)
    {
        Console.WriteLine("Print(string)");
    }
}
{% endhighlight %}

Unfortunately, this still prints:

> Print(string)

Looking at the generated code reveals that the compiler seems to prefer calling `String.Format` on out interpolated string when there is an overload that takes a `String` parameter available.

{% highlight csharp linenos %}
using System;

internal class Program
{
	private static void Main(string[] args)
	{
		Program.Print(string.Format("Hello today {0}", DateTime.Today));
	}

	private static void Print(FormattableString fs)
	{
		Console.WriteLine("Print(FormattableString)");
	}

	private static void Print(string s)
	{
		Console.WriteLine("Print(string)");
	}
}
{% endhighlight %}
 
### Forcing the Compiler to Choose the Overload that Takes a FormattableString? 
 
 Now, would there be a way to _force_  the compiler into calling the `FormattableString` overload instead of the `String` overload? That would be quite nice because it would then allow us to process the `FormattableString` further before it is rendered as a `String`; while still keeping a `String` overload for those that simply want to send a `String`.
 
 My first naïve attempt was to change the signature of the `String` version and add `params object[]` to try to fool the compiler to call the `FormattableString` overload as it has fewer arguments:
 
{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        Print($"Hello today {DateTime.Today}");
    }

    static void Print(FormattableString fs)
    {
        Console.WriteLine("Print(FormattableString)");
    }

    static void Print(string s, params object[] args)
    {
        Console.WriteLine("Print(string)");
    }
}
{% endhighlight %}

But this is a failure, the `String` overload is still being called, and the compiler even goes as far as to add an empty array of objects in the call:

{% highlight csharp linenos %}
Program.Print(string.Format("Hello today {0}", DateTime.Today), Array.Empty<object>());
{% endhighlight %}

### Making the String Overload an Extension Method

What if the `String` overload was an extension method? In essence, I'm trying to make the `String` overload more _distant_, a less likable choice to the compiler, if you will. We know from previous versions of the C# language specifications (C# 6.0 specs not being out yet) that an extension method are less likely to be picked.

We have to change our code a bit to make it all work on instances, but if that works it'll be worth it:

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        var p = new Program();
        p.Print($"Hello today {DateTime.Today}");
    }

    void Print(FormattableString fs)
    {
        Console.WriteLine("Print(FormattableString)");
    }
}

static class ProgramExtensions
{
    public static void Print(this Program p, string s)
    {
        Console.WriteLine("Print(string)");
    }
}
{% endhighlight %}

Lo and behold, this finally prints what we want:

> Print(FormattableString)

The compiler is finally cornered into choosing the `FormattableString` version over the `String` version. Admittedly, it is a bit ugly, but it does the trick and can be put to good use in certain cases.