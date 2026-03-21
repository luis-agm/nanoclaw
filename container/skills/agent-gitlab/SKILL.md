---
name: agent-gitlab
description: Use the GitLab CLI (glab) to manage issues, merge requests, pipelines, and code review. Invoke whenever the user asks about GitLab issues, MRs, CI/CD, or repository operations.
allowed-tools: Bash(glab:*)
---

# GitLab CLI Agent Skill

You have access to `glab`, the official GitLab CLI. Use it to help with GitLab operations.

## Authentication

`glab` authenticates automatically via the `GITLAB_TOKEN` environment variable.
The default host is `gitlab.com`; for self-hosted instances `GITLAB_HOST` is set.

## Default repository

If `GL_REPO` is set, pass it as `--repo $GL_REPO` on commands that need a project target
and you are not inside a cloned repo directory. If you are inside a cloned repo directory
(`git remote` points to GitLab), `glab` will detect it automatically.

## Read operations (always safe)

```bash
# Issues
glab issue list [--repo $GL_REPO] [--state opened|closed|all] [--label bug] [--assignee @me]
glab issue view <id> [--repo $GL_REPO]

# Merge Requests
glab mr list [--repo $GL_REPO] [--state opened|merged|closed] [--author @me]
glab mr view <id> [--repo $GL_REPO]
glab mr diff <id> [--repo $GL_REPO]
glab mr checks <id> [--repo $GL_REPO]

# Pipelines & CI
glab ci list [--repo $GL_REPO]
glab ci view <id> [--repo $GL_REPO]
glab ci trace <job-id> [--repo $GL_REPO]

# Repository
glab repo view [--repo $GL_REPO]
glab label list [--repo $GL_REPO]
glab release list [--repo $GL_REPO]
glab release view <tag> [--repo $GL_REPO]

# Output as JSON for structured processing
glab issue list --output json | jq '.[] | {id: .iid, title: .title}'
```

## Safe write operations

```bash
# Comments
glab issue note <id> --message "comment text" [--repo $GL_REPO]
glab mr note <id> --message "comment text" [--repo $GL_REPO]

# Labels and assignees
glab issue update <id> --label "bug,needs-triage" [--repo $GL_REPO]
glab issue update <id> --assignee username [--repo $GL_REPO]
glab mr update <id> --label "ready" [--repo $GL_REPO]
glab mr update <id> --reviewer username [--repo $GL_REPO]

# MR reviews
glab mr approve <id> [--repo $GL_REPO]

# Creating issues
glab issue create --title "Bug: ..." --description "Details..." [--repo $GL_REPO]
```

## Prohibited actions

**Never perform these without explicit user confirmation:**

- `glab mr merge` — merges a merge request
- `glab issue close` — closes an issue
- `glab mr close` — closes a merge request
- `glab release delete` — deletes a release
- `glab repo delete` — deletes a repository
- `glab variable delete` — removes CI/CD variables
- Any `git push --force` or branch deletion

## Rate limits

GitLab allows 2,000 requests per 10 minutes with PAT authentication. Prefer paginated
queries and narrow filters over fetching all records.

## Tips

- Add `--output json` to most commands and pipe through `jq` for filtering
- Use `glab issue list --search "keyword"` to find issues by text
- `glab mr list --author @me` filters to your own MRs
- Pipeline logs: `glab ci trace <job-id>` streams the log for a specific job
