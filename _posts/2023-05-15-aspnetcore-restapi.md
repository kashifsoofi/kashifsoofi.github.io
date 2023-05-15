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


## Source
Source code for the demo application, and docker-compose files are hosted on GitHub in [movies-api-cs](https://github.com/kashifsoofi/movies-api-cs) repository.  

## References
In no particular order  
[What is a REST API?](https://www.ibm.com/topics/rest-apis)  
[Konga]https://github.com/pantsel/konga)  
[docker-compose.yml for kong, postgres and konga](https://gist.github.com/pantsel/73d949774bd8e917bfd3d9745d71febf)  
[Google](https://www.google.com)  
[How to Update a Single Running docker-compose Container](https://staxmanade.com/2016/09/how-to-update-a-single-running-docker-compose-container/)  
And many more