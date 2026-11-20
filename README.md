# Reproduction for renovatebot/renovate#43238

Renovate cannot look up releases of a **public** GitHub repository when the repository's organization has an
[IP allow list](https://docs.github.com/en/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-allowed-ip-addresses-for-your-organization)
and Renovate runs from an IP address which is not on that list.

- Discussion: <https://github.com/renovatebot/renovate/discussions/43238>
- Earlier report: <https://github.com/renovatebot/renovate/discussions/17826>
- Same GitHub behavior in another tool: <https://github.com/github/gh-actions-lock/issues/274>
- Proposed fix: <https://github.com/renovatebot/renovate/pull/48249>

## Files

| File                                    | Purpose                                                                                                                     |
| :-------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------- |
| `renovate.json`                         | Only enables the `github-actions` manager.                                                                                  |
| `.github/workflows/example.yml`         | Uses the public action `aquasecurity/setup-trivy@v0.2.6`. The latest release is `v0.3.1`.                                   |
| `.github/workflows/renovate-lookup.yml` | Checks access with an authenticated `curl`, then runs the released Renovate and two builds of the proposed fix (read-only). |

## Steps to reproduce

1. Go to **Actions**, select **Renovate lookup**, and choose **Run workflow**.
2. Open the job summary, or the steps of the job.

The workflow runs on GitHub-hosted runners with the `GITHUB_TOKEN` of this repository.
**This repository must belong to an organization**: the refusal did not happen for a repository of a personal account.

## Current behavior

GitHub refuses the token, even though `aquasecurity/setup-trivy` is public and can be read without a token.
The `curl` step shows it, for REST (HTTP `403`) and for GraphQL (HTTP `200` with an error in the body):

```text
| REST `aquasecurity/setup-trivy`    | 403 | YES |
| GraphQL `aquasecurity/setup-trivy` | 200 | YES |
```

Renovate 44.257.0 then finds no release, and no update is proposed for `aquasecurity/setup-trivy`:

```text
DEBUG: Failed to look up github-tags package aquasecurity/setup-trivy: no-result
```

The update of `actions/checkout` in the same run is the control: that organization does not refuse the token.

## Expected behavior

Renovate finds the releases of the public repository and proposes the update from `v0.2.6` to `v0.3.1`.
The same requests without a token succeed, but only for REST: the GraphQL API of GitHub can't be used without a token.

```sh
curl -s "https://api.github.com/repos/aquasecurity/setup-trivy/tags?per_page=1"
```

## Result

[Run](https://github.com/dm3ch-renovate-repro-43238/renovate-reproduction-43238/actions/runs/44758228894) on 3 runners, each running all builds one after the other, so with the same IP address.
Both actions should get an update, `v0.3.1` and `v0.36.0`:

| Renovate                                                                                 | Runner 1      | Runner 2    | Runner 3    |
| :--------------------------------------------------------------------------------------- | :------------ | :---------- | :---------- |
| 44.257.0 (released): updates found / requests to `api.github.com`                        | 0 of 2 / 12   | 0 of 2 / 12 | 0 of 2 / 12 |
| [fix](https://github.com/renovatebot/renovate/pull/48249), REST after GraphQL is refused | 0 of 2 / 4 \* | 2 of 2 / 8  | 2 of 2 / 8  |
| fix, also skips GraphQL for an organization which refused the token                      | 2 of 2 / 6    | 2 of 2 / 6  | 2 of 2 / 6  |

\* The requests without a token were rate limited: GitHub allows 60 per hour for an IP address, and runners share addresses.
Renovate then reports the original error, like the released version.

## Why it can be hard to reproduce

- The refusal depends on the current allow list of the organization. `aquasecurity` has said its list is temporary.
- In a repository of a personal account, 36 runners with 36 different IP addresses and 216 authenticated requests did not hit the refusal.
  The `GITHUB_TOKEN` of such a repository is an installation token of a user, not of an organization.
