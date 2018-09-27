## STEP 5: Clean-up (optional)

### Stop services

    docker-compose down

### Logout from GitLab private docker registry

    docker logout gitlab.session1.techlunch.com:4567

### Permanently delete persistent data

    rm -Rf ./gitlab ./gitlab-runner
