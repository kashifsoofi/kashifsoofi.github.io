---
title:  "REST API using C# .NET 7 with Postgres"
date:   2023-05-18
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
    volumes:
      - moviesdbdata:/var/lib/postgresql/data/
    ports:
      - “5432:5432”
    restart: on-failure
    healthcheck:
      test: [ “CMD-SHELL”, “pg_isready -q -d $${POSTGRES_DB} -U $${POSTGRES_USER}“]
      timeout: 10s
      interval: 5s
      retries: 10

volumes:
  moviesdbdata:
```

Open a terminal at the root of the solution where docker-compose file is location and execute following command to start database server.
```shell
docker-compose -f docker-compose.dev-env.yml up -d
```

## Database Migrations
Before we can start using Postgres we need to create a table to store our data. I will be using excellent [roundhouse](https://github.com/chucknorris/roundhouse) database deployment system to execute database migrations.

I usually create a container that has all database migrations and tool to execute those migrations. I name migrations as [yyyyMMdd-HHmm-migration-name.sql] but please feel free to use any naming scheme, keep in mind how the tool would order multiple files to run those migrations. I have also added a `wait-for-db.csx` file that I would use as the entry point for database migrations container. This is a `dotnet-script` file and would be run using [dotnet-script](https://github.com/dotnet-script/dotnet-script). I have pinned the versions that are compatible with .net sdk 3.1 as this the version `roundhouse` is build against at the time of writing.

For migration, I have added following under `db\up` folder.
- `20230518_1800_extension_uuid_ossp_create.sql`
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```
- `20230518_1801_table_movies_create.sql`
```sql
CREATE TABLE IF NOT EXISTS movies (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    title VARCHAR(100) NOT NULL,
    director VARCHAR(100) NOT NULL,
    release_date TIMESTAMP NOT NULL,
    ticket_price DECIMAL(12, 2) NOT NULL,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT (now() AT TIME ZONE 'utc') NOT NULL,
    updated_at TIMESTAMP WITHOUT TIME ZONE DEFAULT (now() AT TIME ZONE 'utc') NOT NULL
)
```

Open a terminal at the root of the solution where docker-compose file is location and execute following command to start database server and apply migrations to create `uuid-ossp` extension and `movies` table.
```shell
docker-compose -f docker-compose.dev-env.yml up -d
```

## Postgres Movies Store
I will be using [Dapper](https://github.com/DapperLib/Dapper) - a simple object mapper for .Net along with [Npgsql](https://www.npgsql.org/doc/index.html).

## Test
I am not adding any unit or integration tests for this tutorial, perhaps a following tutorial. But all the endpoints can be tested either by the Swagger UI by running the application or using Postman.

## Source
Source code for the demo application is hosted on GitHub in [movies-api-cs](https://github.com/kashifsoofi/movies-api-cs/tree/rest-api-with-postgres) repository.

## References
In no particular order  
* [REST API using C# .NET 7 with InMemory Store](https://kashifsoofi.github.io/aspnetcore/rest/aspnetcore-restapi/)
* [Postgres Database](https://www.postgresql.org/)
* [roundhouse](https://github.com/chucknorris/roundhouse)
* [dotnet-script](https://github.com/dotnet-script/dotnet-script)
* [Dapper](https://github.com/DapperLib/Dapper)
* [Npgsql](https://www.npgsql.org/doc/index.html)
* And many more