---
title: "Hands-on Lab"
description: "MassAuth Hands-on Lab."
lead: "Straight forward Hands-on Lab for StoneAssemblies.MassAuth"
date: 2020-11-16T13:59:39+01:00
lastmod: 2020-11-16T13:59:39+01:00
draft: false
images: []
menu:
  docs:
    parent: "massauth"
weight: 112
toc: true
---

## Prerequisites

- Visual Studio 2019 (16.9.3)
- Docker (2.3.0.4)
- [CakeBuild](https://cakebuild.net/) (1.1.0)
- [Tye](https://github.com/dotnet/tye/blob/main/docs/getting_started.md) (0.9.0-alpha.21380.1)


The complete and final source code of this **Hands-on Lab** is available on [GitHub](https://github.com/alexfdezsauco/StoneAssemblies.MassAuth.QuickStart). You can clone the repository or just follow the steps we describe below.

## Step 1: Setup the workspace

To setup the workspace, open a **PowerShell** console and run the following commands:

<script src="https://gist.github.com/alexfdezsauco/998e4f86c347c9b9d8da1cae9a2841bf.js"></script>

A Visual Studio solution file StoneAssemblies.MassAuth.QuickStart.sln is created after commands, which includes the following projects:

|                      Project                       |                             Description                              |
| -------------------------------------------------- | -------------------------------------------------------------------- |
| StoneAssemblies.MassAuth.QuickStart.**Messages**   | Class library for messages specification.                            |
| StoneAssemblies.MassAuth.QuickStart.**Rules**      | Class library to implement rules for messages.                       |
| StoneAssemblies.MassAuth.QuickStart.**Services**   | Web API to host the services that require to be authorized by rules. |
| StoneAssemblies.MassAuth.QuickStart.**AuthServer** | Authorization server to host the rules for messages.                 |

The commands also add the required NuGet packages and project references.

## Step 2: Contract first

Let's add a bit of complexity to the generated problem, related with weather forecast. For instance, let's say we will allow to request forecast from a certain date, as some forecasts may not be available due to the complexity of the calculations.

For that purpose, we will add the following class to the **Message** project, to request the weather forecast with start date as argument.

<script src="https://gist.github.com/alexfdezsauco/52e10bdad84c154a35cfd2be104d7333.js"></script>

## Step 3: Implementing rules

Now we are ready to implement some rules for the specified message. Continuing with our scenario, let's say the forecast data is only available from today and up to 10 days. This operation could be more complex through query to an external database, but for simplicity it will be implemented as follows in the **Rules** project.

<script src="https://gist.github.com/alexfdezsauco/6b5d4d0b37c3351cb34ddb2f008a3c76.js"></script>

## Step 4: Implementing services

It's time to complete the *WeatherForecastController* implementation in the **Services** project. It should look like this. 

<script src="https://gist.github.com/alexfdezsauco/cfde54e975924aaf39ff3d7664b7fdb2.js"></script>

Notice the usage of *AuthorizeByRule* attribute on the Get method, to indicate that the input message *WeatherForecastRequestMessage* must be processed and validated by the authorization engine before the method execution.

We also have to update the *Startup* class implementation.

<script src="https://gist.github.com/alexfdezsauco/86b681686642537c8d945149031a2302.js"></script>

Basically, we add *AuthorizeByRuleFilter* as scoped service and ensure the communication through the message broker. Remember, StoneAssemblies.MassAuth is built on top of MassTransit.  Finally, to ensure the configuration via environment variables we must update the *Program* class to this. 

<script src="https://gist.github.com/alexfdezsauco/2d6e33f15ee7077577727498a387e8c3.js"></script>

## Step 5: Hosting rules

In order to host rules, we provide a production ready of *StoneAssemblies.MassAuth.Server* as docker image available in [DockerHub](https://hub.docker.com/r/stoneassemblies/massauth-server). But for debugging or even for customization purpose could be useful build your own rule host server. So, in the **AuthServer** project we also have to update the *Startup* class implementation.

<script src="https://gist.github.com/alexfdezsauco/6384b11ed441d6efb06950e5e6babdc9.js"></script>

Again, to ensure the configuration via environment variables the *Program* file must be updated like this.

<script src="https://gist.github.com/alexfdezsauco/9cd40c62c6efba8fc164d73dafe3b117.js"></script>

## Step 6: Build, run and test

In order to build and run the project, open a **PowerShell** terminal in the working directory and run following commands.

```PowerShell
> dotnet cake
> cd deployment/tye
> tye run
```

Open your browser and navigate to [http://localhost:8000](http://localhost:8000) to display the Tye user interface and ensure that everything is working. 


Open a new **PowerShell** terminal and try the following commands: 

**Input**: Valid request

```PowerShell
Invoke-WebRequest http://localhost:6001/WeatherForecast?StartDate=$([System.DateTime]::Now.AddDays(1).Date)
```

**Output:**
```
StatusCode        : 200
StatusDescription : OK
Content           : [{"date":"2021-08-24T00:00:00","temperatureC":21,"temperatureF":69,"summary":"Chilly"},{"date":"2021-08-25T00:00:00","temperatureC":4,"tem
                    peratureF":39,"summary":"Chilly"},{"date":"2021-08-26T00:00:00...
RawContent        : HTTP/1.1 200 OK
                    Transfer-Encoding: chunked
                    Content-Type: application/json; charset=utf-8
                    Date: Sun, 22 Aug 2021 18:38:21 GMT
                    Server: Kestrel

                    [{"date":"2021-08-24T00:00:00","temperatureC":21,"te...
Forms             : {}
Headers           : {[Transfer-Encoding, chunked], [Content-Type, application/json; charset=utf-8], [Date, Sun, 22 Aug 2021 18:38:21 GMT], [Server, Kestrel]}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 435
```

**Input**: Out of range request

```PowerShell
Invoke-WebRequest http://localhost:6001/WeatherForecast?StartDate=$([System.DateTime]::Now.AddDays(11).Date)
```

**Output:**
```
Invoke-WebRequest : The remote server returned an error: (401) Unauthorized.
At line:1 char:1
+ Invoke-WebRequest http://localhost:6001/WeatherForecast?StartDate=$([ ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (System.Net.HttpWebRequest:HttpWebRequest) [Invoke-WebRequest], WebException
    + FullyQualifiedErrorId : WebCmdletWebResponseException,Microsoft.PowerShell.Commands.InvokeWebRequestCommand
```

