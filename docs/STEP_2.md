## STEP 2: Use GitLab as private docker registry

GitLab has native support for private docker registries: https://docs.gitlab.com/ce/user/project/container_registry.html

In order to test the availability of the private docker registry hosted by GitLab, type the following command and enter your GitLab credentials:

    docker login gitlab.playground.test:4567

This login will be used directly in the pipelines that build and push images to the private docker registry. Each project can build and push it's own image to the private docker registry.

This private docker registry can have several uses, first of all can be used as the docker images source for deploying and distributing our applications. Another possible use of the private docker registry will be shown in the next section.

## STEP 3: [Orchestrate GitLab projects](STEP_3.md)