# Automating the deployment of a public hosted Ark Survival Evolved Server with Jenkins and Ansible AWX (Part 1 of some)

The purpose of this project is to help educate others that want to break into the world of DevOps or just wanting to bring more automation into their homelab.  Plus, serve as a fun way for me to automate all the things and better myself at documentation and code control.  I will be taking a break from playing Ark long enough to go over my continuous deployment plan and execution.  Ark server configs will be saved to github with included Jenkins pipeline and ansible playbooks needed to test and deploy the Ark server.  When a commit is pushed to github, Jenkins will see this and pull in the code. When all tests have passed, Jenkins will trigger an ansible template through an AWX (opensource tower) API call, that will pull in any environment variables and build all directories, config files, and finally deploy the whole thing to docker swarm with a docker-stack.yml file.

![Jenkins to AWX](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_to_awx.png)

The [host I'm running this on](https://homelab.business/the-2u-mini-itx-zfs-nas-docker-build-part-2-of-2/) is a very basic setup of ubuntu 18.04 running docker in swarm mode.  Jenkins and Ansible AWX are both running in docker and will also soon be deployed in the same manner as I am preparing to deploy the Ark server, with themselves!  That should be fun.  Data persistence is accomplished by mounting docker volumes at stack deployment time.  A lot of this server build has been manual, but is slowly being put into ansible playbooks, like this one, as I have time.

So, from the beginning.  A system to run this on.  I won't go in to too much detail on [installing ubuntu](https://www.ubuntu.com/server), or [setting up docker swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/), as they have already been documented extensively, beyond my abilities.  But once a system is ready and running docker swarm, Jenkins and Ansible AWX are ready to be deployed.  They are both handled with a docker-stack.yml file and deployed to docker swarm. `/data/` is the root directory on the system where docker will store volumes.

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
          - /data/jenkins:/var/jenkins_home

This stack is deployed manually, with the following commands.

    sudo mkdir -p /data/jenkins
    sudo chown -R 1000:1000 /data/awx
    docker stack deploy -c jenkins-stack.yml jenkins

Follow the logs as it builds

    docker service logs -f jenkins_jenkins

Once it's up and running you'll see that it's written files to the `/data/jenkins/` directory.

    ls /data/jenkins/

    config.xml
    copy_reference_file.log
    credentials.xml
    envinject-plugin-configuration.xml
    fingerprints
    github-plugin-configuration.xml
    hudson.model.UpdateCenter.xml
    ...
    ...


Remember that config.xml file ^ !  You'll need that when you forget your password and lock yourself out of Jenkins.

Browse to http://your_server_ip:8080/ or [http://localhost:8080/](http://localhost:8080/) if you're on the box running swarm.
![jenkins_login.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_login.png)

* Set up a user, password and whatever else and enable security through `jenkins > Manage Jenkins > Configure Global Security > Enable Security`, [http://localhost:8080/configureSecurity/](http://localhost:8080/configureSecurity/)
* tick `Allow users to sign up`
  * untick this after you have created an account to disable further accounts being created
* tick `Logged-in users can do anything`
* untick `￼Allow anonymous read access`
* tick `Enable Agent → Master Access Control`
* Apply and Save

![jenkins_enable_security.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_enable_security.png)
![jenkins_enable_agent.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_enable_agent.png)

Next, bring up the Ansible AWX stack.  It's a bit more complicated than the Jenkins stack and runs multiple containers. You'll want to change the passwords.

**awx-stack.yml**

    version: '3'
    services:

      web:
        image: ansible/awx_web:latest
        depends_on:
          - rabbitmq
          - memcached
          - postgres
        ports:
          - "80:8052"
        hostname: awxweb
        user: root
        deploy:
          restart_policy:
            condition: on-failure
            delay: 5s
            # max_attempts: 3
            window: 60s
        environment:
          http_proxy: 
          https_proxy: 
          no_proxy: 
          SECRET_KEY: awxsecret
          DATABASE_NAME: awx
          DATABASE_USER: awx
          DATABASE_PASSWORD: awxpass
          DATABASE_PORT: 5432
          DATABASE_HOST: postgres
          RABBITMQ_USER: guest
          RABBITMQ_PASSWORD: guest
          RABBITMQ_HOST: rabbitmq
          RABBITMQ_PORT: 5672
          RABBITMQ_VHOST: awx
          MEMCACHED_HOST: memcached
          MEMCACHED_PORT: 11211
          AWX_ADMIN_USER: admin
          AWX_ADMIN_PASSWORD: password

      task:
        image: ansible/awx_task:latest
        depends_on:
          - rabbitmq
          - memcached
          - web
          - postgres
        hostname: awx
        user: root
        deploy:
          restart_policy:
            condition: on-failure
            delay: 5s
            # max_attempts: 3
            window: 60s
        environment:
          http_proxy: 
          https_proxy: 
          no_proxy: 
          SECRET_KEY: awxsecret
          DATABASE_NAME: awx
          DATABASE_USER: awx
          DATABASE_PASSWORD: awxpass
          DATABASE_HOST: postgres
          DATABASE_PORT: 5432
          RABBITMQ_USER: guest
          RABBITMQ_PASSWORD: guest
          RABBITMQ_HOST: rabbitmq
          RABBITMQ_PORT: 5672
          RABBITMQ_VHOST: awx
          MEMCACHED_HOST: memcached
          MEMCACHED_PORT: 11211
          AWX_ADMIN_USER: admin
          AWX_ADMIN_PASSWORD: password

      rabbitmq:
        image: rabbitmq:3
        deploy:
          restart_policy:
            condition: on-failure
            delay: 5s
            # max_attempts: 3
            window: 60s
        environment:
          RABBITMQ_DEFAULT_VHOST: awx

      memcached:
        image: memcached:alpine
        deploy:
          restart_policy:
            condition: on-failure
            delay: 5s
            # max_attempts: 3
            window: 60s

      postgres:
        image: postgres:9.6
        deploy:
          restart_policy:
            condition: on-failure
            delay: 5s
            # max_attempts: 3
            window: 60s
        volumes:
          - /data/awx:/var/lib/postgresql/data:Z
        environment:
          POSTGRES_USER: awx
          POSTGRES_PASSWORD: awxpass
          POSTGRES_DB: awx
          PGDATA: /var/lib/postgresql/data/pgdata

Bring up the stack with the following

    sudo mkdir -p /data/awx
    sudo chown -R 999:999 /data/awx
    docker stack deploy -c awx-stack.yml awx

First, you'll want to follow logs from the postgres service as it builds.

    docker service logs -f awx_postgres

And then follow `awx_web` as it preps the database and performs any upgrades using `awx_task`

    docker service logs -f awx_web
    docker service logs -f awx_task

It takes a bit for awx to build.  Give it a good 10-20 minutes before you give up on it.
Browse to http://your_server_ip:80/#/login or [http://localhost/#/login](http://localhost/#/login) if you're on the box and you'll be greeted with the AWX login screen!  Huzzah!
![awx_login.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_login.png)














