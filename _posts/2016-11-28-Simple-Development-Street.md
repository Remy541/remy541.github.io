---
layout: post
title: Simple Development Street
---
For those people who would like to fiddle around with a few services that could make up a development street, look no further! This post is meant as a starting point to setup a complete development street with only a small amount of work. Afterwards you can tailor the installations so they will better fit your needs.  

This development street consists of the following services:  
  - Buildserver: Jenkins 2.x (At the time of writing 2.19.4)  
  - Version Control System: Gitlab 8.x (At the time of writing 8.14.2-ce.0)  
  - Software Quality profiler: Sonar 6.x (At the time of writing 6.1)  

At the end of this manual you'll have a set of services configured to be usable as a development street. As an added 'bonus' I've added Portainer (From portainer.io) into the mix. When you're new to Docker, the interface portainer offers around the Docker daemon will help you with a few basic Docker activities.  

# Installation  
There really is no point of 'installing' the development street. It's just a matter of starting the right Docker containers. But the hardest part has already been done, which is the combination of getting the right Docker images. This can be found in the Docker-Compose file below.  

To start the development street for the first time, just run the following command:  

    docker-compose -f devstreet.yml up -d  

This will start all Docker images in the devstreet.yml file. After waiting some time the images have been downloaded and started and you can start using the services. Well, almost then.  

# Configuration  

## Jenkins  
When you first start the services, you can visit them at the following addresses:  
- Jenkins: [http://localhost:8080](http://localhost:8080)  
- Gitlab: [http://localhost](http://localhost)  
- Sonar: [http://localhost:9000](http://localhost:9000)  
- Portainer: [http://localhost:9999](http://localhost:9999)  

But before you do, let us just finish the one-time 'installations'.  

Open up a web browser and point it to the Jenkins instance:   [http://localhost:9000](http://localhost:9000)  

You'll notice it asks you for an initial password, which can get from the container by issueing the following command:  

    docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword  

A 32-characters long string will appear. This is the temporary administrator password. Fill this password in the input box in the Jenkins interface. After you have done this, you can either select to let Jenkins continue using some default plugins, or you can select which specific plugins you would like to use. I let the reader choose which (extra) plugins he / she needs. For this manual, I'll just continue with the default plugins.  

When everything is installed, Jenkins lets you create a new administrator user.  When you're done doing this, let's continue to Gitlab.  

## Gitlab  

Open up [http://localhost](http://localhost)  

Gitlab starts with asking the user to enter a new administrator's password. After this, it logs you back out, and you need to use the newly entered password with username 'root'.  

After this, you'll arrive in the main Gitlab interface.  


## SonarQube  

Go to the webinterface of SonarQube at [http://localhost:9000](http://localhost:9000)  

The default username / password combination of the SonarQube instance is: admin / admin.  
If you want you can add new software quality profiles in the configuration area.  

## Portainer  

The webinterface of Portainers can be found at [http://localhost:9999](http://localhost:9999) and with it you can investigate which Docker images are available on the system, which Docker volumes and Docker networks, etc.  


# Development Street Overview  

![Development Street Overview image](/public/Overview_pipeline_services.png "Development Street Overview")  

The idea of this setup is as follows. There is one public repository on Gitlab with all kinds of pipeline tools defined as Groovy functions. This repo can be seen in the image above at the top left and is called 'cd'. I'll cover a sample of the contents of this file in the next blogpost.  

Every project will load these tools in its own pipeline definition file, which has to be called 'Jenkinsfile'. Now in the (Multibranch pipeline) Jenkins job configuration, just point to the Gitlab repository and Jenkins will recognize the Jenkinsfile automatically and will be generating a specific tailored pipeline for this specific build configuration.  


# Stuff to consider  

Of course there are things to consider. When Docker is not used properly for instance, you might lose all data that is generated during the uptime of the services.  

## Stopping / (Re)starting  

When you stop the development street, do not use  

    docker-compose -f devstreet.yml down -v

because otherwise the volumes will be removed as well. We want to be able to start and stop the development street at anytime and without it having removed any data.  

## Persistent storage  
In the light of the point above, it would be a good idea to fix persistent storage on a different file location.  


# devstreet.yml  
So, without further ado, the contents of the docker-compose file, I called it devstreet.yml:  

```
version: '2'

services:
  portainer:
    container_name: portainer
    image: portainer/portainer
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "9999:9000"
  gitlab:
    container_name: gitlab
    image: gitlab/gitlab-ce:8.14.2-ce.0
    ports:
      - "22:22"
      - "80:80"
      - "443:443"
    volumes:
      - gitlab_etc:/etc/gitlab
      - gitlab_log:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab

  # Default administrator password can be obtained via: docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
  jenkins:
    container_name: jenkins
    image: jenkins:2.19.4-alpine
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home


  # Default username / password: admin / admin
  sonar:
    container_name: sonar
    image: sonarqube:6.1-alpine
    environment:
      - SONARQUBE_JDBC_USERNAME=sonar
      - SONARQUBE_JDBC_PASSWORD=sonar
      - SONARQUBE_JDBC_URL=jdbc:postgresql://sonar_db:5432/sonar
    links:
      - sonar_db:sonar_db
    ports:
      - "9000:9000"
      - "9092:9092"
    volumes_from:
      - sonar_plugins
    restart: always

  sonar_plugins:
    container_name: sonar_plugins
    image: sonarqube:5.6-alpine
    volumes:
      - sonar_plugins_extensions:/opt/sonarqube/extensions
      - sonar_plugins_bundled:/opt/sonarqube/bundled-plugins
    command: /bin/true


  sonar_db:
    container_name: sonar_db
    image: postgres:9.4
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
    ports:
      - "5432:5432"


volumes:
    gitlab_etc: {}
    gitlab_log: {}
    gitlab_data: {}
    jenkins_home: {}
    sonar_plugins_extensions: {}
    sonar_plugins_bundled: {}
```
