---
title:  "Integration Test Postgres using testcontainers-net and GitHub Actions"
date:   2023-08-04
categories:
  - integrationtest
  - postgres
  - ci
tags:
  - C#
  - integrationtest
  - testing
  - github actions
  - testcontainers-net
---

This is a continuation of an earlier post [Integration Test Postgres with testcontainers-dotnet](https://kashifsoofi.github.io/aspnetcore/testing/integrationtest/postgres/integration-test-postgres-with-testcontainers-dotnet/). In this tutorial I will show you how I setup [GitHub Actions](https://github.com/features/actions) to run integration tests.

There is another post [Integration Test Postgres using GitHub Actions](https://kashifsoofi.github.io/integrationtest/postgres/ci/github-actions-integration-test-postgres/) that uses [GitHub Actions service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers) to setup postgres and run migrations as a step before running integration tests.

Also another post [Integration Test Postgres using docker-compose and GitHub Actions](https://kashifsoofi.github.io/integrationtest/postgres/ci/integration-test-postgres-using-docker-compose-and-github-actions/) uses docker-compose to setup Postgre database server and run migrations using `docker-compose.dev-env.yml` file we already have for development before running integration tests.

The difference here is we don't need to have a separate step to setup Postgres database server and run migrations as this is baked into our integration tests using [testcontainers-dotnet](https://github.com/isen-ng/testcontainers-dotnet).

## Workflow

### Name and filters
I started off by naming the workflow and defining which branches and paths to run it for.
```yaml
name: Integration Test Postgres (testcontainers-dotnet)

on:
  push:
  pull_request:
    branches: [ "main" ]
    paths:
     - 'integration-test-postgres-with-testcontainers-dotnet/**'
```

### build job
Next step is to define job, we only have a single job for this workflow that is named build. We will be running it on `ubuntu-latest` and we will set the default directory to where sample project is located, we would not need to set the default directory if the repo contains a single solution.
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: integration-test-postgres-with-testcontainers-dotnet
```

### Steps
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

### Run integration tests
Next and final step is to execute integration tests. This is done with following step
```yaml
steps:
  ...
  - name: Run integration tests
    run: dotnet test --configuration Release --no-restore --no-build --verbosity normal
```
Please note we have the connection string hardcoded in `DatabaseFixture`, code can be updated to be read that from environment variable to make it configurable. Also we don't need to worry about cleaning up containers that would be handled by the `DatabaseFixture` tear down step.


## Complete Workflow
Here is the complete workflow, also available at GitHub.
```yaml
name: Integration Test Postgres (testcontainers-dotnet)

on:
  push:
  pull_request:
    branches: [ "main" ]
    paths:
     - 'integration-test-postgres-with-testcontainers-dotnet/**'
    
jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: integration-test-postgres-with-testcontainers-dotnet

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
      - name: Run integration tests
        run: dotnet test --configuration Release --no-restore --no-build --verbosity normal
```

## Source
Source code for the demo application is hosted on GitHub in [blog-code-samples](https://github.com/kashifsoofi/blog-code-samples/tree/main/integration-test-postgres-with-testcontainers-dotnet) repository, source for this workflow is in [integration-test-postgres-testcontainers-dotnet.yml](https://github.com/kashifsoofi/blog-code-samples/blob/main/.github/workflows/integration-test-postgres-testcontainers-dotnet.yml).

## References
In no particular order  
* [REST API with ASP.NET Core 7 and Postgres](https://kashifsoofi.github.io/aspnetcore/rest/postgres/restapi-with-asp.net-core-7-and-postgres/)
* [Postgres Database](https://www.postgresql.org/)
* [Integration Testing](https://en.wikipedia.org/wiki/Integration_testing)
* [Docker](https://www.docker.com/)
* [GitHub Actions](https://github.com/features/actions)
* [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net)
* [About service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers)
* [testcontainers-dotnet](https://github.com/isen-ng/testcontainers-dotnet)
* And many more