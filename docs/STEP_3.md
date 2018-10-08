## STEP 3: Orchestrate GitLab projects

Since we want to have some useful CI/CD scripts to be shared and re-used in our internal CI/CD infrastructure, we can leverage the GitLab private docker registry to build and internally publish some private docker images to be used as `commands` to be runned in pipelines' scripts.

### Add a subgroup for CI/CD commands

Create a new subgroup inside the `tech-lunch` group, name it `ci-cd-commands`.
This subgroup will contain some common CI/CD commands (such as triggering new pipelines or waiting for running pipelines to terminate). These commands will have the form of docker images that can be pulled and runned inside the `tech-lunch` private group pipelines.

### Create the first CI/CD command

Inside the subgroup `ci-cd-commands` create a new project and name it `cmd-tag-project`.
Add the `.gitlab-ci.yml` file into the newly created project with the following contents just to start the first hello world pipeline for this new project and to verify that the docker login actually succeds:

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

In the `before_script` section the pipeline will login into the private docker registry so that it can push the image that will be built in this project. With `after_script` the pipeline just logs out from the private docker registry.

### Build and push the first docker image

Click on the `WebIDE` button in the project `cmd-tag-project` home page and use the WebIDE GUI to add a new file inside the repo, name it `Dockerfile` and put the following contents in it:

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
python-gitlab==1.6.0
click==7.0
</pre>

The libraries specified in the requirements file are:
- `python-gitlab`: a quite useful Python package providing access to the GitLab API (https://python-gitlab.readthedocs.io/en/stable/).
- `click`: a Python package for easily creating command line interfaces.

**cmd-tag-project.py**

<pre>
import click
import gitlab

@click.command()
@click.option('--base_url', help='The GitLab server base URL (e.g: http://gitlab.session1.techlunch.com:9980/)')
@click.option('--api_access_token', help='A GitLab API token')
@click.option('--project_group_and_name', help='The group/name of the project to be tagged (e.g: tech-lunch/service-test)')
@click.option('--tag_name', help='/he tag name to be created')
def main(base_url, api_access_token, project_group_and_name, tag_name):
    print('tagging project "{}" with tag "{}"'.format(project_group_and_name, tag_name))
    (gitlab
        .Gitlab(base_url, private_token=api_access_token)
        .projects
        .get(project_group_and_name)
        .tags
        .create({'tag_name': tag_name, 'ref': 'master'}))

if __name__ == "__main__":
    main()
</pre>

The above script is accepting as input options:
- the GitLab server base URL (e.g: http://gitlab.session1.techlunch.com:9980/)
- a GitLab API token
- the group/name of the project to be tagged (e.g: tech-lunch/service-test)
- the tag name to be created

Now edit the `.gitlab-ci.yml` file by replacing its contents with the following:

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

The newly defined pipeline stage will build and push the `cmd-tag-project` docker image.

Now use the WebIDE GUI to stage and commit all the new and changed files and see the new pipeline running.

Here you can inspect the images built and pushed by the pipeline that has just runned: http://gitlab.session1.techlunch.com:9980/tech-lunch/ci-cd-commands/cmd-tag-project/container_registry

### Create an user with read/write PAT to invoke GitLab API from CI/CD commands

Since this is still an open issue for the CE version of GitLab: https://gitlab.com/gitlab-org/gitlab-ce/issues/41084, we have to create a *fake* user and issue a PAT (Personal Access Token) that we'll pass to the CI/CD commands that work with GitLab API.

- Go to GitLab GUI Admin area, go to Users area and click button `New user`.
- Give the new user `ci-cd-executor` as both name and username and `ci-cd-executor@nowhere.com` as email.
- Click button `Create user`.
- Go to Groups area, enter the `tech-lunch` group and add the user `ci-cd-executor` as `Maintainer` of the group.
- Go back to Users area, enter the `ci-cd-executor` user and click `Impersonate` button.
- Click on the top right `ci-cd-executor` user avatar and access his `Settings` area.
- Access the `Access tokens` area.
- Add a new personal access token with name `ci-cd-execution` and the `api` scope checked.
- Click `Create personal access token` button.
- Copy the newly created token value.
- Click on the top right button to stop impersonating the `ci-cd-executor` user.
- Access the CI/CD settings area for the `tech-lunch` group and open the `Variables` section.
- Add a new protected variable with name `COMMANDS_API_TOKEN` and as value the personal access token you just copied.

### Create a GitLab project for running service tests

Go to the `tech-lunch` group, click on the "New project" button. Enter `service-tests` as project name and click `Create project` button.

Then add a `.gitlab-ci.yml` file to this new repo with the following basic stage just to try out the command that we just builded and pushed in the previous steps:

<pre>
variables:
    CMD_TAG_PROJECT_IMAGE: "gitlab.session1.techlunch.com:4567/tech-lunch/ci-cd-commands/cmd-tag-project:0.0.1"
    CMD_TAG_PROJECT: "--rm --network $HOST_NETWORK $CMD_TAG_PROJECT_IMAGE --base_url $GITLAB_SERVER_BASE_URL --api_access_token $COMMANDS_API_TOKEN --project_group_and_name tech-lunch/service-tests --tag_name ${CI_PROJECT_PATH_SLUG}_${CI_COMMIT_SHA}_${CI_JOB_ID}"

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

### Trigger service-tests after successful hello-world project builds

Now edit again the `.gitlab-ci.yml` file in the `service-tests` project and replace its exisiting content with the following:

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

Then edit the `.gitlab-ci.yml` file in the `hello-world` project and replace its exisiting content with the following:

<pre>
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

<b>trigger-downstream-pipelines:
    stage: downstream
    variables:
        CMD_TAG_PROJECT_IMAGE: "gitlab.session1.techlunch.com:4567/tech-lunch/ci-cd-commands/cmd-tag-project:0.0.1"
        CMD_TAG_SERVICE_TESTS_PROJECT: "--rm --network $HOST_NETWORK $CMD_TAG_PROJECT_IMAGE --base_url $GITLAB_SERVER_BASE_URL --api_access_token $COMMANDS_API_TOKEN --project_group_and_name tech-lunch/service-tests --tag_name ${CI_PROJECT_PATH_SLUG}_${CI_COMMIT_SHA}_${CI_JOB_ID}"
    script:
        - docker pull $CMD_TAG_PROJECT_IMAGE
        - docker run $CMD_TAG_SERVICE_TESTS_PROJECT

before_script:
    - echo $CI_BUILD_TOKEN | docker login --username=$CI_REGISTRY_USER --password-stdin $CI_REGISTRY

after_script:
    - docker logout $CI_REGISTRY</b>
</pre>

Now, after each and every successful build of the `hello-world` project, a new tag is created in the `service-tests` project and a new pipeline runs the service tests.

## STEP 4: [Monitor GitLab pipelines](STEP_4.md)