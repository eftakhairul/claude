---
name: github-pr-create
description: >
  Use this skill whenever the user wants to create a Pull Request (PR) on GitHub or GitLab
  with a smart, auto-formatted title. Triggers on phrases like "create a PR", "open a pull request",
  "make a PR", "submit a PR", "open a MR", "create merge request", or any mention of creating a
  pull/merge request on GitHub or GitLab. Also trigger when the user provides a branch name and
  asks for a PR title suggestion. This skill automatically derives a semantic prefix from the
  branch name (feature, fix, chore, docs, etc.) and generates a concise, human-readable title
  in the imperative mood and lowercase based on the PR's content or description.
---

# GitHub / GitLab PR Creator Skill

Create a well-formatted Pull Request with an auto-derived semantic title based on the branch name
and PR content. Titles are always **lowercase** and use **imperative mood** verb phrases.

---

## Title Prefix Rules

Inspect the **branch name** (the source/head branch) and apply the **first matching rule**:

| Branch prefix (case-insensitive) | PR title prefix |
|---|---|
| `feat/` or `feature/` | `[feature]:` |
| `fix/` or `bugfix/` or `hotfix/` | `[fix]:` |
| `chore/` | `[chore]:` |
| `docs/` or `doc/` | `[docs]:` |
| `refactor/` or `refact/` | `[refactor]:` |
| `test/` or `tests/` | `[test]:` |
| `style/` | `[style]:` |
| `perf/` | `[perf]:` |
| `ci/` | `[ci]:` |
| *(no match / unknown)* | `[feature]:` *(default)* |

---

## Title Message Generation

After the prefix, generate a **short, lowercase, imperative mood summary** of what the PR does:

1. **If the user provides a PR description / summary** → distill it into ≤ 8 words, fully lowercase, imperative mood verb, no trailing period.
2. **If only the branch name is given** → convert the slug after the prefix into a readable phrase.
   - Strip the prefix segment (e.g. `feat/`) and convert hyphens/underscores to spaces.
   - Write fully lowercase.
   - Use the imperative mood (e.g. `add fix` instead of `added fix`, `create user` instead of `creates user`).
   - Example: `feat/create-user` → `create user`
3. **Always use the imperative mood** that describe the change, e.g. "add user creation endpoint", "fix null pointer on login".
4. **Never capitalize** any word in the summary — keep everything lowercase.
5. **Addition rules**
    - Limit the subject line to 50 characters or fewer.
    - Do not end the subject line with a period.

### Full title format
```
[prefix]: <imperative mood lowercase summary>
```
Examples:
- `[feature]: create user`
- `[fix]: resolve null pointer on login page`
- `[chore]: update dependencies`
- `[refactor]: refactor production code`
- `[docs]: add api authentication guide`
- `[test]: add unit tests for module b`
---

## Step-by-Step Workflow

### Step 1 – Gather Information

Ask the user for any missing details (collect all at once if possible):

| Info | Required? | Notes |
|---|---|---|
| **Platform** | Yes | GitHub or GitLab |
| **Repo** | Yes | `owner/repo` format |
| **Branch name** (head/source) | Yes | Used for prefix detection |
| **Base branch** (target) | No | Default: `main` or `master` |
| **PR description / summary** | No | Used to craft the title message |
| **Draft PR?** | No | Default: No |
| **Reviewers** | No | Comma-separated usernames |
| **Labels** | No | Comma-separated label names |

If the user already provided these in their message, do **not** ask again — extract them directly.

### Step 2 – Derive the Title

Apply the prefix rules above to the branch name, then generate the message in the imperative mood and
lowercase from the description or branch slug. Show the user the proposed title before creating:

> **Proposed PR title:** `[feature]: create user`
> Proceed? (or suggest a different title)

If the user confirms or doesn't object, continue. If they suggest a change, apply it.

### Step 3 – Create the PR

#### GitHub (via `gh` CLI)

```bash
gh pr create \
  --repo <owner/repo> \
  --head <branch> \
  --base <base-branch> \
  --title "<GENERATED_TITLE>" \
  --body "<description>" \
  [--draft] \
  [--reviewer <user1,user2>] \
  [--label <label1,label2>]
```

#### GitLab (via `glab` CLI)

```bash
glab mr create \
  --repo <owner/repo> \
  --source-branch <branch> \
  --target-branch <base-branch> \
  --title "<GENERATED_TITLE>" \
  --description "<description>" \
  [--draft] \
  [--reviewer <user1,user2>] \
  [--label <label1,label2>]
```

#### GitHub API (if CLI not available)

```bash
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<owner>/<repo>/pulls \
  -d '{
    "title": "<GENERATED_TITLE>",
    "body": "<description>",
    "head": "<branch>",
    "base": "<base-branch>",
    "draft": false
  }'
```

#### GitLab API (if CLI not available)

```bash
curl -s -X POST \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/<project-id>/merge_requests" \
  -d "source_branch=<branch>&target_branch=<base>&title=<GENERATED_TITLE>"
```

### Step 4 – Confirm & Report

After creation, report back:
- PR/MR URL
- Final title used
- Branch → base branch

---

## Edge Cases

- **Branch has no `/`** (e.g. `my-feature-branch`): match against the whole branch name for
  keyword prefixes (`feat`, `fix`, `chore`, etc.) before the first hyphen or underscore.
- **Multi-level slugs** (e.g. `feat/auth/add-oauth`): strip only the first segment, use the rest
  for the message slug.
- **User provides explicit title**: use it as-is (lowercased) — still prepend the prefix unless
  the user's title already starts with `[`. Convert any leading verb to present tense or imperative mood.
- **Monorepos / scoped branches** (e.g. `feat/payments/add-stripe`): include the scope in the
  summary, e.g. `[feature]: add stripe integration to payments`.

---

## Quick Reference — Examples

| Branch | Description | Generated Title |
|---|---|---|
| `feat/create-user` | *(none)* | `[feature]: create user` |
| `fix/login-null-pointer` | Fix null pointer crash on login | `[fix]: fix null pointer crash on login` |
| `chore/update-deps` | *(none)* | `[chore]: update deps` |
| `docs/api-auth-guide` | Add authentication guide | `[docs]: add authentication guide` |
| `refactor/payment-service` | Clean up payment logic | `[refactor]: clean up payment logic` |
| `my-cool-branch` | *(none)* | `[feature]: my cool branch` |
| `feature/user-profile-avatar` | Allow users to upload avatars | `[feature]: allow users to upload avatars` |
