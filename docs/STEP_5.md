## STEP 5: GitLab CI/CD goodies

### Artifacts

Artifacts are a list of files and directories which are attached to a job and that are browsable/downloadable via GUI and sharable with following jobs as dependencies after the job completes successfully (artifacts can attached also on job failure but this is a little bit tricky to achieve).

### Coverage

Coverage can be automatically extracted (scripts + regexes) and published as job putput.

### Templates

Once the application starts growing, it's always useful to be able to re-use common CI/CD patterns via some templating mechanisms.

GitLab CE will allow to do this starting with the 11.4 version (now we're at 11.3 and for now this feature is limited to EE only) by the `include` keyword that allows the inclusion of external YAML files in the `.gitlab-ci.yml` file.

One important caveat (and limitation) is that the included file will have to be public accessible.

The triggering of the service-tests from any other projects can then became as simple as defining a template (eg. as a GitLab pubblic snippet) for this kind of job and include it in the right `.gitlab-ci.yml` file.

## STEP 6: [Clean-up (optional)](STEP_6.md)
