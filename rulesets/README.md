# Rulesets

Org-level rulesets for Autonommi, stored as the exact request body for the GitHub REST API
(`POST orgs/Autonommi/rulesets`). The same files can be loaded with "Import a ruleset" in the org settings.

Part of Autonommi/etterly-mobile#22.

## Rulesets

| File | Org ruleset id | Targets | Purpose |
|---|---|---|---|
| `protect-main.json` | 26630015 | `etterly-backend`, `etterly-frontend`, `.github`, `standards`, branch `refs/heads/main` | Rule GH-09 in `standards/RULES.md` |
| `require-checks-backend.json` | not created yet | `etterly-backend`, branch `refs/heads/main` | Required checks, up to date with main |
| `require-checks-frontend.json` | not created yet | `etterly-frontend`, branch `refs/heads/main` | Required checks, up to date with main |

## What `protect-main.json` enforces

| # | GH-09 item | Rule in the file |
|---|---|---|
| 1 | Block deletion | `deletion` |
| 2 | Block force push | `non_fast_forward` |
| 3 | PR with at least 1 approval | `pull_request.required_approving_review_count: 1` |
| 4 | Dismiss stale reviews on push | `dismiss_stale_reviews_on_push: true` |
| 5 | All conversations resolved | `required_review_thread_resolution: true` |
| 6 | No bypass actors | `bypass_actors: []` |
| 7 | Rebase merging only (not in GH-09) | `allowed_merge_methods: ["rebase"]` |

Required checks are in the `require-checks-*` rulesets below. No merge queue: it needs GitHub Enterprise
for private repos.

`"protected": true` on `repository_name` means renaming a targeted repo does not drop it from the ruleset.
`require_extra_approval_for_unattributed_changes` is set to `false` explicitly: GitHub sets it to `true` when it is omitted.

## What the `require-checks-*` rulesets enforce

Separate from "Protect main" because `.github` and `standards` don't run these checks; required there,
they would block every PR. One ruleset per app repo because the check lists differ.

| Check | Workflow | Backend | Frontend |
|---|---|---|---|
| `exact versions` | `pins.yml` | Required | Required |
| `allowed licences` | `licences.yml` | Required | Required |
| `approved packages` | `approvals.yml` | Required | Required |
| `check` | `mobile.yml` | n/a | Required |

- Each check pins `integration_id: 15368` (GitHub Actions), so a status with the same name from another
  app or a token doesn't count.
- `strict_required_status_checks_policy: true`: a PR must be up to date with main. After each merge, every
  open PR needs "Update branch" and a new run of its checks. This also applies to hotfix PRs: auto-merge
  waits until the branch is updated.
- `approved packages` is skipped on hotfix PRs (see the app repo READMEs). GitHub counts a skipped job as
  passing, so the hotfix lane still works; the check runs on main after the merge.
- The backend has no build or test job on PRs yet. When it gets one, add its job name to
  `require-checks-backend.json`.

Rules for the workflows behind required checks:

- **No `paths` filter.** A required check whose workflow doesn't trigger stays "Expected" and the PR can't merge.
- **Don't rename a job** without updating its ruleset in the same change; the check is matched by job name.

Known gap: a check result belongs to a commit, not to a PR. A `hotfix/` branch with the `hotfix` label,
pushed at the same commit as another PR's head, records a skipped (passing) `approved packages` on that
commit, and the other PR then shows it as passing. The run on main after the merge still catches an
unapproved package. The PR reviewer should check that the `hotfix` label is only on real hotfixes.

## Requirements

- An org owner of Autonommi.
- A token with the `admin:org` scope: `gh auth refresh -h github.com -s admin:org`.

## Update

Edit the JSON file, commit it, then:

```sh
gh api -X PUT orgs/Autonommi/rulesets/26630015 --input rulesets/protect-main.json
```

For the other rulesets, use the id from the table above. To create a ruleset from a new file, then add
its id to the table:

```sh
gh api -X POST orgs/Autonommi/rulesets --input rulesets/<file>.json --jq .id
```

To add a repo, add it to `conditions.repository_name.include`, run the update, and set the repo to
rebase-only merging in its settings.

Check which rules apply to a repo's `main`, and which ruleset each comes from:

```sh
gh api repos/Autonommi/<repo>/rules/branches/main --jq '.[] | {type, ruleset_id, parameters}'
```

## Old repo-level rulesets

Replaced by the org ruleset and disabled on 2026-11-05. Deleted on 2026-11-09, after PRs had merged in
both repos under the org ruleset alone.

| Repo | Repo ruleset | id | Status |
|---|---|---|---|
| etterly-backend | Protect main | 25512782 | Deleted |
| etterly-mobile | Protect Main | 24211782 | Deleted |

## Emergencies

There are no bypass actors. In an emergency an org owner temporarily sets the org ruleset's
`enforcement` to `disabled`, then sets it back to `active`. Tell the team in Teams when the ruleset is
disabled and again when it is back on.

```sh
gh api -X PUT orgs/Autonommi/rulesets/26630015 -f enforcement=disabled
gh api -X PUT orgs/Autonommi/rulesets/26630015 -f enforcement=active
```

If a required check is the blocker (for example, GitHub Actions is down), do the same with that repo's
`require-checks-*` ruleset id instead.
