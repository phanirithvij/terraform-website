---
layout: "cloud"
page_title: "Run States and Stages - Runs - Terraform Cloud and Terraform Enterprise"
description: |-
  Learn what happens during each stage of a run: pending, plan, cost estimation, policy check, apply, and complete.
---

# Run States and Stages


Each run passes through several stages of action (pending, plan, cost estimation, policy check, apply, and completion), and Terraform Cloud shows the progress through those stages as run states.

In the list of workspaces on Terraform Cloud's main page, each workspace shows the state of the run it's currently processing. (Or, if no run is in progress, the state of the most recent completed run.)

## 1. The Pending Stage

_States in this stage:_

- **Pending:** Terraform Cloud hasn't started action on a run yet. Terraform Cloud processes each workspace's runs in the order they were queued, and a run remains pending until every run before it has completed.

_Leaving this stage:_

- Proceeds automatically to the plan stage (**Planning** state) when it becomes the first run in the queue.
- Can skip to completion (**Discarded** state) if a user discards it before it starts.

## 2. The Plan Stage

_States in this stage:_

- **Planning:** Terraform Cloud is currently running `terraform plan`.
- **Needs Confirmation:** `terraform plan` has finished. Runs sometimes pause in this state, depending on the workspace and organization settings.

_Leaving this stage:_

- If the `terraform plan` command failed, the run skips to completion (**Plan Errored** state).
- If a user canceled the plan by pressing the "Cancel Run" button, the run skips to completion (**Canceled** state).
- If the plan succeeded and neither cost estimation nor Sentinel policy checks will be done, the run skips to completion (**Planned** state).
- If the plan succeeded and requires changes:
    - If cost estimation is enabled, the run proceeds automatically to the cost estimation stage.
    - If cost estimation is disabled and [Sentinel policies][] are enabled, the run proceeds automatically to the policy check stage.
    - If there are no Sentinel policies and the plan can be auto-applied, the run proceeds automatically to the apply stage. Plans can be auto-applied if the auto-apply setting is enabled on the workspace and the plan was queued by a new VCS commit or by a user with permission to apply runs. ([More about permissions.](/docs/cloud/users-teams-organizations/permissions.html))
    - If there are no Sentinel policies and the plan can't be auto-applied, the run pauses in the **Needs Confirmation** state until a user with permission to apply runs takes action. ([More about permissions.](/docs/cloud/users-teams-organizations/permissions.html)) The run proceeds to the apply stage if they approve the apply, or skips to completion (**Discarded** state) if they reject the apply.

[permissions-citation]: #intentionally-unused---keep-for-maintainers

## 3. The Cost Estimation Stage

This stage only occurs if cost estimation is enabled. After a successful `terraform plan`, Terraform Cloud uses plan data to estimate costs for each resource found in the plan.

_States in this stage:_

- **Cost Estimating:** Terraform Cloud is currently estimating the resources in the plan.
- **Cost Estimated:** The cost estimate completed.

_Leaving this stage:_

- If cost estimation succeeded or errors, the run moves to the next stage.
- If there are no policy checks or applies, the run skips to completion (**Planned (Finished)** state).

## 4. The Policy Check Stage

This stage only occurs if [Sentinel policies][] are enabled. After a successful `terraform plan`, Terraform Cloud checks whether the plan obeys policy to determine whether it can be applied.

[Sentinel policies]: ../sentinel/index.html

_States in this stage:_

- **Policy Check:** Terraform Cloud is currently checking the plan against the organization's policies.
- **Policy Override:** The policy check finished, but a soft-mandatory policy failed, so an apply cannot proceed without approval from a user with permission to manage policy overrides for the organization. ([More about permissions.](/docs/cloud/users-teams-organizations/permissions.html)) The run pauses in this state.
- **Policy Checked:** The policy check succeeded, and Sentinel will allow an apply to proceed. The run sometimes pauses in this state, depending on workspace settings.

[permissions-citation]: #intentionally-unused---keep-for-maintainers

_Leaving this stage:_

