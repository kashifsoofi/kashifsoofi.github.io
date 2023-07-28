---
title:  "Integration Test Postgres using docker-compose and GitHub Actions"
date:   2023-07-28
categories:
  - integrationtest
  - postgres
  - ci
tags:
  - C#
  - integrationtest
  - testing
  - github actions
  - docker-compose
---

This is a continuation of an earlier post [Integration Testing Postgres Store](https://kashifsoofi.github.io/aspnetcore/testing/integrationtest/postgres/postgres-store-integration-test/). In this tutorial I will show you how I setup [GitHub Actions](https://github.com/features/actions) to run integration tests using the `docker-compose.dev-env.yml` file.

There is another post [Integration Test Postgres using GitHub Actions](https://kashifsoofi.github.io/integrationtest/postgres/ci/github-actions-integration-test-postgres/) that uses [GitHub Actions service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers) to setup postgres before running integration tests.

## Workflow

### Name and filters
I started off by naming the workflow and defining which branches and paths to run it for.
```yaml
name: Integration Test Postgres (docker-compose)

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

### Steps
Next is to define the steps. I have started with the standard steps defined in [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net) guide. We checkout code, setup dotnet SDK, restore packages and build the solution in release mode.

An additional step is to setup the serivces our integration tests would be dependent on using the `docker-compose` and `docker-compose.dev-env.yml` file, which already contains the development dependencies required. I have taken the liberty to use the same docker-compose file we used during development, however a good practice is to create a separate file e.g. `docker-compose.ci.yml` to keep dependencies needed during CI (continuous integration) separate to what is required during development.
```yaml
steps:
  - uses: actions/checkout@v3
  - name: Start container and apply migrations
    run: docker compose -f "docker-compose.dev-env.yml" up -d --build
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
Next step is to execute integration tests. This is done with following step
```yaml
steps:
  ...
  - name: Run integration tests
    run: dotnet test --configuration Release --no-restore --no-build --verbosity normal
```
Please note we have the connection string hardcoded in `DatabaseFixture`, code can be updated to be read that from environment variable to make it configurable.

### Cleanup containers
Final step is to run `docker compose` command to shutdown running containers and cleanup images and volumes used by those containers.
```yaml
steps:
  ...
  - name: Stop containers
    run: docker compose -f "docker-compose.dev-env.yml" down --remove-orphans --rmi all --volumes
```

## Complete Workflow
Here is the complete workflow, also available at GitHub.
```yaml
name: Integration Test Postgres (docker-compose)

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

    steps:
      - uses: actions/checkout@v3
      - name: Start container and apply migrations
        run: docker compose -f "docker-compose.dev-env.yml" up -d --build
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
      - name: Stop containers
        run: docker compose -f "docker-compose.dev-env.yml" down --remove-orphans --rmi all --volumes
```

## Source
Source code for the demo application is hosted on GitHub in [blog-code-samples](https://github.com/kashifsoofi/blog-code-samples/tree/main/postgres-store-integration-test) repository, source for this workflow is in [integration-test-postgres-docker-compose.yml](https://github.com/kashifsoofi/blog-code-samples/blob/main/.github/workflows/integration-test-postgres-docker-compose.yml).

## References
In no particular order  
* [REST API with ASP.NET Core 7 and Postgres](https://kashifsoofi.github.io/aspnetcore/rest/postgres/restapi-with-asp.net-core-7-and-postgres/)
* [Postgres Database](https://www.postgresql.org/)
* [Integration Testing](https://en.wikipedia.org/wiki/Integration_testing)
* [Docker](https://www.docker.com/)
* [GitHub Actions](https://github.com/features/actions)
* [Building and testing .NET](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net)
* [About service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers)
* And many more