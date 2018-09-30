## STEP 6: Clean-up (optional)

When you are done with this session or you want to restart from scratch, you can reset the whole demo contents with the following commands.

### Stop services

    docker-compose down

### Logout from GitLab private docker registry

    docker logout gitlab.session1.techlunch.com:4567

### Permanently delete persistent data

    rm -Rf ./gitlab ./gitlab-runner
