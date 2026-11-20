# Reproduction for renovatebot/renovate#43238

Renovate cannot look up releases of a **public** GitHub repository when the repository's organization has an
[IP allow list](https://docs.github.com/en/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-allowed-ip-addresses-for-your-organization)
and Renovate runs from an IP address which is not on that list.

- Discussion: <https://github.com/renovatebot/renovate/discussions/43238>
- Earlier report: <https://github.com/renovatebot/renovate/discussions/17826>
- Same GitHub behavior in another tool: <https://github.com/github/gh-actions-lock/issues/274>

## Status

**Not reproduced on 2026-11-20.**
Two runs on GitHub-hosted runners (Renovate 44.257.0, runner IP 52.242.243.207) did not hit the refusal: `aquasecurity/setup-trivy`, `aquasecurity/trivy-action`, `aquasecurity/trivy` and `bridgecrewio/checkov-action` were all readable with the `GITHUB_TOKEN`.
The refusal depends on the IP address of the runner and on the current allow list of the organization, so it is intermittent or no longer applies to these organizations.
See [the first run](https://github.com/dm3ch/renovate-reproduction-43238/actions/runs/44745583620).

The report in <https://github.com/github/gh-actions-lock/issues/274> shows the refusal for `aquasecurity/setup-trivy` on a GitHub-hosted runner with a `GITHUB_TOKEN`.

## Files

| File                                    | Purpose                                                                                                 |
| :-------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| `renovate.json`                         | Only enables the `github-actions` manager.                                                              |
| `.github/workflows/example.yml`         | Uses the public action `aquasecurity/setup-trivy@v0.2.6`. The latest release is `v0.3.1`.               |
| `.github/workflows/renovate-lookup.yml` | Runs Renovate (read-only, `RENOVATE_PLATFORM=local`) on a GitHub-hosted runner with the `GITHUB_TOKEN`. |

## Steps to reproduce

1. Go to **Actions**, select **Renovate lookup**, and choose **Run workflow**.
2. Open the job summary, or the `Run Renovate without changing anything` step.

The workflow runs on a GitHub-hosted runner, because its IP address is not on the allow list of the `aquasecurity` organization.
It cannot be reproduced from an IP address which is on the allow list, so running Renovate on a machine which is allowed shows no problem.

## Current behavior

When the refusal happens, GitHub refuses the token, even though `aquasecurity/setup-trivy` is public and can be read without a token.
Renovate does not find any release, and logs the refusal:

```text
Although you appear to have the correct authorization credentials, the `aquasecurity` organization has an IP allow list enabled, and <runner IP> is not permitted to access this resource.
```

No update is proposed for `aquasecurity/setup-trivy`.
The lookup for `actions/checkout` in the same run is the control: it works, because that organization does not refuse the token.

## Expected behavior

Renovate finds the releases of the public repository and proposes the update from `v0.2.6` to `v0.3.1`, as it does when run from an IP address which is allowed.

The same request without a token succeeds:

```sh
curl -s https://api.github.com/repos/aquasecurity/setup-trivy/tags?per_page=1
```
