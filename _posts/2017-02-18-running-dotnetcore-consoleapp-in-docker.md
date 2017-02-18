---
layout: post
type : post
title : Running a .NET Core ConsoleApp in Docker
date: 2017-02-18
tags : [C#, Docker]
published: true
---
Whie fiddlingz with Docker and .NET Core, I was unable to locate a clear and concise tutorial that would explain what are the steps to create a Docker image with a simple .NET Core ConsoleApp and how to run it properly.

As I am trying to learn Docker, I'd like to understand the basics; as oposed to use Visual Studio's magical "Add Docker support" feature.

Let's first build a very simple ConsoleApp. I used the "Console App (.NET Core)" template in Visual Studio 2017 RC and then setting it to target the NetCoreApp 1.1 Framework (project's Properties > Target framework). The app will display a random quote from our friend Duke Nukem. You can find the content of the quotes.json file [here](https://gist.github.com/Pvlerick/0765e5c6fc389444380aa44860de96e0).

```CSharp
using Newtonsoft.Json;
using System;
using System.IO;

namespace DockerDotNetCoreApp
{
    class Program
    {
        public static void Main(string[] args)
        {
            var content = JsonConvert.DeserializeObject<Content>(File.ReadAllText("quotes.json"));

            var r = new Random().Next(content.Quotes.Length);

            Console.WriteLine(content.Quotes[r]);
        }

        public class Content
        {
            public string[] Quotes { get; set; }
        }
    }
}
```

Next, we need to _publish_ the application by running the _dotnet publish_ command. Head to the command prompt in the root folder of the projet (not the solution) and run _dotnet publish_.
This will output all the required artifacts in the _bin\Debug\netcoreapp1.1\publish_ folder under the project, which is what we need to copy over in our Docker image to be able to run our application in there.

Now that this is done and that the app works, we can head on to Docker. After having switched to Windows Containers as I want my applications to stay in Windows' world, we need to find a suitable image to serve as a base for our container.
I have to admit that this was the hardest part for me, I had trouble with the information I found on [Docker Hub](https://hub.docker.com/) but in retrospect it's mostly my lack of understanding than bad documentation.

The image we need to base ourselves on is [microsoft/dotnet](https://hub.docker.com/r/microsoft/dotnet/). There are miryads of tags available, depending if you want to run a Debian based image or a Windows Nano Server one.

In our case, as we are running Docker with Windows Containers, we need an image that is based on Nano Server and we need to get the 1.1 runtime. Our base image is thus _microsoft/dotnet:1.1-runtime-nanoserver_.

We can finally write our Dockerfile:

```Dockerfile
FROM microsoft/dotnet:1.1-runtime-nanoserver

WORKDIR /app
COPY /bin/Debug/netcoreapp1.1/publish/ .

ENTRYPOINT ["dotnet", "DockerDotNetCoreApp.dll"]
```

My console application is called DockerDotNetCoreApp so the output of the compilation is _DockerDotNetCoreApp.dll_.

We can now build our image issuing the appropriate Docker command in the command prompt at the level of our Dockerfile (which is in our project):

```
docker build -t duke .
```

Which shoudl output somethingzzz like this:

```
Sending build context to Docker daemon 5.631 MB
Step 1/4 : FROM microsoft/dotnet:1.1-runtime-nanoserver
 ---> fcc5543479a5
Step 2/4 : WORKDIR /app
 ---> 5541a61032fd
Removing intermediate container 716e789ead5d
Step 3/4 : COPY /bin/Debug/netcoreapp1.1/publish/ .
 ---> 0cb87b4005af
Removing intermediate container e6ef28a8f892
Step 4/4 : ENTRYPOINT dotnet DockerDotNetCoreApp.dll
 ---> Running in 6469db72ce16
 ---> 3a1dfb37f3e3
Removing intermediate container 6469db72ce16
Successfully built 3a1dfb37f3e3
```

Now that our image is ready, we can run it (and name our container "dukec"):

```
docker run --name dukec duke
```

Which will display a random quote from Duke's repertoir (I got lucky not be insulted by my container):

> Nobody steals our chicks... and lives!

To get rid of the container and the image, simply use these commands:

```
docker rm dukec
docker rmi duke
```

Up next, same thing for an ASP.NET Core app targeting the .NET 4.6.2 Framework!

> Written with [StackEdit](https://stackedit.io/).
