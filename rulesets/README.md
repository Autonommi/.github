# Rulesets

Org-level rulesets for Autonommi, stored as the exact request body for the GitHub REST API
(`POST orgs/Autonommi/rulesets`). The same files can be loaded with "Import a ruleset" in the org settings.

Part of Autonommi/etterly-mobile#22.

## Rulesets

| File | Org ruleset id | Targets | Purpose |
|---|---|---|---|
| `protect-main.json` | 26630015 | `etterly-backend`, `etterly-mobile`, `.github`, `standards`, branch `refs/heads/main` | Rule GH-09 in `standards/RULES.md` |

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

Not included yet: required status checks and merge queue.

`"protected": true` on `repository_name` means renaming a targeted repo does not drop it from the ruleset.
`require_extra_approval_for_unattributed_changes` is set to `false` explicitly: GitHub sets it to `true` when it is omitted.

## Requirements

- An org owner of Autonommi.
- A token with the `admin:org` scope: `gh auth refresh -h github.com -s admin:org`.

## Update

Edit the JSON file, commit it, then:

```sh
gh api -X PUT orgs/Autonommi/rulesets/26630015 --input rulesets/protect-main.json
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
