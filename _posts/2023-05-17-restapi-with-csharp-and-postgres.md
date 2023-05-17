---
title:  "REST API using C# .NET 7 with Postgres"
date:   2023-05-17
categories:
  - aspnetcore
  - rest
  - postgres
tags:
  - C#
  - aspnetcore
  - rest
  - api
---

This is a continuation of an earlier post [REST API using C# .NET 7 with InMemory Store](https://kashifsoofi.github.io/aspnetcore/rest/restapi-with-csharp/). In this tutorial I will extend the service to store data in a [Postgres Database](https://www.postgresql.org/). I will use [Docker](https://www.docker.com/) to run Postgres and use the same to run database migrations.

## Setup Database Server
I will be using a docker-compose to run Postgres in a docker container. This would allow us the add more services that our rest api is depenedent on e.g. redis server for distributed caching.

Let's start by adding a new file by right clicking on Solution name in Visual Studio and Add New File. I like to name file as `docker-compose.dev-env.yml`, feel free to name it as you like. Add following content to add a database instance for movies rest api.
```yaml
version: '3.7'

services:
  movies.db:
    image: postgres:14-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=Password123
      - POSTGRES_DB=moviesdb
    ports:
      - “5432:5432”
    restart: on-failure
    healthcheck:
      test: [ “CMD-SHELL”, “pg_isready -q -d $${POSTGRES_DB} -U $${POSTGRES_USER}“]
      timeout: 10s
      interval: 5s
      retries: 10
```

Open a terminal at the root of the solution where docker-compose file is location and execute following command to start database server.
```shell
docker-compose -f docker-compose.dev-env.yml up -d
```

## Test
I am not adding any unit or integration tests for this tutorial, perhaps a following tutorial. But all the endpoints can be tested either by the Swagger UI by running the application or using Postman.

## Source
Source code for the demo application is hosted on GitHub in [movies-api-cs](https://github.com/kashifsoofi/movies-api-cs/tree/rest-api-with-postgres) repository.

## References
In no particular order  
* [REST API using C# .NET 7 with InMemory Store](https://kashifsoofi.github.io/aspnetcore/rest/aspnetcore-restapi/)
* [Postgres Database](https://www.postgresql.org/)  
* And many more