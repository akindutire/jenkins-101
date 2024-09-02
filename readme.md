
## YouTube Link
For the full 1 hour course watch on youtube:
https://www.youtube.com/watch?v=6YZvp2GwT0A

# Installation
## Build the Jenkins BlueOcean Docker Image (or pull and use the one I built)
```
docker build -t myjenkins-blueocean:2.4747 .

#IF you are having problems building the image yourself, you can pull from my registry (It is version 2.332.3-1 though, the original from the video)

docker pull devopsjourney1/jenkins-blueocean:2.332.3-1 && docker tag devopsjourney1/jenkins-blueocean:2.332.3-1 myjenkins-blueocean:2.332.3-1
```

## Create the network 'jenkins'
```
docker network create jenkins
```

## Run the Container
### MacOS / Linux
```
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.4747
```

### Windows
```
docker run --name jenkins-blueocean --restart=on-failure --detach `
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 `
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 `
  --volume jenkins-data:/var/jenkins_home `
  --volume jenkins-docker-certs:/certs/client:ro `
  --publish 8080:8080 --publish 50000:50000 myjenkins-blueocean:2.414.2
```


## Get the Password
```
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

## Connect to the Jenkins
```
https://localhost:8080/
```

## Installation Reference:
https://www.jenkins.io/doc/book/installing/docker/


## alpine/socat container to forward traffic from Jenkins to Docker Desktop on Host Machine

https://stackoverflow.com/questions/47709208/how-to-find-docker-host-uri-to-be-used-in-jenkins-docker-plugin
```
docker run -d --restart=always -p 127.0.0.1:2376:2375 --network jenkins -v /var/run/docker.sock:/var/run/docker.sock alpine/socat tcp-listen:2375,fork,reuseaddr unix-connect:/var/run/docker.sock --name proxier
docker inspect <container_id> | grep IPAddress
```

## Using my Jenkins Python Agent
```
docker pull devopsjourney1/myjenkinsagents:python
```

## You may use custom
*Agent docker image:* 
- jenkins/agent:alpine-jdk17 or
- cwdigitalservices/jenkins-agent-deployer:latest
- Don't forget to add *type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock* in Mount section of the container settings when adding the docker agent template
- Install Bitbucket plugin
- Setup webhook jenkins-url:port/bitbucket-hook/
- Add push, PR merge to webhook event
- In your jenkins UI pipeline job, enable the poll SCM (with no interval defined i.e empty interval)
- Enable bitbucket push
- Within your pipeline script options, make sure to skipDefaultCheckout(true)
- Create a custom checkout stage using git tool and a predefined credential (username and password)
- Ref: https://community.atlassian.com/t5/Agile-questions/Bitbucket-webhook-is-not-triggering-the-build-in-jenkins/qaq-p/729199
- Clean up your jenkins regularly using *docker system prune --all --volumes --force*

## Necessary plugin for builds
- Environment Injector

## Clean up
```
mkdir -p /var/log/jenkins/
Add the below

#!/bin/bash

# Log file
LOG_FILE="/var/log/jenkins/docker_cleanup.log"

# Ensure log directory exists
mkdir -p $(dirname $LOG_FILE)

# Function to log messages
log_message() {
    echo "$(date): $1" >> $LOG_FILE
}

log_message "Starting Docker overlay cleanup"

# Find and remove unused overlays
for overlay in $(sudo find /var/lib/docker/overlay2 -mindepth 1 -maxdepth 1 -type d); do
    if ! docker ps -qa --filter volume=$overlay | grep -q .; then
        log_message "Removing unused overlay: $overlay"
        sudo rm -rf $overlay
    fi
done

# Run a system prune to clean up other unused resources
log_message "Running Docker system prune"
docker system prune -af --volumes >> $LOG_FILE 2>&1

log_message "Docker overlay cleanup completed"

# Output disk usage after cleanup
log_message "Current disk usage:"
df -h >> $LOG_FILE

exit 0
```

- crontab -e
- 0 2 * * * docker system prune -af --volumes && ./cleanup.sh >> /dev/null 2>&1 //2am everyday
