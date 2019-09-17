# More Docker Magic

Well, not exactly magic. According to Clarke's Law, any sufficiently advanced
technology is indistimguishable from magic, so that is what we are dealing with
here. I was struggling to come up with a more advanced example for tonight, but
seripendipity came to my aid. Last month, I think I mentioned that Rancher Labs
was hosting a *Rancher Rodeo* in Ottawa and that happened yesterday. While
going through the introductory scenario for Kubernetes under Rancher, we used a
service I think I have heard of in the past, but never really utilized -
xip.io.

As a result, I solved the problem that was holding me back from a number of
potential talks and we can all take advantage of it. More about that later.

With a little work, you will be able to duplicate the environment we are using
to display the information for tonight.

## What are we doing?

We are all going to have a self-hosted git server with most of the features of
Github or Gitlab, running in a Docker instance on our laptops, and the ability to
reach anybody's server by DNS name from the local network. It should be fun and
somewhat entertaining.

"How can that work?" you all ask sceptically.

"Well, that is why you are all here tonight. To learn a few new things." I
reply with a maniacal laugh! **[cue maniaical laugh]**

Seriously though, the objective is to create a Docker Compose file that will
assemble all of the requirements for such a service, set up the appriopriate
services, expose them through the host computer and get you started on a simple
web capable git service. This will be suitable for use on a Synology and other
environments, so it does have practical applications.

Of course, this does require the environment from last month to still be on
your systems. We will be building on the base system from last month and
creating a new environment.

What will we be using for all of this?

- Docker
- Docker Compose
- Gitea
- PostgreSQL
- Traefik Reverse Proxy (we could also use nginx)
- Ubuntu
- Xip.io service

### What is Gitea?

Gitea is a fork of Gogs, the easy to use self-hosted Git service. It is similar
to GitHub, Bitbucket, and Gitlab. 

Gitea is:
- A lightweight code hosting solution written in Go.
- Can run on minimal hardware requirements. 
- It is cross-platform, so it can run anywhere Go can be compiled such as Windows, Linux, MacOS,
ARM etc.

## Objectives

You will:
- Create the basic infrastructure for your new container
- Author a suitable Docker Compose file (jind of an advanced Dockerfile)
- Install and configure the lightweight Git service Gitea.
- Deploy the Gitea server using Docker be using the PostgreSQL database and Traefik Reverse proxy.


## Requirements

- Full Docker install
- A working system based on last month's talk
- A little free space for the new container
- Your IP address that you are using tonight.

## Getting started

Verify you still have a Docker command line working: `docker version` 

```
➜  ~ docker version
Client: Docker Engine - Community
 Version:           19.03.1
 API version:       1.40
 Go version:        go1.12.5
 Git commit:        74b1e89
 Built:             Thu Jul 25 21:18:17 2019
 OS/Arch:           darwin/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.1
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.5
  Git commit:       74b1e89
  Built:            Thu Jul 25 21:17:52 2019
  OS/Arch:          linux/amd64
  Experimental:     true
 containerd:
  Version:          v1.2.6
  GitCommit:        894b81a4b802e4eb2a91d1ce216b8817763c29fb
 runc:
  Version:          1.0.0-rc8
  GitCommit:        425e105d5a03fabd737a126ad93d62a9eeede87f
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```

If that doesn't work, make sure the docker service is running. That will depend
on your laptop. On the Mac, there will be a docker whale in the top status bar.

Verify that is works with the hello-world container

```
➜  ~ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete 
Digest: sha256:6540fc08ee6e6b7b63468dc3317e3303aae178cb8a45ed3123180328bcc1d20f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Verify that Docker Compose is available: `docker-compose version`

```
➜  ~ docker-compose version
docker-compose version 1.24.1, build 4667896b
docker-py version: 3.7.3
CPython version: 3.6.8
OpenSSL version: OpenSSL 1.1.0j  20 Nov 2018
```

## Ready to rumble.

We are going to create and deploy a Gitea container. First we will create the necessary environment for all of our requirements.

Create a new Docker network for the traefik 

```
docker network ls
docker network create ovsagenet
docker network ls
```

You should now notice a new bridge network called **ovsagenet**.

Now create a directory called `deployment`

```
mkdir deployment
cd deployment
```

Create directories for Gitea and PostgreSQL

```
mkdir gitea postgres
```

Create an empty Docker Compose file

```
touch docker-compose.yml
```

Create a configuration file for the traefik reverse proxy

```
touch traefik.toml
```

Things should look like this:

```
➜  docker-gitea tree
.
├── deployment
│   ├── docker-compose.yml
│   ├── gitea
│   ├── postgres
│   └── traefik.toml
└── readme.md
```

The database service PostgreSQL is the first service that we want to configure.
The database service will run only on the internal docker network.

And we will be using the Postgres 9.6 image, using 'gitea' as database name,
user, and password, and set up the postgres data volume.

Edit the `docker-compose.yml` file and put the following in it:

```
version: "3"

networks:
  ovsagenet:
    external: true
  internal:
    external: false

