---
layout: post
title:  "Jenkins, Docker and AWS"
date:   2016-01-03 15:53:36 -0500
categories: jenkins docker aws
---

NOTE ABOUT FIRST ARTICLE
NOTE ABOUT DEBUGGING
One of the best things about Docker is effecient utilization of resources.  In my previous Jenkins setup, I had multiple AWS instances for each type of  agent that required specific configuration, such as Hashicorp agents, specific rails app agents, and even Chef agents, and often times these machines were under-utilized.  These agents were provisioned manually via `knife ec2 create` and configured with Chef.  So I set out to figure out if it was possible to run my entire Jenkins infrastructure and jobs inside Docker.  I had 4 goals in mind:

* Shrink machine count by atleast 50% (from roughly 10 agents to less than 5) but handle the same load
* Run it on AWS
* Do as little as possible to get it working
* Utilize some of the newest features of Docker, including docker swarm and docker networking.

### Get it working locally

*** Talk about how to share the socket for launching on the host, and overall architecture that we are trying to achieve by using this 

Step one of getting any Docker project working is to do it locally.  For Jenkins I created two separate Dockerfile's, one for the agent and one for the master.  

#### Jenkins Master

You can find the Jenkins master dockerfile [here](URL NEEDED), but here is a breakdown for each step.

{% highlight bash %}
FROM java:8-jre

ENV JENKINS_MASTER_PORT 8080
ENV JENKINS_ROOT /var/lib/jenkins
ENV JENKINS_VERSION 1.625.3
ENV JENKINS_JOBS_DIR /var/lib/jenkins/jobs
ENV JENKINS_PLUGIN_DIR /var/lib/jenkins/plugins/

RUN apt-get update -qq && apt-get -y install jq build-essential
RUN mkdir -p $JENKINS_ROOT
RUN useradd -ms /bin/bash jenkins && echo "jenkins:password" | chpasswd && adduser jenkins sudo && chown -R jenkins:jenkins $JENKINS_ROOT
RUN su - jenkins -c "wget -O $JENKINS_ROOT/jenkins.war http://mirrors.jenkins-ci.org/war-stable/$JENKINS_VERSION/jenkins.war"

{% endhighlight %}

  Setup ENV variables, install dependencies, create a jenkins user and download the jenkins war file.  NOTE: You can use a non-stable version of jenkins by removing the `stable` from `/war-stable/` in the Jenkins URL.

{% highlight bash %}
# Install plugins
RUN mkdir -p $JENKINS_PLUGIN_DIR
RUN wget -L -O $JENKINS_PLUGIN_DIR/swarm.hpi https://updates.jenkins-ci.org/latest/swarm.hpi
RUN wget -L -O $JENKINS_PLUGIN_DIR/ghprb.hpi https://updates.jenkins-ci.org/latest/ghprb.hpi
RUN wget -L -O $JENKINS_PLUGIN_DIR/github.hpi https://updates.jenkins-ci.org/latest/github.hpi
RUN wget -L -O $JENKINS_PLUGIN_DIR/github-api.hpi https://updates.jenkins-ci.org/latest/github-api.hpi
RUN wget -L -O $JENKINS_PLUGIN_DIR/github-oauth.hpi https://updates.jenkins-ci.org/latest/github-oauth.hpi
{% endhighlight %}

Install Jenkins plugins, including all required plugins to do Pull Request builds (discussed later on)

{% highlight bash %}

# Install template job
RUN mkdir -p $JENKINS_JOBS_DIR
ADD jenkins-config/pr-job-template.xml $JENKINS_JOBS_DIR/pr-job-template.xml
{% endhighlight %}

Create a job template that we can use to make a job that builds project PR's.

{% highlight bash %}

# Copy configs and users
ADD jenkins-config/config.xml $JENKINS_ROOT/config.xml
ADD jenkins-config/users $JENKINS_ROOT/users

{% endhighlight %}

Create the Jenkins config and create our users that we want by default.  This is important as the configuration expects users for the security matrix.  If you want to change this please edit the config

{% highlight bash %}
# Install docker
ENV DOCKER_BUCKET get.docker.com
ENV DOCKER_VERSION 1.7.1
ENV DOCKER_SHA256 4d535a62882f2123fb9545a5d140a6a2ccc7bfc7a3c0ec5361d33e498e4876d5
ENV DOCKER_COMPOSE_VERSION 1.4.2

