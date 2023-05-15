---
title:  "AspNet Core WebApp behind Kong Gateway (or any other proxy)"
date:   2023-05-15
categories:
  - aspnetcore
  - rest
tags:
  - C#
  - aspnetcore
  - rest
  - api
---

## What is a REST API?
An API, or application programming interface, is a set of rules that define how applications or devices can connect to and communicate with each other. A REST API is an API that conforms to the design principles of the REST, or representational state transfer architectural style. For this reason, REST APIs are sometimes referred to RESTful APIs.  

Focus of this tutorial to write a REST API using C# .NET.

## Create Web API Project
I am using Visual Studio 2022 for Mac on an Intel MacBook Pro.
1. Select New -> App -> API project
<figure>
  <a href="/assets/images/2023-05-15/001-select-template.png"><img src="/assets/images/2023-05-15/001-select-template.png"></a>
  <figcaption>Select API Project Template.</figcaption>
</figure>  
2. Configure API as below, I have not configured to use HTTPS for simplicity
<figure>
  <a href="/assets/images/2023-05-15/002-configure-api.png"><img src="/assets/images/2023-05-15/002-configure-api.png"></a>
  <figcaption>Configure API Project.</figcaption>
</figure>  
3. Name your project, I prefer to name my API project as [AggregateName].Api and name solution as [Aggregate Name], this comes from the Domain Driven Design, for this tutorial we are creating a REST API for Movies.
<figure>
  <a href="/assets/images/2023-05-15/003-name-project.png"><img src="/assets/images/2023-05-15/003-name-project.png"></a>
  <figcaption>Name API Project.</figcaption>
</figure>  
4. Click the `Create` button would create the project.
5. Right click on Dependencies and update nuget packages in Solution explorer.
6. Clicking Run button would run the API and show the following
<figure>
  <a href="/assets/images/2023-05-15/004-swagger-ui.png"><img src="/assets/images/2023-05-15/004-swagger-ui.png"></a>
  <figcaption>Movies API Swagger UI.</figcaption>
</figure>  

## Cleanup API Project
- Remove default WeatherForecast Model
- Remove default WeatherForecastController
- Click Add -> New File and select `Web API Controller Class`
- Name new contoller as MoviesController
- Click Run button to run the API and you should see Movies endpoints on Swagger UI
- Add another controller

## Add HealthChecks
Here is an excellent [article](Adding health checks with Liveness, Readiness, and Startup probes) on Liveness, Readiness and Startup health checks in ASP.NET Core.

Lets add a health check to our service.

- Register health check services
```
builder.Services.AddHealthChecks();
```
- Map health checks endpoints
```
app.MapHealthChecks("/healthz");
```

## Source
Source code for the demo application, and docker-compose files are hosted on GitHub in [movies-api-cs](https://github.com/kashifsoofi/movies-api-cs) repository.  

## References
In no particular order  
[What is a REST API?](https://www.ibm.com/topics/rest-apis)  
[Adding health checks with Liveness, Readiness, and Startup probes](https://andrewlock.net/deploying-asp-net-core-applications-to-kubernetes-part-6-adding-health-checks-with-liveness-readiness-and-startup-probes/)  
[Health checks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-7.0)  
And many more