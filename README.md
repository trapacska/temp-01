## Infos (relevant envs) available in the currently running build (in which the step runs)

- BITRISE_APP_SLUG
- BITRISE_BUILD_NUMBER
- BITRISE_GIT_BRANCH
- BITRISEIO_GIT_BRANCH_DEST
- BITRISE_GIT_TAG
- BITRISE_PULL_REQUEST
- BITRISE_GIT_COMMIT

## Bitrise API's `BuildResponseItemModel`

We can start an authenticated (with Bitrise Personal Access Token) request with the app slug available in the currently running build to get the current builds for the current app. Also the API has many query parameters in which the returned build's list can be filtered to avoid too long request time.

**GET** `/apps/{app-slug}/builds`

```JSON
{
  "data": [{
    "abort_reason": "string",
    "branch": "string",
    "build_number": 0,
    "commit_hash": "string",
    "commit_message": "string",
    "commit_view_url": "string",
    "environment_prepare_finished_at": "string",
    "finished_at": "string",
    "is_on_hold": true,
    "original_build_params": "string",
    "pull_request_id": "string",
    "pull_request_target_branch": "string",
    "pull_request_view_url": "string",
    "slug": "string",
    "stack_config_type": "string",
    "stack_identifier": "string",
    "started_on_worker_at": "string",
    "status": 0,
    "status_text": "string",
    "tag": "string",
    "triggered_at": "string",
    "triggered_by": "string",
    "triggered_workflow": "string"
  }]
}
```

## Basic step logic

Based on the model and available infos above.

- Create a step which has one input for Bitrise personal access token(not sure, check build api token if it is able to get the required info) and two output (PREVIOUS_BUILD_STATUS for example and BUILD_STATUS_CHANGED)
- The step should find out what trigger_event_type if the actual build: 
  - BITRISE_GIT_TAG set = tag
  - BITRISE_PULL_REQUEST set = pull request
  - BITRISE_GIT_COMMIT + none of the above = push
  - everything else - manual
- The step should request builds with query parameters(at least): branch, trigger_event_type.
- Find the previous build matching the current build parameters(if PR then PR ID for example, etc...) and check it's status.
- Export an env with the status, for example: `PREVIOUS_BUILD_STATUS=[success or failed or aborted]` and an env BUILD_STATUS_CHANGED if the build status has changed since the previous build.
- The step will not parse the previous build info again and again if the PREVIOUS_BUILD_STATUS env is already exported

Example usage:
- run this step, exported env: PREVIOUS_BUILD_STATUS=failed, BUILD_STATUS_CHANGED=true (at this point the current build is still successful)
- cache-pull step run_if: '{{enveq "PREVIOUS_BUILD_STATUS" "success"}}' (downloads cache if the previous build was successful only)
- run other tasks...
- slack run_if: {{enveq "PREVIOUS_BUILD_STATUS" "failed" | and (not .IsBuildFailed)}} (send message about builds turning to green)
- slack run_if: {{enveq "PREVIOUS_BUILD_STATUS" "success" | and .IsBuildFailed}} (send message about builds turning to failed)
- run this step, exported env: PREVIOUS_BUILD_STATUS=failed, BUILD_STATUS_CHANGED=true (at this point the current build might be failed or successful, set true or false accordingly)
- slack run_if: {{enveq "BUILD_STATUS_CHANGED" "true"}} (send message about build status has changed)

