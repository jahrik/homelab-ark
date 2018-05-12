# Automating the deployment of a publicly hosted Ark Survival Evolved server with Jenkins and Ansible AWX on Docker Swarm

The purpose of this project is to educate others that want to break into the world of DevOps or just wanting to bring more automation into their homelab.  Plus, serve as a fun way for me to automate all the things and better myself at documentation and code control.  I will be taking a break from playing Ark long enough to go over my continuous deployment plan and execution.  Ark server configs will be saved to github with included Jenkins pipeline and Ansible playbooks needed to test and deploy the Ark server.  When a commit is pushed to github, Jenkins will see this and pull in the code. When all tests have passed, Jenkins will trigger an Ansible template through an AWX (opensource tower) API call, that will pull in any environment variables and build all directories, config files, and finally deploy the whole thing to Docker swarm with a docker-stack.yml file.

![Jenkins to AWX](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_to_awx.png)

## Docker

The [host I'm running this on](https://homelab.business/the-2u-mini-itx-zfs-nas-docker-build-part-2-of-2/) is a very basic install of Ubuntu 18.04 running Docker in swarm mode.  Jenkins and Ansible AWX are both running in Docker and will also soon be deployed in the same manner as I am preparing to deploy the Ark server, with a Jenkinsfile and Ansible playbooks.  That should make for some fun problems.  Data persistence is accomplished by mounting Docker volumes at stack deployment time.  A lot of this server build has been manual, but is slowly being put into Ansible playbooks, [like this one](), as I have time.

So, from the beginning, a system to run this on.  I won't go in to too much detail on [installing Ubuntu](https://www.ubuntu.com/server), or [setting up Docker swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/), as they have already been documented extensively, beyond my abilities.  But once a system is ready and running Docker swarm, Jenkins and Ansible AWX are ready to be deployed.  They are both handled with a docker-stack.yml file and deployed to Docker swarm. `/data/` is the root directory on the system where Docker will store volumes.

### Jenkins

Jenkins is being deployed with a pretty standard stack file.  I haven't complicated it too much yet, but I do have plans to add a slave or two also running in Docker, like the master.  As part of this project, I will be hooking up an old laptop to the Jenkins master to act as a slave and handle vagrant and virtualbox for spinning up vms as well as Docker for container driven tests and Docker deployments to swarm. It will also be a manager in the Swarm cluster and have direct access to all `docker stack deploy` commands.  That Slave is not included in this config file, but will be mentioned again later.

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

This stack is deployed manually, with the following commands:

    sudo mkdir -p /data/jenkins
    sudo chown -R 1000:1000 /data/jenkins
    docker stack deploy -c jenkins-stack.yml jenkins

Follow the logs as it builds...

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


**Remember that config.xml file ^ !  You'll need that when you forget your password and lock yourself out of Jenkins.**

Browse to http://your_server_ip:8080/ or [http://localhost:8080/](http://localhost:8080/) if you're on the box running swarm.
![jenkins_login.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_login.png)

* Set up a user, password and whatever else and enable security through `Jenkins > Manage Jenkins > Configure Global Security > Enable Security`, [http://localhost:8080/configureSecurity/](http://localhost:8080/configureSecurity/)
* tick `Allow users to sign up`
  * untick this after you have created an account to disable further accounts being created
* tick `Logged-in users can do anything`
* untick `￼Allow anonymous read access`
* tick `Enable Agent → Master Access Control`
* Apply and Save

![jenkins_enable_security.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_enable_security.png)
![jenkins_enable_agent.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_enable_agent.png)

### AWX

Next, bring up the Ansible AWX stack.  It's a bit more complicated than the Jenkins stack and runs multiple containers. **You'll want to change the passwords. These are only examples.**  Eventually, I will get this streamlined with environment variables in the stack file, that will be populated at build time in either Jenkins as it hands it off to AWX, or pulled in to AWX from vault or something similar. 

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

Bring up the stack with the following:

    sudo mkdir -p /data/awx
    sudo chown -R 999:999 /data/awx
    docker stack deploy -c awx-stack.yml awx

First, you'll want to follow logs from the postgres service as it builds...

    docker service logs -f awx_postgres

And then follow `awx_web` as it preps the database and performs any upgrades using `awx_task`

    docker service logs -f awx_web
    docker service logs -f awx_task

It takes a bit for AWX to build.  Give it a good 10-20 minutes before you give up on it.
Browse to http://your_server_ip:80/#/login or [http://localhost/#/login](http://localhost/#/login) if you're on the box and you'll be greeted with the AWX login screen!  Huzzah!

Defaults are `admin` `password`

![awx_login.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_login.png)

### API

Create a user for Jenkins to use through the API

![awx_jenkins_user.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_jenkins_user.png)

Add that user and password back in Jenkins at `Jenkins > Credentials > System > Global credentials`

![jenkins_awx_user.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_awx_user.png)

Browse to `Jenkins > Manage Jenkins > Ansible Tower` and add the newly built Ansible AWX to Jenkins.  Give it a name, the url to to AWX, and use the credentials made in the last step.  Hit "test" to verify a connection.

![jenkins_awx_add_tower.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_awx_add_tower.png)

## Ansible

Back to AWX, there is more to configure.  First of which, are credentials of a user on the Docker Host with passwordless ssh and sudo access.

![ansible_machine_user.png](https://github.com/jahrik/homelab-ark/raw/master/images/ansible_machine_user.png)

Create an inventory containing the Docker Swarm host.  I'm calling mine by hostname, `shredder`.  An IP works just as well.  I'm populating this inventory by pulling it in as a [project from GitHub](https://github.com/jahrik/home_lab) and configuring AWX to use the [inventory.ini](https://github.com/jahrik/home_lab/blob/master/inventory.ini) file.

![awx_inventory.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_inventory.png)

Create a project to pull in the code from Github.  This project will pull in [this very repository](https://github.com/jahrik/homelab-ark).

![awx_project.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_project.png)

Once a project is created and we're pulling in code, a template can be constructed that Jenkins will call through the API.  This template uses the inventory created above, the credentials created above, the project being pulled in, docker-ark, and uses a [playbook](https://github.com/jahrik/homelab-ark/blob/master/playbook.yml) in that project to call the [main task](https://github.com/jahrik/homelab-ark/blob/master/ark/tasks/main.yml).

![awx_ark_template.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_ark_template.png)

This playbook is still very basic, and only does some minor directory prep and config file generation, but it's a great start to test results.  Here is what a run looks like through AWX.

![ansible_ark_main_task.png](https://github.com/jahrik/homelab-ark/raw/master/images/ansible_ark_main_task.png)

## Jenkinsfile

In order to get Jenkins to fire off this templated playbook, a few plugins need to be installed first.
* [Ansible Tower Plugin](http://wiki.jenkins-ci.org/display/JENKINS/Ansible+Tower+Plugin)
* [AnsiColor Plugin](https://wiki.jenkins.io/display/JENKINS/AnsiColor+Plugin)

Browse to `Jenkins > Manage Jenkins > Manage Plugins` and install any plugins needed.

![jenkins_ansible_tower_plugin.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_ansible_tower_plugin.png)

With the plugins installed, they can be called in the Jenkinsfile.  I'm starting with a very basic Pipeline, that will grow in to more as I build on tests, call Docker builds, test Ansible playbooks with molecule, etc.  During the `deploy` stage, the `ansibleTower` plugin is called and populated with the values that were configured above, when connecting Jenkins to Ansible AWX.  The following variables set, should be all that is needed to get started.  The tower server itself and the template to point at.

    ansibleTower(
      towerServer: 'Ansible AWX'
      jobTemplate: 'ark'
      ...
      ...
    )

**[Jenkinsfile](https://github.com/jahrik/homelab-ark/blob/master/Jenkinsfile)**

    #!/usr/bin/env groovy

    node('master') {

        try {

            stage('build') {
                // Clean workspace
                deleteDir()
                // Checkout the app at the given commit sha from the webhook
                checkout scm
            }

            stage('test') {
                // Run any testing suites
                sh "echo 'WE ARE TESTING'"
            }

            stage('deploy') {
                sh "echo 'WE ARE DEPLOYING'"
                wrap([$class: 'AnsiColorBuildWrapper', colorMapName: "xterm"]) {
                    ansibleTower(
                        towerServer: 'Ansible AWX',
                        jobTemplate: 'ark',
                        importTowerLogs: true,
                        inventory: '',
                        jobTags: '',
                        limit: '',
                        removeColor: false,
                        verbose: true,
                        credential: '',
                        extraVars: ''
                    )
                }
            }

        } catch(error) {
            throw error

        } finally {
            // Any cleanup operations needed, whether we hit an error or not

        }
    }

Be sure to add the Jenkins user to the permissions tab on the template itself, or an error will come back, something like 'template does not exist' because the user accessing the AWX API does not have permissions to see that template unless they're an admin.  Give it at least `Execute` capabilities.

![awx_template_permissions.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_template_permissions.png)

## Webhook

Configure a webhook to trigger a build in Jenkins for every `push` to Github.  This is accomplished by browsing to the [docker-ark](https://github.com/jahrik/homelab-ark) `repo > Settings > Integrations and Services > Add Service > Jenkins (GitHub plugin)`.  The url will be the public IP, (ex. http://123.123.123.123:8080/github-webhook/) of your Jenkins Master.

![jenkins_github_plugin.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_github_plugin.png)

Created a NAT rule to forward traffic hitting port 8080 on the Public WAN (123.123.123.123) and redirect it to the Docker host private ip (192.168.123.123), so it hits the Jenkins Master at port 8080.

![jenkins_pfsense_nat.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_pfsense_nat.png)

If this is something you'd like to try and keep internal and private to your homelab, you can easily bring up a Bitbucket server running in Docker swarm, that will give you all the same functionality of Github and remain free for up to 5 users.  Here is a stack file to bring up a Bitbucket server in Docker swarm.  Deploy it and log in with your Atlassian account or create a new one.

**bitbucket-stack.yml**

    version: '3'

    services:

      Bitbucket:
        image: atlassian/bitbucket-server
        ports:
          - '7990:7990'
          - '7999:7999'
        volumes:
          - /data/bitbucket:/var/atlassian/application-data/bitbucket

Deploy the stack with

    sudo mkdir -p /data/bitbucket
    sudo chown -R daemon:daemon /data/bitbucket
    docker stack deploy -c bitbucket-stack.yml bb

Verify it's running at [http://localhost:7990/login](http://localhost:7990/login)

![bb_server.png](https://github.com/jahrik/homelab-ark/raw/master/images/bb_server.png)

## Build

And with that ladies and gentlemen, a push to Github should trigger a build in Jenkins which will then hit the API to AWX and deploy a playbook to the Docker Swarm host!  As I'm writing and pushing files up in this project I've been watching the builds go by and it's a lot of fun!  The possibilities seam endless with a pipeline like this.  It will make for a great template for deploying more things to my Docker Swarm Cluster and building out the homelab.

Jenkins Pipeline
![jenkins_ark_builds.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_ark_builds.png)

Ansible Playbook
![awx_ark_jobs.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_ark_jobs.png)


## Ark

Finally, to the point of deploying and maintaining the Ark server on Docker swarm!  The main reason I like this game, is that is runs natively on Linux when installed through Steam.  It's been a great time-waster lately and has me excited about gaming again.  I'm still playing solo on the server I'm hosting, but it is publicly available for others to join.  Long term plans are to get more people playing with me and stress testing the server it runs on.  I'm feeling more comfortable in the back end of the Ark server as far as config files and what not goes.  I've got it backing up every 15 minutes and have the restore from backup down to a science now after multiple catastrophic disasters and losing all my dinos from Alpha Raptor attacks!  Fuckers! Yes, it might be considered cheating... But I'm the only one keeping me accountable right now.  I'd have to take that into consideration if more people start playing on the server and can no longer roll it back as I wish.  Another thing I've been trying to get working is mods, but no luck with that yet.  I have been able to tweak harvest multipliers and taming multipliers though and make the game feel a bit less grindy.  I want to spin up a couple more of the expansions as Docker services running along side the Island server currently being hosted.

To deploy Ark to Docker swarm, I started with the [TuRz4m Ark Docker](https://github.com/TuRz4m/Ark-docker) repo, and upgraded their [docker-compose.yml](https://github.com/TuRz4m/Ark-docker/blob/master/docker-compose.yml) file to Docker Compose v3 and renamed it [docker-stack.yml](https://github.com/jahrik/homelab-ark/blob/master/docker-stack.yml) file.

You can see there are some subtle differences, like the formatting of the environment variables.

    sdiff docker-compose.yml docker-stack.yml

    ark:                            | version: '3'
      container_name: ark           |
      image: turzam/ark             | services:
      environment:                  |   island:
        - SESSIONNAME=Ark Docker    |     image: turzam/ark
        - SERVERMAP=TheIsland       |     environment:
        - SERVERPASSWORD=""         |       SESSIONNAME: Ark Docker
        - ADMINPASSWORD=""          |       SERVERMAP: TheIsland
        - BACKUPONSTART=1           |       SERVERPASSWORD: ""
        - UPDATEONSTART=1           |       ADMINPASSWORD: ""
        - TZ=Europe/Paris           |       BACKUPONSTART: 1
        - GID=1000                  |       UPDATEONSTART: 1
        - UID=1000                  |       AUTOBACKUP: 15
      volumes:                      |       TZ: US/Pacific
        - /data/ark:/ark            |       GID: 1000
      ports:                        |       UID: 1000
       - 7778:7778/udp              |     volumes:
       - 7778:7778                  |       - /data/ark:/ark
       - 27015:27015/udp            |     ports:
       - 27015:27015                |       - '7778:7778/udp'
       - 32330:32330                |       - '27015:27015/udp'
                                    >       - '32330:32330'

For a manual deployment, this one works just as easily as AWX and Jenkins did.  It can be deployed with the following.

    sudo mkdir -p /data/ark
    sudo chown -R 1000:1000 /data/ark
    docker stack deploy -c bitbucket-stack.yml ark

But, I want Jenkins and Ansible to handle this deployment for me.  While putting this all together, [here is the commit](https://github.com/jahrik/homelab-ark/commit/a301afe8d29db6cbb3f786b00c3c6bae9d371b45) that finally made Ansible deploy this to the swarm!  Now I'm getting somewhere!

Here is the Jenkins run from that commit.
![jenkins_job_71.png](https://github.com/jahrik/homelab-ark/raw/master/images/jenkins_job_71.png)

And the Ansible job that was kicked off.
![awx_job_322.png](https://github.com/jahrik/homelab-ark/raw/master/images/awx_job_322.png)

Log into the server and check it out.

`docker ps` to list running docker processes 

    docker ps

    CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS                PORTS                                                                            NAMES
    015cc0306a64        turzam/ark:latest                                     "/home/steam/user.sh"    43 minutes ago      Up 43 minutes         7778/tcp, 7778/udp, 27015/tcp, 32330/tcp, 27015/udp                              ark_ark.1.b5ifdto69xfdzw9p0wy800dhu

`docker exec -it container_id bash` to open a shell

    docker exec -it 015cc0306a64 bash

    root@015cc0306a64:/ark#

Verify that Ark is running with `arkmanager` commands

    arkmanager status

    Running command 'status' for instance 'main'
     Server running:   Yes 
     Server listening:   Yes 
    Server Name: Ark Docker - (v279.275)
    Players: 0 / 70
     Server online:   No 
     Server version:   2762965 

Then, sign into Steam and test that the server is accessible.

Navigate to `View > Servers`

![steam_view_servers.png](https://github.com/jahrik/homelab-ark/raw/master/images/steam_view_servers.png)

Next, click on the `Favorites > ADD A SERVER`.  Put in the private IP of the Docker swarm host.  Example `192.168.123.123`. Click on `FIND GAMES AT THIS ADDRESS` or specify the port if you know it.  Finally, select the Docker Ark server and click on `ADD SELECETED GAME SERVER TO FAVORITES`

![steam_add_servers.png](https://github.com/jahrik/homelab-ark/raw/master/images/steam_add_servers.png)

Start the game and find the server under the favorites tab.

![ark_favorites.png](https://github.com/jahrik/homelab-ark/raw/master/images/ark_favorites.png)

HAVE FUN!!!

![ark_have_fun.png](https://github.com/jahrik/homelab-ark/raw/master/images/ark_have_fun.png)

## Next

* Because I'm deploying the Jenkins Master to Docker Swarm,
  * I'm unable to build any sort of virtualisation on the master itself.
  * I need a separate hardware box to act as a Jenkins slave for this.
    * Using an old laptop with a base install of ubuntu 18.04
    * Few other packages installed.
      * Ansible
      * Vagrant
      * Virtualbox
      * to name a few
    * For running all tests on the playbooks before sending it to production.
    * I'm going with Ubuntu Desktop for when it comes time to test a VM that needs a GUI for something.
  * I'm currently able to run tests in molecule locally in the workstation.
    * I want that functionality in Jenkins as well.
    * A slave will be the easiest way to accomplish this.

* I still need to figure out a way to pass the environment variable to the stack on deployment...
  * Set variables in AWX inventory after it's already pulled from Github
  * Set ENV vars in Jenkins and pass them in during the API call to AWX
