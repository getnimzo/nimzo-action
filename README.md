# Nimzo evaluation action

Evaluates the Terraform folders a pull request changes against the policies of their Nimzo stacks, and reports the verdict on the pull request.

For each folder of the pull request that holds a changed `*.tf`, `*.tf.json`, `*.tfvars` or `*.tfvars.json` file, the action finds the Nimzo stack imported from this repository at that folder, and evaluates the folder's Terraform files against the policy bundles assigned to the stack. A folder with no Nimzo stack is skipped. The action then reports:

- **The `Nimzo policies` check**, created on the pull request's head commit. Mark it as required to block merging (see [Required check](#required-check)).
- **One pull request comment**, updated on every new commit, with one block per changed stack. An evaluated stack opens with a GitHub alert (caution for blocking findings, warning for warnings only, tip when everything passed) that gives the commit checked, the policies and bundles evaluated or how many findings blocked on the previous commit, and a link to its audit in Nimzo. A table lists each finding, blocking first: its policy, linked to Nimzo, its severity, and its file and line. Under it, each finding unfolds into why its policy exists, how to fix it, the rule's message and the offending snippet; blocking findings start unfolded. A stack with no policy to apply, or that could not be evaluated, gets a note: where to assign bundles, or the reason and the link to its audit. Findings that disappeared since an earlier commit are listed as resolved, with the commit that fixed them. A folder with no Nimzo stack gets no block. When the comment would exceed GitHub's size limit, it drops the details, then findings, and points to the job summary, which holds the full verdict. A first run with nothing to report leaves no comment.
- An annotation on the offending line of each finding, on the `Nimzo policies` check: a failure for Critical and High, a warning for Medium, a notice for Low, with the link to the policy in Nimzo. Every finding gets one: the check takes them 50 per request, without the cap of 10 per level that annotations written from a step have. Errors, skipped folders and findings without a location are annotated on the job. The job summary holds the full verdict.
- The [`verdict` output](#verdict-output), for the workflow's next steps.

The action runs on `pull_request` events only. It fails a pull request that changes more files than GitHub lists (3000), since the folders of the unlisted ones would go unevaluated.

Nimzo records each evaluation on its stack with the pull request number and branch, and the commit evaluated: the workflow's `GITHUB_SHA`, which `actions/checkout` checks out by default on a pull request, the pull request merged into its base branch.

## Usage

```yaml
name: Nimzo

on:
  pull_request:

permissions:
  contents: read
  pull-requests: write
  checks: write
  packages: read

# A newer commit cancels the run of the previous one.
concurrency:
  group: nimzo-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: getnimzo/nimzo-action@main
        with:
          nimzo-token: ${{ secrets.NIMZO_TOKEN }}
```

| Input           | Default                     | Description                                                                                                                   |
| --------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `nimzo-url`     | `https://staging.nimzo.dev` | URL of Nimzo, without a path: the CLI calls the API under it, and the policy links in the comment and annotations point at it |
| `nimzo-token`   | required                    | Organization API key (`ak_…`) the Nimzo CLI sends to the governance API, see [Token](#token)                                  |
| `github-token`  | `${{ github.token }}`       | Lists the pull request's changed files and downloads the Nimzo CLI from GitHub Packages                                       |
| `nimzo-version` | `latest`                    | Tag of `ghcr.io/getnimzo/nimzo-cli` to run, or the digest (`sha256:…`) of its manifest                                        |

The permissions are needed to read the checked-out code (`contents`), list the changed files and write the comment (`pull-requests`), create the `Nimzo policies` check (`checks`), and download the CLI (`packages`).

### Verdict

The `Nimzo policies` check concludes, the worst case first:

| Conclusion | When                                                                                                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ❌ failure | A blocking finding, or a folder or policy that could not be evaluated, stated plainly                                                                                    |
| ⚠️ neutral | Only findings that do not block, or a folder whose Nimzo stack has no policy to apply: "No policies applied" (no bundle assigned, or no published policy in its bundles) |
| ✅ success | No finding, the changed folders have no Nimzo stack, or the pull request changes no Terraform folder                                                                     |

Which findings block follows the enforcement mode set in Nimzo for the folder's stack, or else for the organization: Critical and High findings by default, Medium ones too under "Errors and warnings", every finding under "All violations". The job fails with the check, so it can be required instead. The workflow's job itself stays green otherwise, since GitHub cannot show a job as neutral.

"Could not evaluate" names the cause plainly: Nimzo's error message, the reason of a task that failed (such as `POLICY` for a policy that crashed, or `TIMEOUT`), or that the `nimzo-token` is expired or invalid. An error Nimzo did not explain, such as an unreachable host or a proxy's page, shows only "Nimzo could not be reached or sent an unexpected response", and a task that failed for a reason other than `POLICY` or `INVALID_INPUT` shows only "Nimzo could not execute the evaluation" with its reason: re-run the job with debug logging to see the full error in the log. "No policies applied" is the one case that is not red: assign a bundle to the stack in Nimzo to check it.

When the action cannot run at all, such as when the CLI cannot be downloaded, it still tries to create a failed `Nimzo policies` check with the error.

A finding is the same from one commit to the next when its folder, policy, resource and message are the same: moving code to other lines, or rebasing the branch, keeps it. A finding of a folder that could not be evaluated on the latest commit is listed as not re-evaluated, rather than resolved.

### Required check

In the repository's branch protection rule or ruleset, require the status check named **`Nimzo policies`**, after the action has run once. GitHub treats `neutral` as passing, so warnings never block. Do not name a workflow job `Nimzo policies`: two checks with the same name make the result ambiguous.

### Verdict output

The `verdict` output is the verdict as JSON. Give the action step an `id`, and read the output in a later step through an environment variable:

```yaml
- uses: getnimzo/nimzo-action@main
  id: nimzo
  with:
    nimzo-token: ${{ secrets.NIMZO_TOKEN }}

- name: Show the verdict output
  if: always()
  env:
    VERDICT: ${{ steps.nimzo.outputs.verdict }}
  run: echo "$VERDICT" | jq .
```

Never write `${{ steps.nimzo.outputs.verdict }}` inside `run:`: the findings' messages hold quotes that break the shell command, and text from the pull request would run as shell code.

The output looks like this:

```json
{
  "conclusion": "failure",
  "title": "1 blocking finding",
  "folders": [
    {
      "path": "stacks/vpc",
      "outcome": "failed",
      "taskId": "01a15a0a-c94e-7675-8d10-4e0fbacd85f7",
      "message": null,
      "errorCode": null,
      "taskFailureReason": null
    }
  ],
  "findings": [
    {
      "key": "5c2d9a1e7b3f4a60",
      "folder": "stacks/vpc",
      "blocking": true,
      "policyId": "NIMZO_EC2_001",
      "title": "EC2 instance does not require IMDSv2",
      "description": "An aws_instance that does not set metadata_options.http_tokens to \"required\" allows IMDSv1…",
      "severity": "High",
      "remediation": "Add a metadata_options block to the aws_instance resource and set http_tokens = \"required\".",
      "resourceType": "aws_instance",
      "resourceId": "aws_instance.web",
      "message": "EC2 instance 'aws_instance.web' must require IMDSv2 for instance metadata access",
      "file": "stacks/vpc/main.tf",
      "line": 11
    }
  ],
  "staleFindings": [],
  "resolvedFindings": []
}
```

- A folder's `outcome` is `passed`, `warning` (only findings that do not block), `failed`, `skipped` (no Nimzo stack), `no_policies` (no policy applies to its stack) or `error`. For an `error`, `errorCode` is the error code of Nimzo's refusal (such as `GOVERNANCE_BUNDLE_NOT_FOUND` or `GENERIC_UNAUTHORIZED`) and `taskFailureReason` why its task failed (`ZIP`, `INVALID_INPUT`, `POLICY`, `TIMEOUT` or `INTERNAL`).
- A finding's `blocking` says whether it blocks under the enforcement mode Nimzo applied.
- `file` is relative to the repository root; `file` and `line` are `null` when Nimzo reports no location.
- `staleFindings` and `resolvedFindings` hold `{ key, folder, policyId, title, severity, resourceId, blocking, file, line, resolvedIn }`; `resolvedIn` is the commit a resolved finding was first missing from, and `null` for a stale one.

### How folders map to stacks

A folder is matched to the stack imported from `https://github.com/<owner>/<repo>` with that folder as its path. `.` is the repository root. The stack must have been imported with the repository's HTTPS URL.

## Current limits

- The action runs on Linux x64 runners only.
- Pull requests from forks and from Dependabot cannot be evaluated: GitHub gives them no secrets, so `nimzo-token` is empty, and a read-only token, so no comment nor check can be written. The job fails and says so.
- `nimzo-version` defaults to `latest`, which every change to the CLI republishes, so a new CLI reaches every workflow at once. To pin a CLI, set it to the manifest digest the `Publish package` run of `getnimzo/nimzo-cli` prints.

## Token

The CLI authenticates to the Nimzo API with an organization API key, which acts for the organization as a whole rather than for a user, so it keeps working when people leave. Creating one takes the Admin role:

1. In the Nimzo web app, with the organization active, open **Settings → API keys** and create a key. Give it an expiry that suits your rotation and copy it once: it starts with `ak_` and Nimzo never shows it again.
2. Store the key as a repository secret, such as `NIMZO_TOKEN`, and pass it as `nimzo-token`.

A revoked or expired key fails the next evaluation with "Could not evaluate" and the error code `GENERIC_UNAUTHORIZED`. Revoke a key in the same screen and create a new one to rotate it.
