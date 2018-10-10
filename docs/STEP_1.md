## STEP 1: Create a GitLab project and add CI/CD capabilities to it

### Start GitLab server

Add the following entry to your `/etc/hosts` file:

    127.0.0.1       gitlab.playground.test

Clone this repository and start the services with docker-compose:

    git clone https://github.com/fagia/gitlab-playground.git gitlab-playground
    cd gitlab-playground
    docker-compose up -d

You can inspect the GitLab startup logs with:

    docker-compose logs -f gitlab.playground.test

### Reset GitLab root password

Since this is the first startup, it might take a while before the GitLab Docker container starts to respond to queries. The initial setup should have been successfully completed once you read `gitlab Reconfigured!` in the logs.

    docker-compose logs -f gitlab.playground.test 2>&1 | grep 'gitlab Reconfigured!'

After the GitLab service has started successfully, visit the GitLab web GUI here: http://gitlab.playground.test:9980/. 
Enter and confirm the new password for the newly create user `root`. Click `Change your password` button.

Now you can log in to the GitLab web GUI with default user `root` and the password you have just entered.

### Change GitLab's default user username (optional)

After you logged in to the GitLab web GUI, click on the top right of your icon profile, and click the `Settings` icon to setup your profile.

Go to the `Account` tab and change the default root username with your own username, then click the `Update username` button.

Now you can log out and log back into the GitLab web GUI with the username you have just entered and the password you have entered before.

### Create a GitLab group

After you logged in to the GitLab web GUI, click on the "Create a group" link. Enter `tech-lunch` as group path and click `Create group` button.

### Configure a GitLab CI group runner

In order to run CI/CD jobs, GitLab needs to have runners.

A `GitLab runner` is a process in charge of executing the scripts defined in the `.gitlab-ci.yml` file.

Each GitLab runner has to define an `executor` in which scripts are actually executed. There are various type of executors that a GitLab runner can define (SSH, Shell, VirtualBox, Parallels, Docker or Kubernetes). You can get a deeper understanding of the available options here https://docs.gitlab.com/ce/ci/runners/README.html and here https://docs.gitlab.com/runner/executors/.

In this session we will leverage a `docker group runner`.

A `docker runner` executes it's scripts inside a docker image. In our case we will setup a `docker socket binding` execution by running the scripts inside a docker image which have docker available through the bind-mount of the unix socket that the host's docker daemon listens to (for more details: https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding).

A `group runner` is a runner that can be picked up by any job of any project that belongs to the group which the runner is associated to (as long as the runner is configured to run for any tag or the job is marked with a matching tag).

Visit the `tech-lunch` group's CI/CD settings section (http://gitlab.playground.test:9980/groups/tech-lunch/-/settings/ci_cd), expand `Runners` section and copy the registration token, then run the following command after having relaced `GROUP-TOKEN-HERE` with the value you have just copied:

    REGISTRATION_TOKEN=GROUP-TOKEN-HERE
    docker exec -it gitlab-playground_gitlab-runner_1 gitlab-runner register \
        --non-interactive \
        --url "http://gitlab.playground.test:9980/" \
        --registration-token "$REGISTRATION_TOKEN" \
        --description "docker-group-runner" \
        --run-untagged \
        --locked="false" \
        --executor "docker" \
        --env "HOST_BUILDS_VOLUME_PREFIX=$(pwd)/gitlab-runner" \
        --env "HOST_NETWORK=gitlab-playground_default" \
        --env "GITLAB_SERVER_BASE_URL=http://gitlab.playground.test:9980/" \
        --docker-image docker:stable \
        --docker-network-mode gitlab-playground_default \
        --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
        --docker-volumes "$(pwd)/gitlab-runner/builds:/builds"

You can inspect the GitLab Runner configuration logs with:

    docker-compose logs -f gitlab-runner

Now all the projects that will be created in this group will be able to use this runner.

### Create a GitLab project

Go to the `tech-lunch` group, click on the "New project" button. Enter `hello-world` as project name and click `Create project` button.

### Add CI/CD to the first project

Go to `hello-world` project and click `New file` button in order to create the `.gitlab-ci.yml` file which will define the specific CI/CD stages that will be executed for this project.

Paste the following as the `.gitlab-ci.yml` file content:

    stages:
        - hello

    hello-docker:
        stage: hello
        script:
            - docker run --rm hello-world

As soon as the file gets pushed to the repo, a new pipeline will start, you can check the output of the first execution here: http://gitlab.playground.test:9980/tech-lunch/hello-world/pipelines

### Improve the CI/CD stages for the first project

Each project has to define the stages it needs to build, validate and deploy each and every commit that is pushed to its git repository.

Here it is a sample of some of the possible stages that a project should define:

    stages:
        - build
        - test
        - deploy
        - downstream

    build-some-stuff:
        stage: build
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake build starting...' && echo '...fake build done!'"

    unit-tests:
        stage: test
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake unit tests starting...' && echo '...fake unit tests done!'"

    lint-tests:
        stage: test
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake lint starting...' && echo '...fake lint done!'"

    static-code-analysis:
        stage: test
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake static code analysis starting...' && echo '...fake static code analysis done!'"

    package-and-deploy:
        stage: deploy
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake packaging starting...' && echo '...fake packaging done!'"
            - docker run --rm alpine /bin/sh -c "echo 'fake deploy starting...' && echo '...fake deploy done!'"

    trigger-downstream-pipelines:
        stage: downstream
        script:
            - docker run --rm alpine /bin/sh -c "echo 'trigger downstream pipelines is coming soon ;)'"

### Increase GitLab runner concurrency

As you may have noticed, as of now, even if there are more than one job defined for a specific stage, these jobs are being run sequentially.
This is because when registering a runner, the default concurrency of 1 cannot be changed (there are several open issues on this).
However it's possible to tweak runners configuration at any time by editing the config.toml file.
To increase the concurrency level to 5 jobs, run the following command:

    docker exec -it gitlab-playground_gitlab-runner_1 bin/sh -c "sed -i -E 's/^concurrent = [0-9]+$/concurrent = 5/' /etc/gitlab-runner/config.toml && cat /etc/gitlab-runner/config.toml"

The runner configurations is automatically reloaded (you can check the gitlab-runner log for it).

Now you can retrigger the pipeline you runned in the previous step and see the three jobs in the `test` stage running in parallel.

## STEP 2: [Use GitLab as private docker registry](STEP_2.md)