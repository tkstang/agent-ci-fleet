# agent-ci-fleet

Public maintenance caller for [agent-ci](https://github.com/tkstang/agent-ci).
It runs the OAT maintenance presets (`docs-update`, `instructions-update`,
`tools-update`) across public repositories on GitHub-hosted runners and opens
a pull request on each target with the result.

This repository is the `tkstang` public fleet. Other owners can copy the caller
workflow and `targets.yaml` to run their own fleet.

Private repositories belong in the private fleet, not here. The caller passes
`public_only: true`, so a run refuses any target GitHub does not report as
public before a job runs its code. This repository's Actions logs and
artifacts are public.

## Files

- `.github/workflows/agent-ci-fleet.yml`: the caller, copied from agent-ci's
  `consumer-workflows/agent-ci-fleet.yml` and pinned to `v0.2.0`. Manual
  dispatch only.
- `targets.yaml`: the target list. It starts with one smoke target, this
  repository (`fleet-self`). See [Targets](#targets) for its fields.

## Targets

agent-ci validates `targets.yaml` on every run: the schema, unique target ids
and repositories, `allowedOwners`, and that each model resolves under the
routing policy in CI. This file sets these fields:

| Field                            | Value here                          | Purpose                                            |
| -------------------------------- | ----------------------------------- | -------------------------------------------------- |
| `version`                        | `1`                                 | Target-file schema version.                        |
| `allowedOwners`                  | `[tkstang]`                         | GitHub owners a target may belong to.              |
| `defaults.base`                  | `main`                              | Base branch for target pull requests.              |
| `defaults.model`                 | `claude-haiku-5-5`                  | Model used when a target does not set one.         |
| `defaults.mode`                  | `edit`                              | Mode applied to each target run.                   |
| `defaults.inputs.timeoutMinutes` | `30`                                | Per-run timeout in minutes.                        |
| `defaults.inputs.maxAttempts`    | `2`                                 | Per-run attempt limit.                             |
| `targets[].id`                   | set per target                      | Unique target id, used by the `target` input.      |
| `targets[].repository`           | set per target                      | Target repository as `owner/name`; must be public. |
| `targets[].model`                | `fleet-self`: `deepseek-v4.1-flash` | Optional per-target model override.                |

These are this file's values, not built-in defaults: a target with no base
branch, here or in `defaults`, fails validation.

To add a target, add an entry under `targets` with a unique `id`, set
`repository` to an `owner/name` under `allowedOwners`, and set `model` only when
it differs from `defaults.model`. Add the model credential in
[Setup](#setup), then cover the target with `AGENT_CI_TARGET_TOKEN`.

To append a JSON target array after `targets.yaml`, set the
`AGENT_CI_TARGETS_JSON` repository variable (Settings, Secrets and variables,
Actions, Variables). Leave it unset to use only the file. Variables are visible
configuration.

## Setup

1. Secrets (repository Settings, Secrets and variables, Actions):
   - `CLAUDE_CODE_OAUTH_TOKEN`, from `claude setup-token`, for the default
     `claude-haiku-5-5` model. The value starts with `sk-ant-oat01-`; store
     it on one line with no surrounding spaces. `tools-update` needs no model
     credential.
   - `OPENCODE_API_KEY`, the OpenCode Go key, for targets that run an
     OpenCode Go model (`fleet-self` runs `deepseek-v4.1-flash`).
   - `AGENT_CI_TARGET_TOKEN`, a fine-grained token with contents and
     pull-requests write on every target other than this repository. Only the
     resolve and publish jobs receive it.
2. To publish to `fleet-self`, enable "Allow GitHub Actions to create and
   approve pull requests" (Settings, Actions, General). Other targets do not
   need it.
3. Add public targets to `targets.yaml` and cover each with
   `AGENT_CI_TARGET_TOKEN`.

Set the credential for every model route your targets use. Target validation
does not check credentials, so a missing credential fails at run time.

Do not attach a self-hosted runner to this repository: any pull request,
including one from a fork, can add a workflow that runs on it. To use a
proxy pool, reach it at its https endpoint instead; the commented pool lines
in the workflow show how. See agent-ci's
[Choosing how models run](https://github.com/tkstang/agent-ci/blob/main/docs/running-models.md).

## Running

Dispatch **[agent-ci] Run fleet** from the Actions tab.

| Input    | Default       | Effect                                                       |
| -------- | ------------- | ------------------------------------------------------------ |
| `preset` | `docs-update` | Maintenance preset.                                          |
| `target` | `all`         | A target id or `owner/name`, or every target for the preset. |
| `reset`  | `false`       | Replace one target's static branch content, keeping history. |

| Preset                | What it does                                                                                | Model |
| --------------------- | ------------------------------------------------------------------------------------------- | ----- |
| `docs-update`         | Runs the OAT docs analysis, then applies its recommendations to the target's documentation. | yes   |
| `instructions-update` | Updates agent instruction files (`AGENTS.md` and similar) the same way.                     | yes   |
| `tools-update`        | Runs `oat tools update` for OAT tools installed in the target repository itself.            | no    |

A target that installs OAT only at user scope has nothing for `tools-update`
to change.

A run that changes files commits them to the preset's static branch on the
target (for example `agent-ci/oat-docs-update`) and opens one pull request;
later runs update it while it is open. A run with nothing to change succeeds
without a pull request. Each target's result is uploaded as the run artifact
`agent-ci-<target id>`: `run-result.json` holds its status (`changed`,
`no-op` or `failed`), summary and errors.

Start with one manual `docs-update` run on `fleet-self` before adding
schedules (examples are commented in the workflow).

## Troubleshooting

- **Target validation fails.** agent-ci validates the schema, unique target
  ids and repositories, `allowedOwners`, and model resolution on every run.
  Fix the entry in `targets.yaml` before dispatching again.
- **A run refuses a target with `E_TARGET_VISIBILITY`.** GitHub does not report
  the target repository as public. Move private repositories to the private
  fleet.
- **A run fails on a missing model credential.** Target validation does not
  check credentials. Add the secret for that model route in
  [Setup](#setup); `tools-update` needs none.
- **A Claude run fails with `E_HARNESS_AUTH` (401 Invalid bearer token).**
  `CLAUDE_CODE_OAUTH_TOKEN` is not a valid setup token: it must be the whole
  `sk-ant-oat01-` value from `claude setup-token`, with no leading or
  trailing space. Store it again.
- **A Claude run fails with a usage-limit message.** The subscription behind
  the token has hit its limit. Wait for the reset, or point the target at
  another model.
- **No pull request appears.** Check the target's `run-result.json`: `no-op`
  means there was nothing to change. For `fleet-self`, also enable "Allow
  GitHub Actions to create and approve pull requests" under Settings,
  Actions, General.
- **A run refuses to update the static branch (`E_PUBLISH`, "agent-ci did
  not publish").** Someone else's commit is the branch head, for example a
  manual fix pushed to the pull request. Merge or close that pull request
  and delete the branch, or dispatch again for that one target with
  `reset: true`, which replaces the branch content and keeps its history.
