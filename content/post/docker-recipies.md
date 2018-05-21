---
title: "Docker Recipies"
date: 2018-05-18T14:38:10-04:00
lastmod: 2018-05-18T14:38:10-04:00
draft: false
keywords: []
description: "Handy recipies when working with docker"
tags: ["docker"]
categories: ["docker"]
author: "Wes Shaddix"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---

# Handy recipies when working with docker
The more you work with docker the more little tips and tricks you pick up on to make your life easier. These are some of the more useful things that I've learned along the way.
<!--more-->
## Building Images from a Dockerfile

### 1. Optimizing the Build Context

#### Scenario
I typically build .net core microservice images using multi-stage builds. In the first stage, where I have my SDK tools installed, I'm typically copying over the solution (.sln), project (.csproj) and nuget (nuget.config) files. After that I restore my project dependencies (nuget restore) and then I'll copy in *everything else* before I publish the application.

#### Problem
If you don't have a `.dockerignore` file, then when you copy *everything else* you'll get a lot more than you need copied into your docker build context which is wasteful for both the image building process as well as your resulting image contents.

#### Solution
After you copy in *everything else*, but before you build/publish your code (just comment those lines out in your `Dockerfile`) you can create a temporary container and login to view what you've actually copied into the build context. The easiest way I've found to do this is to grab the `id` of the layer from the `docker build` output and create a temporary container from that intermidiate layer to view the contents of the filesystem.

`docker run --entrypoint sh -it --rm <layer id>`

`/app # find (this will show you the files that you copied from the build context to the filesystem)`

You can use this list of files to guide you as you ignore them and the unnecessary directories in your `.dockerignore` file. Repeat until it's optimized with no wasted files or directories.

### 2. .Net Core .dockerfile
My typical starting point for a `.dockerignore` file for a .net core application hosted on bitbucket is:
``` bash
.dockerignore
.gitignore
bitbucket-pipelines.yml
docker-compose.yml
Dockerfile
README.md
**/*/bin*
**/*/obj*
.git
.vs
```

### 3. .Net Core Dockerfile
My typical starting point for a `Dockerfile` for a .net core application is:
``` docker
# STAGE 1 - COMPILE AND PUBLISH

# use the image that has the build tools for the "build stage"
FROM microsoft/dotnet:2.1-sdk-alpine AS build-env

# set the working directory in the image as "app"
WORKDIR /app

# copy the solution, project and nuget files
COPY <SOLUTION>.sln ./<SOLUTION>.sln
COPY global.json ./global.json
COPY ./src/<PROJECT>.csproj ./src/<PROJECT>.csproj
COPY ./tests/<TESTS>.csproj ./tests/<TEST>.csproj
COPY NuGet.config .

# restore the nuget packages (cache this layer since it doesn't change often)
RUN dotnet restore

# copy the rest of the code
COPY . ./

# publish
RUN dotnet publish -c Release -o /publish ./<SOLUTION>.sln

# STAGE 2 - BUILD RUNTIME OPTIMIZED IMAGE

# use the runtime optimized image that does not have any build tools, only the .net core runtime
FROM microsoft/dotnet:2.1-runtime-alpine

# set the working directory in the image as "app"
WORKDIR /app

# copy the compiled code from the published output folder of the "build stage" into this image
COPY --from=build-env /publish .

# set the default command that will run when this image is ran
ENTRYPOINT [ "dotnet", "<PROJECT>.dll"]

```