RUN curl -fSL "https://${DOCKER_BUCKET}/builds/Linux/x86_64/docker-$DOCKER_VERSION" -o /usr/local/bin/docker \
  && echo "${DOCKER_SHA256}  /usr/local/bin/docker" | sha256sum -c - \
  && chmod +x /usr/local/bin/docker

# Install docker compose
RUN curl -L https://github.com/docker/compose/releases/download/$DOCKER_COMPOSE_VERSION/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose \
  && chmod +x /usr/local/bin/docker-compose

{% endhighlight %}

Install Docker and docker-compose.  These aren't required, but sometimes you may want to run Docker commands on the Jenkins master.

{% highlight bash %}

EXPOSE 8080

CMD JENKINS_HOME=$JENKINS_ROOT java -Xmx256m -Xms128m -jar $JENKINS_ROOT/jenkins.war --httpListenAddress=0.0.0.0 --httpPort=8080 --ajp13Port=-1 --debug=9

{% endhighlight %}

Start Jenkins with 256MB of heap and listen on port 8080.  Our agents will connect to Jenkins on this port, but we will forward port 80 to 8080 as well to allow for easy access to the UI.

#### Jenkins Agent

{% highlight bash %}
FROM java:8

ENV JENKINS_ROOT /var/jenkins
ENV SWARM_CLIENT_VERSION 2.0

RUN apt-get update -qq && apt-get -y install build-essential
RUN mkdir -p $JENKINS_ROOT
RUN useradd -ms /bin/bash jenkins && echo "jenkins:password" | chpasswd && adduser jenkins sudo && chown -R jenkins:jenkins $JENKINS_ROOT
RUN apt-get update && apt-get install -y wget
RUN su - jenkins -c "wget -O $JENKINS_ROOT/swarm-client-$SWARM_CLIENT_VERSION-jar-with-dependencies.jar https://s3.amazonaws.com/articulate-jenkins/swarm-client/$SWARM_CLIENT_VERSION/swarm-client-$SWARM_CLIENT_VERSION-jar-with-dependencies.jar"

# Install docker
ENV DOCKER_BUCKET get.docker.com
ENV DOCKER_VERSION 1.7.1
ENV DOCKER_SHA256 4d535a62882f2123fb9545a5d140a6a2ccc7bfc7a3c0ec5361d33e498e4876d5 
ENV DOCKER_COMPOSE_VERSION 1.4.2

RUN curl -fSL "https://${DOCKER_BUCKET}/builds/Linux/x86_64/docker-$DOCKER_VERSION" -o /usr/local/bin/docker \
  && echo "${DOCKER_SHA256}  /usr/local/bin/docker" | sha256sum -c - \
  && chmod +x /usr/local/bin/docker

# Install docker compose
RUN curl -L https://github.com/docker/compose/releases/download/$DOCKER_COMPOSE_VERSION/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose \
  && chmod +x /usr/local/bin/docker-compose

# Clean up
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

## Start jenkins agent
CMD java -DHOME=$JENKINS_ROOT -Xmx256m -Xms128m \
  -jar $JENKINS_ROOT/swarm-client-$SWARM_CLIENT_VERSION-jar-with-dependencies.jar \
  -executors 1 -fsroot $JENKINS_ROOT -labels swarm -mode exclusive \
  -name $JENKINS_AGENT_NAME \
  -username $JENKINS_USERNAME \
  -password $JENKINS_PASSWORD \
  -master http://$JENKINS_MASTER:$JENKINS_MASTER_PORT
{% endhighlight %}

Creation of the Jenkins agent is very similar except for the war file we use.  In this case we are using the swarm client, which allows for our agents to be automagically added to the Jenkins master without any human intervention.  In addition we use basic Jenkins security to allow our agents to authenticate with the master securely (without this security step there is nothing stopping agents from being connected to the cluster).

#### Start the containers locally, on a single Virtualbox machine.

First build our containers
{% highlight bash %}
docker build -t jenkins-master -f Dockerfile-master .
docker build -t jenkins-agent -f Dockerfile-agent .
{% endhighlight %}

Now start the master container
{% highlight bash %}
docker run -itd -p 80:8080 --name=jenkins-master jenkins-master
{% endhighlight %}

