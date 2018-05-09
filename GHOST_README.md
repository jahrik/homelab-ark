# Deploying and hosting a public Ark server on the homelab

The purpose of this project is to help educate others that want to break into the world of DevOps or just wanting to bring more automation into their homelab.  Plus, serve as a fun way for me to automate all the things and better myself at documentation and code control.  I will be taking a break from playing Ark long enough to go over my continuous deployment plan and execution.  Ark server configs will be saved to github with included Jenkins pipeline and ansible playbooks needed to test and deploy the Ark server.  When I push a commit to github, Jenkins will see this and pull in the code. When all tests have passed, Jenkins will trigger an ansible template through an AWX (opensource tower) API call, that will pull in any environment variables and build all directories, config files, and finally deploy the whole thing to docker swarm with a docker-stack.yml file.

![Jenkins to AWX](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_to_awx.png)

The [host I'm running this on](https://homelab.business/the-2u-mini-itx-zfs-nas-docker-build-part-2-of-2/) is a very basic setup of ubuntu 18.04 running docker in swarm mode.  Jenkins and Ansible AWX are both running in docker and will also soon be deployed in the same manner as I am preparing to deploy the Ark server, with themselves!  That should be fun.  Data persistence is accomplished by mounting docker volumes at stack deployment time.  A lot of this server build has been manual, but is slowly being put into ansible playbooks, like this one, as I have time.

So, from the beginning.  A system to run this on.  I won't go in to too much detail on [installing ubuntu](https://www.ubuntu.com/server), or [setting up docker swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/), as they have already been documented extensively, beyond my abilities.  But once a system is ready and running docker swarm, Jenkins and Ansible AWX are ready to be deployed.  They are both handled with a docker-stack.yml file and deployed to docker swarm.

**jenkins-stack.yml**

    version: '3'

    services:

      # https://github.com/jenkinsci/docker/blob/master/README.md
      jenkins:
        image: jenkins/jenkins:lts
        ports:
          - '8080:8080'
          - '50000:50000'
        # environment:
          # JAVA_OPTS: "-Djava.awt.headless=true"
        volumes:
          - /shredder_pool/jenkins:/var/jenkins_home

This stack is deployed manually, with the following command.

    docker stack deploy -c jenkins-stack.yml jenkins


