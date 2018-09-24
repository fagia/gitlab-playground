# Tech Lunch | Session 1 | GitLab CI

## Prerequisites

- Git: https://git-scm.com/
- Docker + docker-compose: https://www.docker.com/get-started

## Step-by-step guide

### Create a GitLab project and add CI/CD capabilities to it

#### Start GitLab server

Add the following entry to your */etc/hosts* file:

    127.0.0.1       gitlab.session1.techlunch.com

Clone this repository and start the services with docker-compose:

    git clone https://github.com/fagia/tech-lunch-session-1-gitlab-ci.git session-1-gitlab-ci
    cd session-1-gitlab-ci
    docker-compose up -d

You can inspect the GitLab startup logs with:

    docker-compose logs -f gitlab.session1.techlunch.com

#### Reset GitLab root password

Since this is the first startup, it might take a while before the GitLab Docker container starts to respond to queries. The initial setup should have been successfully completed once you read 'gitlab Reconfigured!' in the logs.

    docker-compose logs -f gitlab.session1.techlunch.com 2>&1 | grep 'gitlab Reconfigured!'

After the GitLab service has started successfully, visit the GitLab web GUI here: http://gitlab.session1.techlunch.com:9980/. 
Enter and confirm the new password for the newly create user *root*. Click 'Change your password' button.

Now you can log in to the GitLab web GUI with default user 'root' and the password you have just entered.

#### Change GitLab's default user username (optional)

After you logged in to the GitLab web GUI, click on the top right of your icon profile, and click the 'Settings' icon to setup your profile.

Go to the 'Account' tab and change the default root username with your own username, then click the 'Update username' button.

Now you can log out and log back into the GitLab web GUI with the username you have just entered and the password you have entered before.

#### Create a GitLab group

After you logged in to the GitLab web GUI, click on the "Create a group" link. Enter 'tech-lunch' as group path and click 'Create group' button.

#### Configure a GitLab CI group runner

In order to run CI/CD jobs, GitLab needs to have runners. There are various type of runners and you can get a deeper understanding of the available options here: https://docs.gitlab.com/ce/ci/runners/README.html. In this session we will leverage a *docker group runner*.

A *docker runner* executes it's job inside a docker image, in our case we will setup a docker in docker execution by running the job inside a docker image ('docker in docker').

A *group runner* is a runner that can be picked up by any job of any project that belongs to the group which the runner is associated to (as long as the runner is configured to run for any tag or the job is marked with a matching tag).

Visit the 'tech-lunch' group's CI/CD settings section: http://gitlab.session1.techlunch.com:9980/groups/tech-lunch/-/settings/ci_cd, expand 'Runners' section and copy the registration token, then run the following command after having relaced *GROUP-TOKEN-HERE* with the value you've just copied:

    REGISTRATION_TOKEN=GROUP-TOKEN-HERE
    docker exec -it session-1-gitlab-ci_gitlab-runner_1 gitlab-runner register \
        --non-interactive \
        --url "http://gitlab.session1.techlunch.com:9980/" \
        --registration-token "$REGISTRATION_TOKEN" \
        --description "docker-group-runner" \
        --run-untagged \
        --locked="false" \
        --executor "docker" \
        --docker-image docker:stable \
        --docker-network-mode session-1-gitlab-ci_default \
        --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"

You can inspect the GitLab Runner configuration logs with:

    docker-compose logs -f gitlab-runner

Now all the projects that will be created in this group will be able to use this runner.

#### Create a GitLab project

Go to the 'tech-lunch' group, click on the "New project" button. Enter 'hello-world' as project name and click 'Create project' button.

#### Add CI/CD to the first project

Go to 'hello-world' project and click 'New file' button in order to create the *.gitlab-ci.yml* file which will define the specific CI/CD stages that will be executed for this project.

Paste the following as the *.gitlab-ci.yml* file content:

    stages:
        - hello

    hello-docker:
        stage: hello
        script:
            - docker run hello-world

As soon as the file gets pushed to the repo, a new pipeline will start, you can check the output of the first execution here: http://gitlab.session1.techlunch.com:9980/tech-lunch/hello-world/pipelines

#### Improve the CI/CD stages for the first project

Each project has to define the stages it needs to build, validate and deploy each and every commit that is pushed to its git repository.

Here it is a sample of some of the possible stages that a project should define:

TODO

### Orchestrate GitLab projects

#### Use GitLab as private docker registry

Since we need to build some private docker images to be used in our internal CI/CD infrastructure, we can leverage GitLab support for private docker registries: https://docs.gitlab.com/ce/user/project/container_registry.html

In order to test the availability of the private docker registry hosted by GitLab, type the following command and enter your GitLab credentials:

    docker login gitlab.session1.techlunch.com:4567

This login will be used directly in the pipelines that build and push images to the private docker registry.

#### Add a repository for building and pushing private images

Create a new project inside the 'tech-lunch' group, name it 'ci-cd-commands'.
We're going to use this project to be able to share inside the 'tech-lunch' private group some common CI/CD commands such as triggering new pipelines or waiting for running pipelines to terminate.

Add the *.gitlab-ci.yml* file into the newly created project with the following contents:

<pre>
<b>before_script:
    - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN gitlab.session1.techlunch.com:4567</b>

stages:
    - hello

hello-docker:
    stage: hello
    script:
        - docker run hello-world
</pre>

In the *before_script* section the pipeline will login into the private docker registry so that it can push the images that will be built in this project's pipelines.

### Build and push the first docker image

Click on the 'WebIDE' button in the project 'ci-cd-commands' home page and use the WebIDE GUI to add a new folder inside the repo and to name it 'tech-lunch-hello-world' (this is going to be the base directory in which we build a new private private docker image).

Add a new file inside the new dir and name it *Dockerfile*, put the following contents in it:

<pre>

</pre>

#### Create another GitLab project

Go to the 'tech-lunch' group, click on the "New project" button. Enter 'service-tests' as project name and click 'Create project' button.

Then add a *.gitlab-ci.yml* file to this new repo with the following basic stage:

    stages:
        - hello

    hello-docker:
        stage: hello
        script:
            - docker run hello-world

#### Trigger service-tests after each successful hello-world project build

TODO

## Clean-up (optional)

### Stop services

    docker-compose down

### Logout from GitLab private docker registry

    docker logout gitlab.session1.techlunch.com:4567

### Permanently delete persistent data

    rm -Rf ./gitlab ./gitlab-runner
