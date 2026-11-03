# Rulesets

Org-level rulesets for Autonommi, stored as the exact request body for the GitHub REST API
(`POST orgs/Autonommi/rulesets`). The same files can be loaded with "Import a ruleset" in the org settings.

Part of Autonommi/etterly-mobile#22.

## Files

| File | Targets | Purpose |
|---|---|---|
| `protect-main.json` | `etterly-backend` (`etterly-mobile` after testing), branch `refs/heads/main` | Rule GH-09 in `standards/RULES.md` |

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

## Requirements

- An org owner of Autonommi.
- A token with the `admin:org` scope: `gh auth refresh -h github.com -s admin:org`.
- Org rulesets are available on the Team plan since 2025-06-16
  ([GitHub changelog](https://github.blog/changelog/2025-06-16-organization-rulesets-now-available-for-github-team-plans/)).

## Create

```sh
gh api -X POST orgs/Autonommi/rulesets --input rulesets/protect-main.json
```

Record the returned `id` here:

| Ruleset | Org ruleset id | Created |
|---|---|---|
| Protect main | not created yet | |

## Update

Edit the JSON file, commit it, then:

```sh
gh api -X PUT orgs/Autonommi/rulesets/<id> --input rulesets/protect-main.json
```

To add `etterly-mobile`, add it to `conditions.repository_name.include` and run the update.

## Repo merge settings (rebase only)

Run for each targeted repo. The ruleset already blocks squash and merge commits; this removes the buttons.

```sh
gh api -X PATCH repos/Autonommi/<repo> -F allow_rebase_merge=true -F allow_squash_merge=false -F allow_merge_commit=false
```

| Repo | Before (read 2026-11-02) | After |
|---|---|---|
| etterly-backend | rebase, squash, merge commit | rebase only |
| etterly-mobile | rebase, squash, merge commit | rebase only |

## Check and test

List every rule that applies to `main`, with the ruleset each comes from:

```sh
gh api repos/Autonommi/<repo>/rules/branches/main --jq '.[] | {type, ruleset_id, parameters}'
```

While a repo-level ruleset is still active, both appear. GitHub applies the most restrictive version of each rule.
**Not verified:** how `allowed_merge_methods` combines across two rulesets. Test 6 below checks it directly.

Tests (U1 in the epic, plus the review rules):

**Safety:** tests 1 to 3 are run by a person, not a bot. Before running them:

1. Confirm the check output lists `non_fast_forward` and `deletion`.
2. Note `main`'s current commit SHA (`gh api repos/Autonommi/<repo>/commits/main --jq .sha`), so it can be restored if anything goes wrong.

| # | Test | Expected |
|---|---|---|
| 1 | Direct push to `main` | Rejected |
| 2 | Force push to `main` | Rejected |
| 3 | Deletion rule present (no live delete) | Check output lists `deletion` |
| 4 | Merge a PR with 0 approvals, or with an unresolved conversation | Blocked |
| 5 | Push to an approved PR | Approval dismissed |
| 6 | Open the merge menu on a PR | Only "Rebase and merge" offered |

## Retire the repo-level rulesets

Do this per repo, only after the tests pass with the org ruleset active.

| Repo | Repo ruleset | id |
|---|---|---|
| etterly-backend | Protect main | 25512782 |
| etterly-mobile | Protect Main | 24211782 |

1. Save a copy outside this repo:
   `gh api repos/Autonommi/<repo>/rulesets/<id> > <backup-path>/<repo>-ruleset-<id>.json`
2. Disable it (reversible):
   `gh api -X PUT repos/Autonommi/<repo>/rulesets/<id> -f enforcement=disabled`
3. Re-run the check above: only rules from the org ruleset should remain. Re-run the tests.
4. After a short wait with no problems, delete it:
   `gh api -X DELETE repos/Autonommi/<repo>/rulesets/<id>`

Retiring mobile's ruleset (24211782) drops `require_extra_approval_for_unattributed_changes: true`.
This is intended: it is not in the ticket or GH-09.

## Emergencies

There are no bypass actors. In an emergency an org owner temporarily sets the org ruleset's
`enforcement` to `disabled`, then sets it back to `active`. Tell the team in Teams when the ruleset is
disabled and again when it is back on.

```sh
gh api -X PUT orgs/Autonommi/rulesets/<id> -f enforcement=disabled
gh api -X PUT orgs/Autonommi/rulesets/<id> -f enforcement=active
```
