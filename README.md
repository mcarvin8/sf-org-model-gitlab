# Salesforce Org Model for GitLab CI/CD using the Salesforce CLI (`sf`)
This repository demonstrates how to use GitLab actions, the Salesforce CLI, Salesforce CLI plugins, and bash scripts to validate, deploy, or destroy metadata in a Salesforce org following the org development model, without using packages/scratch orgs.

The model uses these Salesforce CLI plugins:
1. [SFDX Git Delta](https://github.com/scolladon/sfdx-git-delta)
2. [Apex Tests List](https://github.com/renatoliveira/apex-test-list)
3. [Apex Code Coverage Transformer](https://github.com/mcarvin8/apex-code-coverage-transformer)

## CI/CD Model

The CI/CD model in `.gitlab-ci.yml` is the org branching model, where each Salesforce org has its own long-running Git branch. The rules can be updated based on your branching strategy, i.e. update the branches in the rules to target 1 branch upon merge or 1 branch when a merge request is opened against it.

- The `pipeline` stage contains optional ad-hoc jobs.
   - The `rollback` job can be used to roll-back previous deployments via a web-based pipeline. This requires a GitLab project access token with sufficent repo access.
       - the `BOT_NAME` CI/CD variable should be the name of the project access token user
       - the `BOT_USER_NAME` CI/CD variable should be the user name of the project access token user
       - the `PROJECT_TOKEN` CI/CD variable should contain the project access token value
       - Go to the repo, then go to Build > Pipelines. Press "New pipeline". Select the applicable branch in "Run for branch name or tag". Enter 1 new CI/CD variable for the job. The variable key should be `SHA` and hte variable value should be the SHA to revert/roll-back.
    - The `prodBackfill` job can be used depending on your branching strategy. If you create branches from `main` (default branch) but have to merge them into other long-running branches, this job can be used to refresh those long-running branches with changes from `main`.
       - This job can use the same GitLab project access token which the `rollback` job uses and requires the `BOT_NAME`/`BOT_USER_NAME`/`PROJECT_TOKEN` variables.
- The `test` stage contains jobs to either validate metadata before deployment or run all local tests in an org on a schedule. 
    - When a merge request is opened against one of the org branches, it will validate the changes in the org.
    - When you create a scheduled pipeline with the `$JOB_NAME` as "unitTest", it will run all local tests in your org.
        - Define `$AUTH_URL` and `$AUTH_ALIAS` when creating this scheduled pipeline.
        - See inspiration behind this method: https://www.pablogonzalez.io/how-to-schedule-run-all-tests-in-salesforce-with-github-actions-for-unlimited-salesforce-orgs-nothing-to-install/
    - All test jobs uses the [Apex Code Coverage Transformer](https://github.com/mcarvin8/apex-code-coverage-transformer) to create the code coverage artifact. 
        - By default, it creates a SonarQube XML that is passed to the `quality` job for SonarQube. This coverage format can be updated to other ones like Cobertura based on what code quality tool you want to use.
- The `quality` stage runs SonarQube scans if you have SonarQube. The job will run after either `test` job above (pre-deployment validation or unit tests).
- The `destroy` stage contains jobs for each org that will delete the metadata from the org if the files were deleted from the org branch. This job is allowed to fail and will fail if there are no metadata types detected in the destructive package.
    - This will be a standalone destructive deployment that will run before the deployment by default. If you need to deploy the destructive changes after the deployment, cancel the `destroy` stage when the pipeline is created, allow the `deploy` stage to complete, then re-run the `destroy` stage.
    - To destroy Apex in production, you must run tests per Salesforce requirement. Set the `DESTRUCTIVE_TESTS` variable in `.gitlab-ci.yml` with the pre-defined tests to run when destroying Apex in production.
- The `deploy` stage contains jobs for each org that will deploy the metadata to the assigned org after a merge into the org branch.

## Slack Posts

The deployment, test, and destruction statuses can be posted to a Slack channel. Update the webhook variable in the `.gitlab-ci.yml` you use:

``` yaml
  # Update webhook URL here for your slack channel
  SLACK_WEBHOOK_URL: https://hooks.slack.com/services/
```

Delete this variable and the step in each `after_script` section that runs `scripts/bash/deploy_slack_status.sh` if you are not using slack.

## Declare Metadata to Deploy

This org model uses a manifest file (package.xml) to run delta deployments. The SFDX git delta plugin will create a package.xml by comparing the changes between the current commit and previous commit.

This package is then checked to search for Apex Classes, Apex Triggers, and Connected Apps. See below:

### Declare Apex Tests

If Apex classes/trigger are found in the package for validations or deployments, it will run the apex tests list plugin to determine the specified tests to run, instead of running all local tests in the org.

You must add the `@tests:` or `@testsuites:` annotations to each Apex class/trigger per the [Apex Test List plugin documentation](https://github.com/renatoliveira/apex-test-list?tab=readme-ov-file#apex-test-list).

This plugin is not used in destructive deployments.

### Connected Apps

If connected apps are found in the package for validations or deployments, the `<consumerKey>` line in each connected app meta file will be automatically removed before deployment. Deployments with connected apps will fail if you leave the consumer key in the file.

## Branch Protection

### Validation Merge Request Pipelines

In the "Merge requests" settings, you can enable "Pipelines must succeed" to ensure the merge request validation passes before the request can be accepted.

### Protected CI/CD Environments

You can protect each CI/CD environment to limit those who can deploy to the orgs.

The pre-deploy validation environments start with "validate-", which you can allow anyone to validate.

But for destructions and deployments, I would recommend protecting these environments to limit those who can permanently change the target org.

## Bot Deployments

To deploy Einstein Bots, you should update the `.forceignore` file with bot versions to not deploy/retrieve (such as the active bot version) and you should also update the `scripts/replacementFiles` with the Bot User for each org, if you are configuring the bot user. The metadata string replacements are done automatically by the Salesforce CLI before deployment and they are dependent on the `AUTH_ALIAS` variables configure in the `.gitlab-ci.yml`.

If you do not want to use this feature, remove the `replacements` key in the `sfdx-project.json`.

## Other CI/CD Platforms

The bash scripts in `scripts/bash` could work on other CI/CD platforms as long as the container sets these environment variables to match the GitLab predefined CI/CD variables.

The primary scripts to destroy, deploy, and validate metadata are:
- `scripts/bash/deploy_metadata_sf.sh` - To validate and deploy metadata to Salesforce orgs. Tests are set by `scripts/bash/package_check.sh`.
    - `$CI_PIPELINE_SOURCE` must be "push" to be deploy and some other value to validate (like `merge_request_event`) from a merge request/pull request. Only the value "push" is hard-coded into the bash script.
    - `$DEPLOY_PACKAGE` should be the path to the package.xml created by the sfdx-git-delta plugin.
    - `$DEPLOY_TIMEOUT` should be the wait period for the CLI. Set to 240 in the `.gitlab-ci.yml`.
- `scripts/bash/package_check.sh` - To check the package before validating and deploying metadata to Salesforce orgs.
    - `$DEPLOY_PACKAGE` should be the path to the package.xml created by the sfdx-git-delta plugin.
- `scripts/bash/destroy_metadata_sf.sh`
    - `$DESTRUCTIVE_CHANGES_PACKAGE` should be the path to the `destructive/destructiveChanges.xml` created by the sfdx-git-delta plugin. `$DESTRUCTIVE_PACKAGE` should be the path to the `destructive/package.xml` created by the sfdx-git-delta plugin.
    - `$CI_ENVIRONMENT_NAME` must be "prd" for production orgs in order to run apex tests when destroying Apex in production, per Salesforce requirement. Sandbox org names do not matter.
    - `$DEPLOY_TIMEOUT` should be the wait period for the CLI. Set to 240 in the `.gitlab-ci.yml`.   
- `scripts/bash/deploy_slack_status.sh`
    - `$CI_ENVIRONMENT_NAME` must be set with the org name. Optionally, validation environments can start with "validate-", which will be removed in the slack status. This is useful to create separate CI/CD environments for validations and deployments to limit those who can deploy over those who can validate.
    - `$CI_JOB_STAGE` must be "validate", "destroy", or "deploy" to have slack post the right message.
    - `$CI_JOB_STATUS` must be "success" for successful pipelines and some other value for failed pipelines.
    - `GITLAB_USER_NAME`, `CI_JOB_URL`, `CI_PROJECT_URL`, `CI_COMMIT_SHA` should be adjusted for the platform to have the correct details.
