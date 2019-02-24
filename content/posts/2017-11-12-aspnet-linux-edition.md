---
title: "ASP.NET Core. Linux Edition"
date: "2017-11-12"
template: "post"
draft: false
slug: "/posts/aspnet-linux-edition/"
category: "ASP.NET Core"
tags:
  - "ASP.NET Core"
  - "Linux"
  - "Docker"
description: "ASP.NET Core + Linux = Freedom. Love ASP.NET? What about Linux? Do you wish to bring the two together? Well now you can with ASP.NET core. Deploy your favorite web stack to your favorate hosting environment. Or, develop from the comfort of your favorite MacBook or Linux box."
---

[ASP.NET Core](https://www.microsoft.com/net/learn/apps/web); When the cross-platform version of the infamous ASP.NET we all love (to hate?) was released I immediately began exploring the process of deploying a .NET web application to a Linux host. After all, I was already running several [Node.js](https://nodejs.org) web applications on a Linux box so why not add a few ASP.NET applications and host all apps under one roof. Plus I had to see it to believe it; a Microsoft web app running in a Linux environment!

In this article, I walk through the process of deploying an ASP.NET application to a Linux environment. The application will run inside a [Docker](https://www.docker.com/) container within a Ubuntu host. The Docker environment is managed with [Dokku](http://dokku.viewdocs.io/dokku/) which makes it possible to kick off builds and deployments with simple git pushes. Dokku will also handle (sub)domain routing by managing [nginx](https://www.nginx.com/); good riddance.

**Update:** The final source code lives [here](https://github.com/dworthen/aspnet-linux-edition/tree/1.0.0).

---

## Setup

1. [Download and install dotnet](https://dot.net) on your development environment.
2. [Install Dokku on a fresh Ubuntu environment](http://dokku.viewdocs.io/dokku/getting-started/installation/).
   - I am using [Linode](https://www.linode.com/) to host a Ubuntu 16.04 LTS environment. Instructions for installing Dokku on Linode can be found [here](http://dokku.viewdocs.io/dokku~v0.6.2/getting-started/install/linode/).
   - Alternatively, use [Windows Subsystem for Linux](https://msdn.microsoft.com/en-us/commandline/wsl/install-win10) to explore this process on a local machine.
3. [Optional] Point a [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) towards the Ubuntu host. If using Linode, as I am, then check out their [DNS Manager](https://www.linode.com/docs/networking/dns/dns-manager-overview).
   - For this article, I have both derekworthen.com, `<SERVER_ADDRESS>`, and blog-demo-aspnet-linux-environment.derekworthen.com, `<FQDN>`, pointing towards my Dokku configured server. `<APP_NAME>` refers to blog-demo-aspnet-linux-environment.

---

## App Creation

Since this article is focused on the process of deploying to a Ubuntu host we will stick with creating a bare-bones web application. Visit Microsoft docs for an in-depth guide to [getting started with ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/mvc/razor-pages/?tabs=visual-studio).

#### .NET Core CLI

At the time of this writing, I am using .Net CLI tools and SDK version 2.0.2 to create a sample app but any version of .NET Core should work.

**Note:** Run `dotnet --info` to print out the version of dotnet CLI tools and SDK installed.

```Bash
mkdir <APP_NAME> && cd <APP_NAME> && dotnet new razor
```

Verify the app builds and runs with `dotnet restore && dotnet run`. Head over to localhost:5000 for a front row viewing of the out-of-the-box ASP.NET template in all its glory.

![Default ASP.NET application running on port 5000](/media/aspnetcore-app.png)

---

## App Management with Dokku

Shell into the Ubuntu host as the dokku user account created during the setup process. I am using the [Windows Subsystem for Linux](https://msdn.microsoft.com/en-us/commandline/wsl/install-win10) to drop into Bash and connect to the server instead of using an application like [Putty](http://www.putty.org/). Mac and Linux users can ignore the `bash -c` part of the command and delete the quotes surrounding the `ssh` command.

```Bash
bash -c "ssh dokku@<SERVER_ADDRESS>"
```

#### Dokku

```Bash
# Linux host
$ dokku apps:create <APP_NAME>
```

This will create a git repo at `<SERVER_ADDRESS>:<APP_NAME>`, e.g., derekworthen\.com:blog-demo-aspnet-linux-environment in my case. Dokku will also register a reverse proxy routing with nginx using the `<APP_NAME>` as the subdomain to create the app's `<FQDN>`, e.g., blog-demo-aspnet-linux-environment.derekworthen.com. By default, Dokku will configure nginix to route all traffic on port 80 and 443 (if SSL is enabled) that matches the `<FQDN>` to port 5000 of the app's docker container.

#### Git

On the local machine, initialize git and add a remote location for the newly created dokku repo. (Don't forget to add an [appropriate](https://www.gitignore.io/api/aspnetcore) [.gitignore](https://www.gitignore.io/) file to the project.)

```Bash
# Local development machine
git init && git add . && git commit -m "initial commit"
# Must use the dokku user account created during the setup process
git remote add dokku dokku@<SERVER_ADDRESS>:<APP_NAME>
```

Up until now, we have been following the [basic guide for deploying a Dokku app](http://dokku.viewdocs.io/dokku/deployment/application-deployment/). We will now follow the instructions for [Dockerfile deployment](http://dokku.viewdocs.io/dokku/deployment/methods/dockerfiles/). Dokku will build, run and deploy a docker container based on the provided Dockerfile.

#### Dockerfile; Building and Running ASP.NET Core Applications

Create a `Dockerfile` in the root of the application with the following contents.

```Dockerfile
# Stage 1. Building.
# May need use a different tag to get the build to work.
# https://hub.docker.com/r/microsoft/aspnetcore-build/
FROM microsoft/aspnetcore-build:2.0.2
WORKDIR /source

# caches restore result by copying csproj file separately
# copies project files into the /source/ of the container.
COPY *.csproj .

# Restore dotnet dependencies
RUN dotnet restore

# copies the rest of your code into /source/
COPY . .

# microsoft/aspnetcore-build image contains Node
# so may also run npm install
# or other npm related scripts.
# RUN npm install
# RUN npm run build

# Builds to /out in the root of the container
RUN dotnet publish --output /out/ --configuration Release

# Set environment variables for container
# Dokku maps external port 80 to internal port 5000 and
# the following lines tells ASP.NET to listen on port 5000
# May also omit the following line and add EXPOSE 80
# This will tell dokku to setup port mapping 80:80 instead 80:5000
# and omitting this environment variable will also tell ASP.NET Core
# to listen on port 80
ENV ASPNETCORE_URLS http://*:5000
# EXPOSE 80

# Stage 2. Running the application
WORKDIR /out
ENTRYPOINT ["dotnet", "<APP_NAME>.dll"]
```

The above Dockerfile provides a runtime environment along with tools to build from source code. In a future article, I will cover deploying to a Dokku configured server using a CI/CD pipeline where the build artifacts are created prior to the Docker image. In such scenario, the Docker container does not need build tools and can be imaged from `microsoft/aspnetcore` for the runtime environment.

#### Deploy

It is finally time to push the project to the hosting server with git and watch Dokku build and run the application!

```Bash
git add Dockerfile && git commit -m "Add Dockerfile"
git push dokku master
```

If all goes well, Dokku will build and deploy a docker container housing our beautiful ASP.NET application and expose it to the world at the configured `<FQDN>`, e.g., http://blog-demo-aspnet-linux-environment.derekworthen.com/.

![Deploying a Dokku app](/media/deploy-dokku-app.png)

## Conclusion

Using Dokku as a simple [PaaS](https://en.wikipedia.org/wiki/Platform_as_a_service), we managed to deploy an ASP.NET Core web application to a Ubuntu server bundled as a Docker container. Now that the app creation and configuration is out of the way, changes pushed to the dokku remote location will rebuild and redeploy the application.

---

## Bonus; SSL Encryption

With the [Lets Encrypt](https://letsencrypt.org/) [Dokku plugin](https://github.com/dokku/dokku-letsencrypt), this could not be easier.

#### Install

```Bash
# Need to be logged in as a sudo user instead of the dokku user account
$ sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
```

#### Configure

```Bash
$ dokku config:set --no-restart <APP_NAME> DOKKU_LETSENCRYPT_EMAIL=<EMAIL>
```

#### Enable

```Bash
$ dokku letsencrypt <APP_NAME>
```

#### Auto-renew

```Bash
$ dokku letsencrypt:cron-job --add
```

#### Verify

```Bash
$ dokku proxy:report <APP_NAME>
```

Look for a https:443:5000 port mapping in the output.

#### Test

Open your favorite web browser and navigate to the https (or http as it should redirect to https) version of your site.
