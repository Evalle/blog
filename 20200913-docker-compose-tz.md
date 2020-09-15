# How to Change Time Zone in Docker Containers

*Time is money*

... or so they say. This blog post will help you to change a timezone in your docker containers easily.
The term *timezone* can be used to describe several different things, but mostly it refers to the local time of a region or a country.

On Linux or MacOS you can check your time zone via

```bash
$ date
Mon Jun 11 19:26:35 CEST 2018
```

On Linux, you can go even further and check timezone via

```bash
$ timedatectl | grep -i "time zone"
       Time zone: Europe/Prague (CEST, +0200)
```

So my time zone is CEST. The problem here is that most of the (if not all) containers are run with UTC timezone:

```bash
$ docker exec  2391ef6bfda8 date
Mon Jun 11 17:32:06 UTC 2018
```

It can be a problem if you collect all logs from your containers with the software such as Splunk. In fact, that was my case. We aggregate all of the logs from one of our containerized services with Splunk and we found out that the logs from the containers have wrong timestamps. 

OK, enough with lyrics let's fix this issue :)

## Simple Docker Container

Let's imagine we want to fix timezone issue within a single container. 
My host OS is CentOS so I will run centos container on top of it. Let's check our time zone on the host system first:

```bash
$ date
Mon Jun 11 19:49:50 CEST 2018
```

and if we run a simple container on top of it and check it's timezone: 

```bash
$ docker run  centos date
Mon Jun 11 17:50:36 UTC 2018
```

UTC! Let's mount `/etc/localtime` from the host system to the container and check the result: 

```bash
$ docker run -v /etc/localtime:/etc/localtime:ro centos date
Mon Jun 11 19:52:19 CEST 2018
```

Now you have it - container has the same timezone as the host OS. Now let's do the same with Docker-Compose files. 

## Docker Compose 

Let's imagine that we have a simple docker compose file, like this one:

```yaml
version: "2"
services:
  web:
    build: web
    command: python app.py
    ports:
     - "5000:5000"
    volumes:
     - ./web:/code
    links:
     - redis
    environment:
     - DATADOG_HOST=datadog
  redis:
    image: redis
  # agent section
  datadog:
    build: datadog
    links:
     - redis
     - web
    environment:
     - API_KEY=__your_datadog_api_key_here__
    volumes:
     - /var/run/docker.sock:/var/run/docker.sock
     - /proc/mounts:/host/proc/mounts:ro
     - /sys/fs/cgroup:/host/sys/fs/cgroup:ro
```

Its containers have  UTC timezones, if we want to change then to our hosts' timezone we should mount the same `/etc/localtime` file to them: 

```bash
version: "2"
services:
  web:
    build: web
    command: python app.py
    ports:
     - "5000:5000"
    volumes:
     - ./web:/code
     - /etc/localtime:/etc/localtime:ro 
    links:
     - redis
    environment:
     - DATADOG_HOST=datadog
  redis:
    image: redis
  # agent section
  datadog:
    build: datadog
    links:
     - redis
     - web
    environment:
     - API_KEY=__your_datadog_api_key_here__
    volumes:
     - /var/run/docker.sock:/var/run/docker.sock
     - /proc/mounts:/host/proc/mounts:ro
     - /sys/fs/cgroup:/host/sys/fs/cgroup:ro
     - /etc/localtime:/etc/localtime:ro 
```

Now both `redis` and `web` containers will have the right timezone and you can easily search through your aggregated logs and you will always have the same time across all of your containerized and non-containerized services. 

In the next blog post, I will show you how to change the time zone in Kubernetes pods. Stay tuned.  

## Useful links:

- [Timezones](https://en.wikipedia.org/wiki/Time_zone)
- [Docker compose](https://docs.docker.com/compose/)
