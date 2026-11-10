# agent-ci-fleet

Public maintenance caller for [agent-ci](https://github.com/tkstang/agent-ci).
It runs the OAT maintenance presets (`docs-update`, `instructions-update`,
`tools-update`) across public repositories on GitHub-hosted runners and opens
a pull request on each target with the result.

Private repositories belong in the private fleet, not here. The caller passes
`public_only: true`, so a run refuses any target GitHub does not report as
public before a job runs its code. This repository's Actions logs and
artifacts are public.

## Files

- `.github/workflows/agent-ci-fleet.yml`: the caller, copied from agent-ci's
  `consumer-workflows/agent-ci-fleet.yml` and pinned to `v0.2.0`. Manual
  dispatch only.
- `targets.yaml`: the target list. It starts with one smoke target, this
  repository (`fleet-self`).

## Setup

1. Secrets (repository Settings, Secrets and variables, Actions):
   - `CLAUDE_CODE_OAUTH_TOKEN`, from `claude setup-token`, for the default
     `claude-haiku-5-5` model. `tools-update` needs no model credential.
   - `AGENT_CI_TARGET_TOKEN`, a fine-grained token with contents and
     pull-requests write on every target other than this repository. Only the
     resolve and publish jobs receive it.
2. To publish to `fleet-self`, enable "Allow GitHub Actions to create and
   approve pull requests" (Settings, Actions, General). Other targets do not
   need it.
3. Add public targets to `targets.yaml` and cover each with
   `AGENT_CI_TARGET_TOKEN`.

Do not attach a self-hosted runner to this repository: any pull request,
including one from a fork, can add a workflow that runs on it. To use a
proxy pool, reach it at its https endpoint instead; the commented pool lines
in the workflow show how. See agent-ci's
[Choosing how models run](https://github.com/tkstang/agent-ci/blob/main/docs/running-models.md).

## Running

Dispatch **[agent-ci] Run fleet** from the Actions tab.

| Input    | Default       | Effect                                                         |
| -------- | ------------- | -------------------------------------------------------------- |
| `preset` | `docs-update` | Maintenance preset.                                            |
| `target` | `all`         | A target id or `owner/name`, or every target for the preset.   |
| `reset`  | `false`       | Replace one target's static branch content, keeping history.   |

Start with `tools-update` on `fleet-self` (no model), then one model-backed
run, before adding schedules (examples are commented in the workflow).
