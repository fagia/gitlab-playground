# Tech Lunch | Session 1 | GitLab CI

## Prerequisites

- Git: https://git-scm.com/
- Docker + docker-compose: https://www.docker.com/get-started

## Step-by-step guide

Add the following entry to your */etc/hosts* file:

    127.0.0.1       gitlab.session1.techlunch.com

Clone this repository and start the services with docker-compose:

    git clone https://github.com/fagia/tech-lunch-session-1-gitlab-ci.git session-1-gitlab-ci
    cd session-1-gitlab-ci
    docker-compose up -d

You can inspect the GitLab startup logs with:

    docker-compose logs -f gitlab

After starting the services, you can visit the GitLab web GUI here: http://gitlab.session1.techlunch.com:9980/. Since this is the first startup, it might take a while before the GitLab Docker container starts to respond to queries.
