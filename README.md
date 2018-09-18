# Tech Lunch | Session 1 | GitLab CI

## Prerequisites

- Git: https://git-scm.com/
- Docker + docker-compose: https://www.docker.com/get-started

## Step-by-step guide

### Start services

Add the following entry to your */etc/hosts* file:

    127.0.0.1       gitlab.session1.techlunch.com

Clone this repository and start the services with docker-compose:

    git clone https://github.com/fagia/tech-lunch-session-1-gitlab-ci.git session-1-gitlab-ci
    cd session-1-gitlab-ci
    docker-compose up -d

You can inspect the GitLab startup logs with:

    docker-compose logs -f gitlab.session1.techlunch.com

### Reset GitLab root password

After starting the services, visit the GitLab web GUI here: http://gitlab.session1.techlunch.com:9980/. Since this is the first startup, it might take a while before the GitLab Docker container starts to respond to queries. The initial setup should have been successfully completed once you read 'gitlab Reconfigured!' in the logs.

    docker-compose logs -f gitlab.session1.techlunch.com 2>&1 | grep 'gitlab Reconfigured!'

Enter and confirm the new password for the newly create user *root*. Click 'Change your password' button.

Now you can log in to the GitLab web GUI with default user 'root' and the password you have just entered.

### Change GitLab's default user username

After you logged in to the GitLab web GUI, click on the top right of your icon profile, and click the 'Settings' icon to setup your profile.

Go to the 'Account' tab and change the default root username with your own username, then click the 'Update username' button.

Now you can log out and log back into the GitLab web GUI with the username you have just entered and the password you have entered before.

### Create a GitLab group

After you logged in to the GitLab web GUI, click on the "Create a group" link. Enter 'tech-lunch' as group path and click 'Create group' button.

### Create a GitLab project

After having created the 'tech-lunch' group, click on the "New project" button. Enter 'hello-world' as project name and click 'Create project' button.

### Configure the GitLab CI runner

Visit the 'tech-lunch' group's CI/CD settings section: http://gitlab.session1.techlunch.com:9980/groups/tech-lunch/-/settings/ci_cd, expand 'Runners' section and copy the registration token, then run the following command after having relaced *<GROUP-TOKEN-HERE>* with the value you've just copied:

    REGISTRATION_TOKEN=<GROUP-TOKEN-HERE>
    docker exec -it session-1-gitlab-ci_gitlab-runner_1 gitlab-runner register \
        --non-interactive \
        --url "http://gitlab.session1.techlunch.com:9980/" \
        --registration-token "$REGISTRATION_TOKEN" \
        --description "docker-runner" \
        --run-untagged \
        --locked="false" \
        --executor "docker" \
        --docker-image docker:stable \
        --docker-network-mode session-1-gitlab-ci_default \
        --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"

You can inspect the GitLab Runner configuration logs with:

    docker-compose logs -f gitlab-runner

### Add CI/CD to first project

Go to 'hello-world' project and click 'New file' button in order to create the *.gitlab-ci.yml* file which will define the specific CI/CD stages that will be executed for this project.

Paste the following as the *.gitlab-ci.yml* file content:

    image: docker:stable

    before_script:
    - docker info

    stages:
    - hello

    say-hello:
        stage: hello
        script:
            - docker run hello-world

## Clean-up

### Stop services

    docker-compose down

### Permanently delete persistent data

    cd .. && rm -Rf session-1-gitlab-ci