services:
  db:
    image: postgres:9.6
    restart: always
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=gitea
      - POSTGRES_DB=gitea
    labels:
      - "traefik.enable=false"
    networks:
      - internal
    volumes:
      - ./postgres:/var/lib/postgresql/data
```

Save and continue editing. We will add the trafik configuration at the bottom next:

Where you see IPADDR, use the ip address you gathered earlier. If we were going
to utilize HTTPS, we would also add port 443 below 80.

```
  traefik:
    image: traefik:latest
    command: --docker
    ports:
      - 80:80
    labels:
      - "traefik.enable=true"
      - "traefik.backend=dashboard"
      - "traefik.frontend.rule=Host:traefik.IPADDR.xip.io"
      - "traefik.port=8080"
    networks:
      - ovsagenet
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/traefik.toml
    container_name: traefik
    restart: always
```

Save and continue editing.

Add the gitea portion below the trafik section:

```
  server:
    image: gitea/gitea:latest
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    networks:
      - internal
    volumes:
      - ./gitea:/data
    ports:
      - "3000"
      - "22"
    labels:
      - "traefik.enabled=true"
      - "traefik.backend=gitea"
      - "traefik.frontend.rule=Host:git.IPADDR.xip.io"
      - "traefik.docker.network=ovsagenet"
      - "traefik.port=3000"
    networks:
      - internal
      - ovsagenet
    depends_on:
      - db
      - traefik
```

Now we configure the `traefik.toml` file

Edit `traefik.toml` and add the following:

Note the comments where we are disbaling things, If we wanted to utilize
SSL/TLS, we would enable all of this and use LetsEncrypt to provide
certificates.

```
#Traefik Global Configuration
debug = false
checkNewVersion = true
logLevel = "ERROR"

#Define the EntryPoint for HTTP and HTTPS
#defaultEntryPoints = ["https","http"]
defaultEntryPoints = ["http"]

#Define the HTTP port 80 and
#HTTPS port 443 EntryPoint
#Enable automatically redirect HTTP to HTTPS (which we will not do tonight)
[entryPoints]
[entryPoints.http]
address = ":80"
#[entryPoints.http.redirect]
#entryPoint = "https"
#[entryPoints.https]
#address = ":443"
#[entryPoints.https.tls]

#Enable Traefik Dashboard on port 8080
#with basic authentication method
#ovsage and password
[entryPoints.dash]
address=":8080"
[entryPoints.dash.auth]
[entryPoints.dash.auth.basic]
    users = [
        "ovsage:$apr1$hEgpZUN2$OYG3KwpzI3T1FqIg9LIbi.",
    ]

[api]
entrypoint="dash"
dashboard = true

#Enable retry sending a request if the network error
[retry]

#Define Docker Backend Configuration
[docker]
endpoint = "unix:///var/run/docker.sock"
domain = "xip.io"
watch = true
exposedbydefault = false

#Letsencrypt Registration
#Define the Letsencrypt ACME HTTP challenge (we are skipping this as well)
#[acme]
#email = "ovsage@gmail.com"
#storage = "acme.json"
#entryPoint = "https"
#OnHostRule = true
#  [acme.httpChallenge]
#  entryPoint = "http"
```

Save and exit.

If you made it to here, then when we build this container and run it, the Gitea service will be running on the TCP port '3000', using those two docker networks 'internal' and 'ovsagenet', and will run under the traefik reverse proxy on domain 'git.IPADDR.xip.io'.

Your Docker Compose configuration for Gitea deployment has been completed.

## The fun part

Also known as hurry up and wait.

Issue the command `docker-compose up -d' and wait a bit.

When it finished, you will be able to run the command `docker-compose ps` and see interesting things.

```
➜  deployment docker-compose ps
       Name                      Command               State                       Ports                    
------------------------------------------------------------------------------------------------------------
deployment_db_1       docker-entrypoint.sh postgres    Up      5432/tcp                                     
deployment_server_1   /usr/bin/entrypoint /bin/s ...   Up      0.0.0.0:32783->22/tcp,                       
                                                               0.0.0.0:32782->3000/tcp                      
traefik               /traefik --docker                Up      0.0.0.0:80->80/tcp                  
```

You can see that your services are up and running.

### Gitea configuration

If you browse to http://git.IPADDR.xip.io, you will see the gitea splash page.
Now you can start configuring. Go to http://git.IPADDR.xip.io/install to
perform your initial configuration.

#### First the database

Select PostgreSQL, `db` as the host and `gitea` for the next three items. Leave
SSL disabled. Why gitea, look at the docker-compose file for the configuration
you gave it.

#### Then the general config

Change the Site Title to your name. Makes it easy to see which git server we are connecting to. 

Make sure that Server domain is set to git.IPADDR.xip.io

Make sure the Gitea Base URL is set to http://git.IPADDR.xip.io

#### Finally the admin settings

Set the admin username, password and an email address.

Click the INSTALL button.

You should now be redirected to your repository dashboard.


## Things to do later

- Docker upgrade. My version of Docker flagged an update when I loaded it while preparing for this. You can safely ignore it for now. You may want to update it on a higher speed network later.
- Read up on Gitea. It is an elegant solution for much of the things you would want a git ecosystem for.
- 
