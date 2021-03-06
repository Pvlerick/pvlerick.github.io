---
layout: post
type : post
title : Running an ASP.NET Core App in Docker
date: 2017-02-24
tags : [C#, ASP.NET Core, Docker]
published: true
---
In this post, I'll explain the necessary steps to run an ASP.NET Core app (targeting the 4.6.2 .NET Framework) in a Docker container.

My aim here is to create to do the whole process by hand, especially the writing of the Dockerfile and selecting the correct image.

### The Application

Let's create an empty web application targeting the .NET Framework and use the empty template. We'll keep it simple here and reuse most of the code from [the previous post](http://pvlerick.github.io/2017/02/running-dotnetcore-consoleapp-in-docker).

_Program.cs_
```csharp
public class Program
{
    public static void Main(string[] args)
    {
        var host = new WebHostBuilder()
            .UseKestrel()
            .UseContentRoot(Directory.GetCurrentDirectory())
            .UseUrls("http://*:5000")
            .UseIISIntegration()
            .UseStartup<Startup>()
            .UseApplicationInsights()
            .Build();

        host.Run();
    }
}
```

It is important to add the _.UseUrls()_ command in the builder to ensure that the application listens on all IP addresses, not only _localhost_. This is required because of an issue with Windows Containers on Windows 10.

_Startup.cs_
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMvcCore()
            .AddJsonFormatters();
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        loggerFactory.AddConsole(LogLevel.Debug);

        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseMvc();
    }
}
```

Nothing fancy here, just adding Mvc and using Json as a return format.

_QuotesController.cs_
```csharp
[Route("/[controller]")]
public class QuotesController : ControllerBase
{
    static string[] Quotes;

    static QuotesController()
    {
        Quotes = JsonConvert.DeserializeObject<Content>(System.IO.File.ReadAllText("quotes.json")).Quotes;
    }

    [HttpGet]
    public IActionResult Get() => Ok(Quotes[new Random().Next(Quotes.Length)]);

    [HttpGet("{count}")]
    public IActionResult GetMany(int count)
    {
        if (count < 0 || count > Quotes.Length)
            return BadRequest($"number of quotes must be between 0 and {Quotes.Length}");

        var r = new Random();
        return Ok(Quotes.OrderBy(_ => r.Next(Quotes.Length)).Take(count));
    }
}

public class Content
{
    public string[] Quotes { get; set; }
}
```

Not much here either, just some logic to be consistent.

The application can now be run with F5 and you can check that http://localhost:5000/quotes/ and http://localhost:5000/quotes/10 both work and return poetic quotes from our friend.

### The Dockerfile

```dockerfile
FROM microsoft/dotnet-framework:4.6.2

WORKDIR /app
COPY bin/Debug/net462/win7-x86/publish/ .

ENTRYPOINT ["DukeQuote.exe"]
```

As our application runs ASP.NET Core with the .NET Framework 4.6.2, our base image needs to be _[microsoft/dotnet-framework:4.6.2](https://hub.docker.com/r/microsoft/dotnet-framework/)_,

You can notice that there is no _EXPOSE_ command. That is because I am running this on Windows 10 machine where exposing ports does not work at the moment. The container will have to be targeted using it's ip address on the internal network.

After building and publishing (using ````dotnet publish```), we can build the image

```
docker build -t duke-quotes .
```

### Running it in Docker

Now that our image is ready, all that is left is to make a container out of it. We'll give it a specific IP address on the network for convenience.

Docker has a set of default networks (```docker network ls``` to list them), the easiest is to us an address that is in the range of the ```nat``` network - we can see that range by using the ```docker network inspect nat``` command.

Let's pick an address in that range and run out container with that specific IP address:

```
docker run -d --net nat --ip 172.18.64.42 --name duke-quotes-c duke-quotes
```

Short explanation of the flags
 - ```d```: detached mode; the container will start in the background
 - ```net```: specify in which network the container should reside
 - ```ip```: assigns a specific ip to the container
 - ```name```: assigns a name to the container

Hitting [http://172.18.64.42:5000/quotes/5](http://172.18.64.42:5000/quotes/5) will now get you quotes by calling the app in the container.

You can have a look at the logs of the application by using

```
docker logs duke-quotes-c
```

You can find the [full sources here](https://github.com/Pvlerick/DockerSandbox/tree/master/DockerAspNetCoreNetFramework).

> Written with [StackEdit](https://stackedit.io/).
