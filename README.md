# Tech Lunch | Session 1 | GitLab CI

## Prerequisites

- Git: https://git-scm.com/
- Docker + docker-compose: https://www.docker.com/get-started
- Run <pre>docker pull gitlab/gitlab-ce:latest</pre> (this will speed the next steps :) )
- Run <pre>docker pull gitlab/gitlab-runner:latest</pre> (this too :) )

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

A *docker runner* executes it's job inside a docker image. In our case we will setup a **docker socket binding** execution by running the job inside a docker image which have docker available through the bind-mount */var/run/docker.sock* (for more details: https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding).

A *group runner* is a runner that can be picked up by any job of any project that belongs to the group which the runner is associated to (as long as the runner is configured to run for any tag or the job is marked with a matching tag).

Visit the 'tech-lunch' group's CI/CD settings section (http://gitlab.session1.techlunch.com:9980/groups/tech-lunch/-/settings/ci_cd), expand 'Runners' section and copy the registration token, then run the following command after having relaced *GROUP-TOKEN-HERE* with the value you've just copied:

    REGISTRATION_TOKEN=GROUP-TOKEN-HERE
    docker exec -it session-1-gitlab-ci_gitlab-runner_1 gitlab-runner register \
        --non-interactive \
        --url "http://gitlab.session1.techlunch.com:9980/" \
        --registration-token "$REGISTRATION_TOKEN" \
        --description "docker-group-runner" \
        --run-untagged \
        --locked="false" \
        --executor "docker" \
        --env "HOST_BUILDS_VOLUME_PREFIX=$(pwd)/gitlab-runner" \
        --env "HOST_NETWORK=session-1-gitlab-ci_default" \
        --env "GITLAB_SERVER_BASE_URL=http://gitlab.session1.techlunch.com:9980/" \
        --docker-image docker:stable \
        --docker-network-mode session-1-gitlab-ci_default \
        --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
        --docker-volumes "$(pwd)/gitlab-runner/builds:/builds"

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
            - docker run --rm hello-world

As soon as the file gets pushed to the repo, a new pipeline will start, you can check the output of the first execution here: http://gitlab.session1.techlunch.com:9980/tech-lunch/hello-world/pipelines

#### Improve the CI/CD stages for the first project

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
            - docker run --rm alpine /bin/sh -c "echo 'fake build starting...' && sleep 3 && echo '...fake build done!'"

    unit-tests:
        stage: test
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake unit tests starting...' && sleep 6 && echo '...fake unit tests done!'"

    lint-tests:
        stage: test
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake lint starting...' && sleep 2 && echo '...fake lint done!'"

    static-code-analysis:
        stage: test
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake static code analysis starting...' && sleep 4 && echo '...fake static code analysis done!'"

    package-and-deploy:
        stage: deploy
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake packaging starting...' && sleep 2 && echo '...fake packaging done!'"
            - docker run --rm alpine /bin/sh -c "echo 'fake deploy starting...' && sleep 2 && echo '...fake deploy done!'"

    trigger-downstream-pipelines:
        stage: downstream
        script:
            - docker run --rm alpine /bin/sh -c "echo 'trigger downstream pipelines is coming soon ;)'"

#### Increase GitLab runner concurrency

As you may have noticed, as of now, even if there are more than one job defined for a specific stage, these jobs are being run sequentially.
This is because when registering a runner, the default concurrency of 1 cannot be changed (there are several open issues on this).
However it's possible to tweak runners configuration at any time by editing the config.toml file.
To increase the concurrency level to 5 jobs, run the following command:

    docker exec -it session-1-gitlab-ci_gitlab-runner_1 bin/sh -c "sed -i -E 's/^concurrent = [0-9]+$/concurrent = 5/' /etc/gitlab-runner/config.toml && cat /etc/gitlab-runner/config.toml"

The ruuner configurations is automatically reloaded (you can check the gitlab-runner log for it).

Now you can retrigger the pipeline you runned in the previous step and see the three jobs in the 'test' stage running in parallel.

### Orchestrate GitLab projects

#### Use GitLab as private docker registry

Since we need to build some private docker images to be used in our internal CI/CD infrastructure, we can leverage GitLab support for private docker registries: https://docs.gitlab.com/ce/user/project/container_registry.html

In order to test the availability of the private docker registry hosted by GitLab, type the following command and enter your GitLab credentials:

    docker login gitlab.session1.techlunch.com:4567

This login will be used directly in the pipelines that build and push images to the private docker registry.

#### Add a subgroup for CI/CD commands

Create a new subgroup inside the 'tech-lunch' group, name it 'ci-cd-commands'.
This subgroup will contain some common CI/CD commands (such as triggering new pipelines or waiting for running pipelines to terminate). These commands will have the form of docker images that can be pulled and runned inside the 'tech-lunch' private group pipelines.

#### Create the first CI/CD command

Inside the subgroup 'ci-cd-commands' create a new project and name it 'cmd-tag-project'.
Add the *.gitlab-ci.yml* file into the newly created project with the following contents just to start the first hello world pipeline for this new project and to verify that the docker login actually succeds:

