## STEP 6: Clean-up (optional)

When you are done with this session or you want to restart from scratch, you can reset the whole demo contents with the following commands.

### Stop services

    docker-compose down

### Logout from GitLab private docker registry

    docker logout gitlab.playground.test:4567

### Permanently delete persistent data

    rm -Rf ./gitlab ./gitlab-runner

### Prune docker system

https://docs.docker.com/engine/reference/commandline/system_prune/

Warning: this could remove other docker elements not related to this tutorial, make you sure you know what are you going to prune before executing the command!
