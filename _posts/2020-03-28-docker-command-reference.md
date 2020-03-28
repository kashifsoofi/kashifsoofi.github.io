---
title:  "Docker, Docker-Compose Command Reference"
date:   2020-03-28
categories:
  - container
tags:
  - docker
  - docker-compose
---
In this article I will document some of the docker/docker-compose commands I have come to use most frequently. The purpose is more to serve as a personal reference.

## Docker
### Build Docker Image
As this is docker command reference, writting Dockerfile is not the scope of this article but in order for following commands to work there should be at least a Dockerfile in current directory.  
`docker build .`  
Specify dockerfile instead of using default Dockerfile  
`docker build -f Dockerfile.custom .`  
Or use Dockerfile from a different path  
`docker build -f ./src/Dockerfile .`  
Tag docker image  
`docker build -f Dockerfile -t myimage`  

### Run docker container

Run MySql 5.6 in Docker container mapping host port 3306 to container
`docker run --name dev-mysql -p 3306:3306 -v mysql-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=[ROOT_PASSWORD] -d mysql:5.6`  

### Stop all docker containers
`docker stop $(docker ps -a -q)`  

### Remove all docker containers
`docker rm $(docker ps -a -q)`  

### Remove all intermediate/dangling containers
`docker rmi $(docker images --filter "dangling=true" -q --no-trunc)`  
Combine this with stop all containers to remove any running instances.  

## Docker Compose
### Run all containers
The following command would use docker-compose.yml file in current directory
`docker-compose up`  
Run in background  
`docker-compose up -d`  
Use a custom docker-compose file  
`docker-compose -f docker-compose.custom.yml up`  
Or run in background  
`docker-compose -f docker-compose.custom.yml up -d`  

### Run single container from docker-compose
`docker-compose run app1`  
or with custom file  
`docker-compose -f docker-compose.custom.yml run app1`

### Stop containers
`docker-compose stop`    
or with custom file  
`docker-compose -f docker-compose.custom.yml stop`  

### Stop and remove containers and network
`docker-compose down`  
or with custom file  
`docker-compose -f docker-compose.custom.yml down`  
You can go 1 step further and remove volumns as well  
`docker-compose -f docker-compose.custom.yml down -v`  
While development I ususally use the following command, it will remove any images that are built locally  
`docker-compose -f docker-compose.custom.yml down -v --rmi local --remove-orphans`  

## References
Countless...