- If any hard-mandatory policies failed, the run skips to completion (**Plan Errored** state).
- If any soft-mandatory policies failed, the run pauses in the **Policy Override** state.
    - If a user with permission to manage policy overrides, overrides the failed policy, the run proceeds to the **Policy Checked** state. ([More about permissions.](/docs/cloud/users-teams-organizations/permissions.html))
    - If a user with permission to apply runs discards the run, the run skips to completion (**Discarded** state). ([More about permissions.](/docs/cloud/users-teams-organizations/permissions.html))
- If the run reaches the **Policy Checked** state (no mandatory policies failed, or soft-mandatory policies were overridden):
    - If the plan can be auto-applied, the run proceeds automatically to the apply stage. Plans can be auto-applied if the auto-apply setting is enabled on the workspace and the plan was queued by a new VCS commit or by a user with permission to apply runs. ([More about permissions.](/docs/cloud/users-teams-organizations/permissions.html))
    - If the plan can't be auto-applied, the run pauses in the **Policy Checked** state until a user with permission to apply runs takes action. ([More about permissions.](/docs/cloud/users-teams-organizations/permissions.html)) The run proceeds to the apply stage if they approve the apply, or skips to completion (**Discarded** state) if they reject the apply.

[permissions-citation]: #intentionally-unused---keep-for-maintainers

## 5. The Pre-Apply Stage

-> **Note:** As of September 2021, Run Tasks are available only as a beta feature, are subject to change, and not all customers will see this functionality in their Terraform Cloud organization.

This phase only occurs if there are [run tasks](../workspaces/run-tasks.html) enabled for the workspace, and it executes any configured run tasks after the Plan, optional Cost Estimation, and optional Policy Check phases have completed. During this phase, Terraform Cloud sends information about your run to the configured external system and waits for a pass or fail response to determine whether the plan can be applied. 

-> **Note:** The information sent to the configured external system includes the [JSON output](/docs/internals/json-format.html) of your plan.

_States in this stage:_

- **Running tasks:** Terraform Cloud is currently waiting for a response from the configured external system(s).
    - External systems must respond initially with a `200 OK` acknowledging the request is in progress. After that, they have 10 minutes to return a status of `passed` or `failed`, or the timeout will expire and the task will be assumed to be in the `failed` status.

_Leaving this stage:_

- If any mandatory tasks failed, the run skips to completion (**Plan Errored** state).
- If any advisory tasks failed, the run proceeds to the **Applying** state, with a visible warning regarding the failed task.
- If a combination of mandatory and advisory tasks are configured for a single run, Terraform will take the most restrictive action. This means that if there are two advisory tasks that succeed and one mandatory task that failed, the run will fail. If one mandatory task succeeds and two advisory tasks fail, the run will succeed with a warning.
- If a user canceled the apply by pressing the "Cancel Run" button, the run ends in the **Canceled** state.

## 6. The Apply Stage

_States in this stage:_

- **Applying:** Terraform Cloud is currently running `terraform apply`.

_Leaving this stage:_

After applying, the run proceeds automatically to completion.

- If the apply succeeded, the run ends in the **Applied** state.
- If the apply failed, the run ends in the **Apply Errored** state.
- If a user canceled the apply by pressing the "Cancel Run" button, the run ends in the **Canceled** state.

## 7. Completion

A run is considered completed if it finishes applying, if any part of the run fails, if there's nothing to do, or if a user chooses not to continue. Once a run is completed, the next run in the queue can enter the plan stage.

_States in this stage:_

- **Applied:** The run was successfully applied.
- **Planned:** `terraform plan`'s output already matches the current infrastructure state, so `terraform apply` doesn't need to do anything.
- **Apply Errored:** The `terraform apply` command failed, possibly due to a missing or misconfigured provider or an illegal operation on a provider.
- **Plan Errored:** The `terraform plan` command failed (usually requiring fixes to variables or code), or a hard-mandatory Sentinel policy failed. The run cannot be applied.
- **Discarded:** A user chose not to continue this run.
- **Canceled:** A user interrupted the `terraform plan` or `terraform apply` command with the "Cancel Run" button.
