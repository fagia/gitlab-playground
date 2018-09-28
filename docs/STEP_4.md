## STEP 4: Monitor GitLab pipelines

As of now, The GitLab community edition has not a proper built-in GUI to monitor several projects' pipelines at the same time.
Some open source projects for monitoring GitLab pipelines exist, one of these is called `GitLab Radiator` (https://github.com/heikkipora/gitlab-radiator) and it has a basic set of features that allow a decent level of monitoring capabilities of GitLab pipelines.

In this session we're going to use a fork of the `GitLab Radiator` that has support for running as docker containter: https://github.com/fagia/gitlab-radiator. The require docker container is already running in the docker-compose application, what is missing is to restart it with the GitLab API private access token that has been created in step 3.
Run the following command after having relaced `API_TOKEN_HERE` with the value you can copy from the `tech-lunch` CI/CD variable named `COMMANDS_API_TOKEN`(http://gitlab.session1.techlunch.com:9980/groups/tech-lunch/-/settings/ci_cd):

    docker-compose rm -sf gitlab-radiator
    GITLAB_ACCESS_TOKEN=API_TOKEN_HERE docker-compose up gitlab-radiator

Now you can see the GitLab Radiator GUI here: http://gitlab.session1.techlunch.com:9933/