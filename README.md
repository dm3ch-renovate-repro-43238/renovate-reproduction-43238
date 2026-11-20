# Reproduction for renovatebot/renovate#43238

Renovate cannot look up releases of a **public** GitHub repository when the repository's organization has an
[IP allow list](https://docs.github.com/en/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-allowed-ip-addresses-for-your-organization)
and Renovate runs from an IP address which is not on that list.

- Discussion: <https://github.com/renovatebot/renovate/discussions/43238>
- Earlier report: <https://github.com/renovatebot/renovate/discussions/17826>
- Same GitHub behavior in another tool: <https://github.com/github/gh-actions-lock/issues/274>
- Proposed fix: <https://github.com/renovatebot/renovate/pull/48249>

## Files

| File                                    | Purpose                                                                                                       |
| :-------------------------------------- | :------------------------------------------------------------------------------------------------------------ |
| `renovate.json`                         | Only enables the `github-actions` manager.                                                                    |
| `.github/workflows/example.yml`         | Uses the public action `aquasecurity/setup-trivy@v0.2.6`. The latest release is `v0.3.1`.                     |
| `.github/workflows/renovate-lookup.yml` | Checks access with an authenticated `curl`, then runs the released Renovate and the proposed fix (read-only). |

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

[Run](https://github.com/dm3ch-renovate-repro-43238/renovate-reproduction-43238/actions/runs/44755009624) on 3 runners, each running both Renovate builds one after the other, so with the same IP address:

| Renovate                                                           | `setup-trivy` update found | Lookup failures |
| :----------------------------------------------------------------- | :------------------------- | :-------------- |
| 44.257.0 (released)                                                | no                         | 2               |
| [proposed fix](https://github.com/renovatebot/renovate/pull/48249) | yes (`v0.3.1`)             | 0               |

## Why it can be hard to reproduce

- The refusal depends on the current allow list of the organization. `aquasecurity` has said its list is temporary.
- In a repository of a personal account, 36 runners with 36 different IP addresses and 216 authenticated requests did not hit the refusal.
  The `GITHUB_TOKEN` of such a repository is an installation token of a user, not of an organization.
