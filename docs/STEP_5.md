## STEP 5: GitLab CI/CD goodies

### Templates

Once the application starts growing, it's often useful to be able to re-use common CI/CD patterns via some templating mechanisms. Templates can ease changing and maintaining the CI/CD infrastructure by avoiding to have the common tasks replicated in many different places.

GitLab CE allows to do this starting with the 11.4 CE version with the `include` keyword that allows the inclusion of external YAML files in the `.gitlab-ci.yml` file.

One important caveat (and limitation) is that the included file will have to be publicly accessible: so be careful to do not put any sensitive data inside the template files!

For example, the triggering of the service-tests from any other projects can be done by including the same template in different projects.

Go to the `Snippets` page (http://gitlab.playground.test:9980/dashboard/snippets) and create a new snippet with title `Trigger service-tests`, `public` visibility level, `trigger-service-tests.yml` as filename and the following content:

<pre>
.trigger-downstream-pipelines:
    variables:
        CMD_TAG_PROJECT_IMAGE: "gitlab.playground.test:4567/playground/ci-cd-commands/cmd-tag-project:0.0.1"
        CMD_TAG_SERVICE_TESTS_PROJECT: "--rm --network $HOST_NETWORK $CMD_TAG_PROJECT_IMAGE --base_url $GITLAB_SERVER_BASE_URL --api_access_token $COMMANDS_API_TOKEN --project_group_and_name playground/service-tests --tag_name ${CI_PROJECT_PATH_SLUG}_${CI_COMMIT_SHA}_${CI_JOB_ID}"
    before_script:
        - echo $CI_BUILD_TOKEN | docker login --username=$CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    script:
        - docker pull $CMD_TAG_PROJECT_IMAGE
        - docker run $CMD_TAG_SERVICE_TESTS_PROJECT
    after_script:
        - docker logout $CI_REGISTRY
</pre>

Now replace the content of the `.gitlab-ci.yml` file of the `hello-world` project with the following:

<pre>
<b>include: "http://gitlab.playground.test:9980/snippets/1/raw?.yml"</b>

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
    extends: .trigger-downstream-pipelines</b>
</pre>

The triggering of the `service-tests` pipeline is now made by including the public snippet `http://gitlab.playground.test:9980/snippets/1/raw?.yml` (the `?.yml` suffix is an hack to make GitLab accept to include the referenced snippet... see https://gitlab.com/gitlab-org/gitlab-ce/issues/48446) and then referencing the `.trigger-downstream-pipelines` element from the snippet as an extension to the `trigger-downstream-pipelines` job. 

### Artifacts

Artifacts are a list of files and directories which are attached to a job and that are browsable/downloadable via GUI and sharable with following jobs as dependencies after the job completes successfully (artifacts can attached also on job failure but this is a little bit tricky to achieve).

### Coverage

Coverage can be automatically extracted (scripts + regexes) and published as job putput.

## STEP 6: [Clean-up (optional)](STEP_6.md)
