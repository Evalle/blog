# Hygiene of Docker Images

As you probably know, 17 malicious Docker container images have been deleted from Docker Hub. The images, downloaded over 5 million times, helped crooks mine Monero worth over $90,000 at today's exchange rate. I wanted to share my thoughts and examples of how you can secure yourself and your infrastructure from the mining of cryptocurrency for some folks over there. 

## First, The Obvious One

Never-ever-ever download and run the unknown Docker Image from the internet. It's the same as you run the unknown binary from root user on your server.

## Check out the author of the image

It's much, much better to use the Docker Image from known company, such as Oracle https://hub.docker.com/_/mysql/ than from some unknown guy with the nickname docker_user123.

## Double-check the name of the author

It should be "google" not "goog1e".

## Use the Docker Images that were built in automated-way if it's possible

It's preferred because you can check how that image was built. 
Let's take a look at [this image](https://hub.docker.com/r/mysql/mysql-server/) for example. 
You can check the [Dockerfile] (https://hub.docker.com/r/mysql/mysql-server/~/dockerfile/):
See all of its [build history](https://hub.docker.com/r/mysql/mysql-server/builds/):
And even look at its internals in the [GitHub repository](https://github.com/mysql/mysql-docker). 

## Analyze the scripts inside

It's related to the previous point. If you have an access to the git repository of the Image you can (and you should!) analyze the scripts inside of it. You're going to run scripts from that Image on your machine(s) so it's better to know what they're going to do. 

## Build the Image in your isolated environment

After all above steps, you can try to build the Image in your isolated environment and check if it does what was promised. 

## Check the popularity of the Image. 

Even so, it's not the main reason to download the Image, the popularity of the image most means that it was downloaded many times before and somebody could check it. 

## You cannot check everything

That's why I prefer to build the core Docker Images for my projects from the base Docker Images (busybox, alpine, etc) and add all the needed functional on the way. Then I push that image into my private Docker Registry where I can scan it via [clair](https://github.com/coreos/clair) or via [micro-scanner](https://blog.aquasec.com/microscanner-free-image-vulnerability-scanner-for-developers).

That's all of my advice, Please share your thoughts about them so I can add some additional points to this "checklist". 
