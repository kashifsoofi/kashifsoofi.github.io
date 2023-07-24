---
title:  "Integration Test Postgres using GitHub Actions"
date:   2023-07-21
categories:
  - integrationtest
  - postgres
  - ci
tags:
  - C#
  - integrationtest
  - testing
  - github actions
  - postgres
---

This is a continuation of an earlier post [Integration Testing Postgres Store](https://kashifsoofi.github.io/aspnetcore/testing/integrationtest/postgres/postgres-store-integration-test/). In this tutorial I will show you how I setup [GitHub Actions](https://github.com/features/actions) to run integration tests when a change is made to the repository. 

Prior to this sample, only way to run those integration tests was to start database server, apply migrations and run the tests either using VisualStudio or dotnet command line. This automates running those tests and ensures that tests are successful not only on developer's machine.

## Workflow

### Name and filters
I started off by naming the workflow and defining which branches and paths to run it for.
```yaml
name: Integration Test Postgres

on:
  push:
    branches: [ "main" ]
    paths:
     - 'postgres-store-integration-test/**'
```

### build job
Next step is to define job, we only have a single job for this workflow that is named build. We will be running it on `ubuntu-latest` and we will set the default directory to where sample project is located, we would not need to set the default directory if the repo contains a single solution.
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: postgres-store-integration-test
```

### services
Next is to define the services our tests are dependent on, in this case its a `postgres` database server. We will leavarage [service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers) to run `postgres` container, setting up username, password, database and exposing port so that we can access it on Docker host.
```yaml
services:
      postgres:
        image: postgres:14-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: Password123
          POSTGRES_DB: moviesdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
```

### Build solution
Next is to define the steps. I have started with the standard steps defined in [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net) guide. We checkout code, setup dotnet SDK, restore packages and build the solution in release mode.
```yaml
steps:
  - uses: actions/checkout@v3
  - name: Setup .NET Core SDK
    uses: actions/setup-dotnet@v3
    with:
        dotnet-version: 7.0.x
  - name: Install dependencies
    run: dotnet restore
  - name: Build
    run: dotnet build --configuration Release --no-restore
```

### Apply Migrations
Before we can run our integration tests, we need to apply database migrations so that database has all the necessary tables our code and tests are dependent on. We already have a Dockerfile to build a container that can run migrations, so I have added a step to build that container and a step to apply those migrations to database running as a container under services.

Please note the connection string, its connecting migrations container to host where our database service is accessible on default port.
```yaml
steps:
  ...
  - name: Build migratinos Docker image
    run: docker build --file ./db/Dockerfile -t movies.db.migrations ./db
  - name: Run migrations
    run: docker run --add-host=host.docker.internal:host-gateway movies.db.migrations "Host=host.docker.internal;Username=postgres;Password=Password123;Database=moviesdb;Integrated Security=false;
```

### Run integration tests
Final step is to execute integration tests. This is done with following step
```yaml
steps:
  ...
  - name: Run integration tests
    run: dotnet test --configuration Release --no-restore --no-build --verbosity normal
```
Please note we have the connection string hardcoded in `DatabaseFixture`, code can be updated to be read that from environment variable to make it configurable.

## Complete Workflow
Here is the complete workflow, also available at GitHub.
```yaml
name: Integration Test Postgres

on:
  push:
    branches: [ "main" ]
    paths:
     - 'postgres-store-integration-test/**'

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: postgres-store-integration-test

    services:
      postgres:
        image: postgres:14-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: Password123
          POSTGRES_DB: moviesdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3
      - name: Setup .NET Core SDK
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: 7.0.x
      - name: Install dependencies
        run: dotnet restore
      - name: Build
        run: dotnet build --configuration Release --no-restore
      - name: Build migratinos Docker image
        run: docker build --file ./db/Dockerfile -t movies.db.migrations ./db
      - name: Run migrations
        run: docker run --add-host=host.docker.internal:host-gateway movies.db.migrations "Host=host.docker.internal;Username=postgres;Password=Password123;Database=moviesdb;Integrated Security=false;"
      - name: Run integration tests
        run: dotnet test --configuration Release --no-restore --no-build --verbosity normal
```

## Source
Source code for the demo application is hosted on GitHub in [blog-code-samples](https://github.com/kashifsoofi/blog-code-samples/tree/main/postgres-store-integration-test) repository, source for this workflow is in [integration-test-postgres.yml](https://github.com/kashifsoofi/blog-code-samples/blob/main/.github/workflows/integration-test-postgres.yml).

## References
In no particular order  
* [REST API with ASP.NET Core 7 and Postgres](https://kashifsoofi.github.io/aspnetcore/rest/postgres/restapi-with-asp.net-core-7-and-postgres/)
* [Postgres Database](https://www.postgresql.org/)
* [Integration Testing](https://en.wikipedia.org/wiki/Integration_testing)
* [Docker](https://www.docker.com/)
* [GitHub Actions](https://github.com/features/actions)
* [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net)
* [About service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers)
* [Creating PostgreSQL service containers](https://docs.github.com/en/actions/using-containerized-services/creating-postgresql-service-containers)
* And many more