<pre>
<b>before_script:
    - echo $CI_BUILD_TOKEN | docker login --username=$CI_REGISTRY_USER --password-stdin $CI_REGISTRY

after_script:
    - docker logout $CI_REGISTRY</b>

stages:
    - hello

hello-docker:
    stage: hello
    script:
        - docker run --rm hello-world
</pre>

In the *before_script* section the pipeline will login into the private docker registry so that it can push the image that will be built in this project's pipeline stages. In the *after_script* the pipeline just logs out from the private docker registry.

### Build and push the first docker image

Click on the 'WebIDE' button in the project 'cmd-tag-project' home page and use the WebIDE GUI to add a new file inside the repo, name it *Dockerfile* and put the following contents in it:

<pre>
FROM python:3-alpine

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENTRYPOINT [ "python", "./cmd-tag-project.py" ]
</pre>

In the same root directory, create also the two following files:

**requirements.txt**

<pre>
python-gitlab
</pre>

For now, the only library specified in the requirements file will be *python-gitlab* that is a quite useful Python package providing access to the GitLab API (https://python-gitlab.readthedocs.io/en/stable/).

**cmd-tag-project.py**

<pre>
import getopt
import gitlab
import sys

def check_mandatory_opt(help_msg, name, value):
    if not value:
        print('Error: missing option "{}"'.format(name))
        print(help_msg)
        sys.exit(2)

def parse_opts(argv):
    help_msg = 'Usage: {} -t &lt;gitlabPrivateToken&gt; -u &lt;gitlabBaseUrl&gt;'.format(argv[0])
    token = None
    base_url = None
    project_group_and_name = None
    tag_name = None
    try:
        optlist, args = getopt.getopt(argv[1:], 'ht:u:p:t:', ['help', 'token=', 'url=', 'projectgroupandname=', 'tagname='])
    except getopt.GetoptError:
        print('Error: unknown option(s)')
        print(help_msg)
        sys.exit(2)
    for opt, val in optlist:
        if opt in ("-h", "--help"):
            print(help_msg)
            sys.exit()
        elif opt in ("-t", "--token"):
            token = val
        elif opt in ("-u", "--url"):
            base_url = val
        elif opt in ("-p", "--projectgroupandname"):
            project_group_and_name = val
        elif opt in ("-t", "--tagname"):
            tag_name = val
    check_mandatory_opt(help_msg, 'token', token)
    check_mandatory_opt(help_msg, 'url', base_url)
    check_mandatory_opt(help_msg, 'projectgroupandname', project_group_and_name)
    check_mandatory_opt(help_msg, 'tagname', tag_name)
    return base_url, token, project_group_and_name, tag_name

def init_connection_to_gitlab(base_url, token):
    return gitlab.Gitlab(base_url, private_token=token)

def tag_gitlab_project(gl, project_group_and_name, tag_name):
    print('tagging project "{}" with tag "{}"'.format(project_group_and_name, tag_name))
    project = gl.projects.get(project_group_and_name)
    project.tags.create({'tag_name': tag_name, 'ref': 'master'})

def main(argv):
    base_url, token, project_group_and_name, tag_name = parse_opts(argv)
    gl = init_connection_to_gitlab(base_url, token)
    tag_gitlab_project(gl, project_group_and_name, tag_name)

if __name__ == "__main__":
    main(sys.argv)
</pre>

The above script is accepting as input options:
- an url pointing to a GitLab server
- an API token
- the group/name of the project to be tagged
- the tag name to be created

Now edit the *.gitlab-ci.yml* file by replacing its contents with the following:

<pre>
variables:
    IMAGE_VERSION: 0.0.1

before_script:
    - echo $CI_BUILD_TOKEN | docker login --username=$CI_REGISTRY_USER --password-stdin $CI_REGISTRY

after_script:
    - docker logout $CI_REGISTRY

stages:
    - build and push docker image

build-and-push:
    stage: build and push docker image
    script:
        - docker build --tag $CI_REGISTRY_IMAGE:$IMAGE_VERSION .
        - docker build --tag $CI_REGISTRY_IMAGE:latest .
        - docker push $CI_REGISTRY_IMAGE:$IMAGE_VERSION
        - docker push $CI_REGISTRY_IMAGE:latest
</pre>

The newly defined pipeline stage will build and push the *cmd-tag-project* docker image.

Now use the WebIDE GUI to stage and commit all the new and changed files and see the new pipeline running.

#### Create an user with read/write PAT to invoke GitLab API from CI/CD commands

Since this is still an open issue for the CE version of GitLab: https://gitlab.com/gitlab-org/gitlab-ce/issues/41084, we have to create a *fake* user and issue a PAT (Personal Access Token) that we'll pass to the CI/CD commands that work with GitLab API.

Go to GitLab GUI Admin area, go to Users area and click button 'New user'. Give the new user 'ci-cd-executor' as both name and username and 'ci-cd-executor@nowhere.com' as email. Click button 'Create user' and then 'Edit' button and enter a password for this user and 'Save changes'.
Go to Groups area, enter the 'tech-lunch' group and add the user 'ci-cd-executor' as *Maintainer* of the group.
Now logout from 'root' user and log back in as 'ci-cd-executor'.
Once a new passowrd has been set and logged in again, click on the top right user avatar and access the 'Settings' area and then the 'Access tokens' area.
Add a new personal access token with name 'ci-cd-execution' and the 'api' scope checked. Copy the newly created token value and log back as 'root'.
Access the CI/CD settings area for the 'tech-lunch' group and open the 'Variables' section, here add a new protected variable with name 'COMMANDS_API_TOKEN' and value the personal access token value you just copied.

#### Create a GitLab project for running service tests

Go to the 'tech-lunch' group, click on the "New project" button. Enter 'service-tests' as project name and click 'Create project' button.

Then add a *.gitlab-ci.yml* file to this new repo with the following basic stage just to try out the command that we just builded and pushed in the previous steps:

<pre>
variables:
    CMD_TAG_PROJECT_IMAGE: "gitlab.session1.techlunch.com:4567/tech-lunch/ci-cd-commands/cmd-tag-project:0.0.1"
    CMD_TAG_PROJECT: "--rm --network $HOST_NETWORK $CMD_TAG_PROJECT_IMAGE --url $GITLAB_SERVER_BASE_URL --token $COMMANDS_API_TOKEN --projectgroupandname tech-lunch/service-tests --tagname ${CI_PROJECT_PATH_SLUG}_${CI_COMMIT_SHA}"

before_script:
    - echo $CI_BUILD_TOKEN | docker login --username=$CI_REGISTRY_USER --password-stdin $CI_REGISTRY

after_script:
    - docker logout $CI_REGISTRY

stages:
    - hello

tag-project:
    stage: hello
    only:
        - master
    script:
        - docker pull $CMD_TAG_PROJECT_IMAGE
        - docker run $CMD_TAG_PROJECT
</pre>

Once the pipeline that has just been created completes, a new tag comes into the repo (http://gitlab.session1.techlunch.com:9980/tech-lunch/service-tests/tags)

#### Trigger service-tests after successful hello-world project builds

Now edit again the *.gitlab-ci.yml* file in the 'service-tests' project and replace its exisiting content with the following:

    stages:
        - service tests

    run-on-upstream:
        stage: service tests
        only:
            - tags
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake run service tests on upstream change'"

    run-on-commit:
        stage: service tests
        only:
            - branches
        script:
            - docker run --rm alpine /bin/sh -c "echo 'fake run service tests on commit'"

Then edit the *.gitlab-ci.yml* file in the 'hello-world' project and replace its exisiting content with the following:

<pre>
stages:
    - build
    - test
    - deploy
    - downstream

build-some-stuff:
    stage: build
    script:
        - docker run --rm alpine /bin/sh -c "echo 'fake build starting...' && sleep 3 && echo '...fake build done!'"

unit-tests:
    stage: test
    script:
        - docker run --rm alpine /bin/sh -c "echo 'fake unit tests starting...' && sleep 6 && echo '...fake unit tests done!'"

lint-tests:
    stage: test
    script:
        - docker run --rm alpine /bin/sh -c "echo 'fake lint starting...' && sleep 2 && echo '...fake lint done!'"

static-code-analysis:
    stage: test
    script:
        - docker run --rm alpine /bin/sh -c "echo 'fake static code analysis starting...' && sleep 4 && echo '...fake static code analysis done!'"

package-and-deploy:
    stage: deploy
    script:
        - docker run --rm alpine /bin/sh -c "echo 'fake packaging starting...' && sleep 2 && echo '...fake packaging done!'"
        - docker run --rm alpine /bin/sh -c "echo 'fake deploy starting...' && sleep 2 && echo '...fake deploy done!'"

<b>trigger-downstream-pipelines:
    stage: downstream
    variables:
        CMD_TAG_PROJECT_IMAGE: "gitlab.session1.techlunch.com:4567/tech-lunch/ci-cd-commands/cmd-tag-project:0.0.1"
        CMD_TAG_SERVICE_TESTS_PROJECT: "--rm --network $HOST_NETWORK $CMD_TAG_PROJECT_IMAGE --url $GITLAB_SERVER_BASE_URL --token $COMMANDS_API_TOKEN --projectgroupandname tech-lunch/service-tests --tagname ${CI_PROJECT_PATH_SLUG}_${CI_COMMIT_SHA}"
    script:
        - docker pull $CMD_TAG_PROJECT_IMAGE
        - docker run $CMD_TAG_SERVICE_TESTS_PROJECT

before_script:
    - echo $CI_BUILD_TOKEN | docker login --username=$CI_REGISTRY_USER --password-stdin $CI_REGISTRY

after_script:
    - docker logout $CI_REGISTRY</b>
</pre>

Now, after each and every successful build of the 'hello-world' project, a new tag is created in the 'service-tests' project and a new pipeline runs the service tests.

### Monitor GitLab pipelines

TODO

## Clean-up (optional)

### Stop services

    docker-compose down

### Logout from GitLab private docker registry

    docker logout gitlab.session1.techlunch.com:4567

### Permanently delete persistent data

    rm -Rf ./gitlab ./gitlab-runner
