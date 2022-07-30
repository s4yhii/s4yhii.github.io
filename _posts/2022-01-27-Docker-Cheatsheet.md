---
title: Docker Cheatsheet
path: 2022-01-27 12:00:00 -0400
image: 
    src: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/DevOps/Docker/docker1.jpg
    height: 1100
    width: 500
categories: [DevOps, Docker]
tags: [cheatsheet, docker]
---

"With Docker, developers can build any app in any language using any toolchain. “Dockerized” apps are completely portable and can run anywhere - colleagues" OS X and Windows laptops, QA servers running Ubuntu in the cloud, and production data center VMs running Red Hat.

## Basic Commands
Verified cli can talk to engine
```bash
docker version
```

Most config values of engine
```bash
docker info
docker ps #see al docker running
docker top <name> #see info about the container
```

Docker command line structure
```bash
docker <command> (options) #old (still works)
docker <command> <sub-command> (options) #new
```
Deploy a nginx server

```bash
docker container run --publish 80:80 nginx
```

List all container running 
```bash
docker container ls <options>
```

| Syntax        | Description                                |
|:------------- |:-------------------------------------------|
| --all, -a     | Show all containers (default show running) |
| --filter, -f  | Filter output based on conditions          |
| --format      | Pretty-print containers using Go templates |
| --last, -n    | Show n last creates containers             |
| --latest, -l  | Show the latest created container          |
| --no-trunc    | Dont truncate output                       |
| --quiet, -q   | Only display container ids                 |
| --size, -s    | Display total file sizes                   |

Stop the container process but not remove it
```bash
docker container stop <id>
```
Assign a name to a container
```
docker container run --publish 80:80 --name webhost nginx
```

Show logs for a specific container
```bash
docker container logs 
```
Remove many containers together
```bash
docker container rm <id> <id> <id> -f #use f for stop the container before
```