Now start the agent, and create a link to the master

{% highlight bash %}
docker run -itd -v /var/run/docker.sock:/var/run/docker.sock --name=jenkins-agent-1 --link jenkins-master:jenkins-master -e "JENKINS_MASTER=jenkins-master" -e "JENKINS_MASTER_PORT=8080" jenkins-agent
{% endhighlight %}

Now you can visit the Jenkins UI by running `open http://$(docker-machine ip default)`.  You should be greeted with a login prompt, which is (LOGIN CREDS HERE).  Once logged in you should see 1 agent connected and 1 template job available.

### Swarm and networking

NOTES ABOUT SWITCHINGBETWEEN MACHINES VIA DM

The next step is to get Jenkins running with the agents on separate VMs.  The goal here is to have one agent per machine that you want to run Docker commands.  Each agent will then launch Docker containers on the host it is running on.

My first and best resource was [this Docker article](https://docs.docker.com/engine/userguide/networking/get-started-overlay/) that specifically talks about setting up a local cluster of VM's using Virtualbox and docker-machine.  Feel free to walk through that article and skip this section and move onto the [AWS section](#onto-aws).  For the full file containing all the commands please see [here](FILE HERE).

{% highlight bash %}
# Consul master for swarm discovery
docker-machine create \
    -d virtualbox \
    swarm-consul-master

docker $(docker-machine config swarm-consul-master) run -d \
    -p "8500:8500" \
    -h "consul" \
    --name=swarm-consul-master \
    progrium/consul -server -bootstrap

{% endhighlight %}

Consul is used as a key value store for Docker swarm and swarm networking.  It runs on it's own machine and runs as a single container.  This machine can not be a part of the swarm cluster as it is a prequesite for starting the cluster and cluster discovery (I mention this in case you wanted to reuse this same Consul service for something other than Docker swarm config).

{% highlight bash %}
# Create swarm master
docker-machine create \
    -d virtualbox \
    --virtualbox-disk-size 50000 \
    --swarm \
    --swarm-master \
    --swarm-discovery="consul://$(docker-machine ip swarm-consul-master):8500" \
    --engine-opt="cluster-store=consul://$(docker-machine ip swarm-consul-master):8500" \
    --engine-opt="cluster-advertise=eth1:0" \
    swarm-master

{% endhighlight %}

Here we are creating our first swarm machine, in this case the swarm master (designated because of the `--swarm-master` flag).  This machine will be the one you connect to with the --swarm flag to view cluster status.

{% highlight bash %}
eval $(docker-machine env --swarm swarm-master)
docker network create --driver overlay jenkins-net
{% endhighlight %}

Connect to the swarm master and create the overlay network that will be used for communication between containers.  It is important that you create the network on the swarm master after connecting with the `--swarm` flag.  
{% highlight bash %}
eval $(docker-machine env swarm-master)
docker build -t jenkins-master -f Dockerfile-master .
docker run -itd -p 80:8080 -p 8080:8080 --name=jenkins-master --net=jenkins-net jenkins-master
{% endhighlight %}

Build the agent and run it.  Passing `--net=jenkins-net` tells Docker to add our overlay network to the container.

{% highlight bash %}
# Create swarm machine 1
docker-machine create \
    -d virtualbox \
    --virtualbox-disk-size 50000 \
    --swarm \
    --swarm-discovery="consul://$(docker-machine ip swarm-consul-master):8500" \
    --engine-opt="cluster-store=consul://$(docker-machine ip swarm-consul-master):8500" \
    --engine-opt="cluster-advertise=eth1:0" \
    swarm-agent-1 
{% endhighlight %}

Create your first agent box and join the swarm via discovery (via consul), aka magic.

{% highlight bash %}
eval $(docker-machine env swarm-agent-1)
docker build -t jenkins-agent -f Dockerfile-agent .
docker run -itd \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name=jenkins-agent-1 \
  --net=jenkins-net \
  -e "JENKINS_MASTER=jenkins-master" \
  -e "JENKINS_MASTER_PORT=8080" \
  jenkins-agent
{% endhighlight %}

Create your first agent container, connect it to the host Docker socket, and join it to the `jenkins-net` overlay network.

### Onto AWS

### Dockerizing jobs




Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
