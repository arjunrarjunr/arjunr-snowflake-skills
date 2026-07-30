---
name: creating-git-issue-commit
description: "Analyzes staged git changes and generates issues.md (GitHub issue) and/or commit.md (conventional commit message) based on user request. Use when: user asks to generate issue, commit, or both from staged files. Triggers: generate issue, generate commit, issue and commit from staged, create issue commit files, create issue, create commit."
---

# Git Issue & Commit Generator

Analyzes staged git changes and produces one or both of:
- `issues.md` - structured Github issue
- `commit.md` - conventional commit message

## Intent Detection

Determine which output(s) the user wants:

| User Request | Output |
|--------------|--------|
|"generate issue", "create issue" | `issues.md` only |
|"generate commit", "create commit" | `commit.md` only |
|"generate issue and commit", "create issue commit files", "both" | Both Files |
| Ambiguous or unspecified | Ask the user which file(s) to generate |


Set `GENERATE_ISSUE` and `GENERATE_COMMIT` flags accordingly before proceeding.

## Workflow

### Step 1: Check Staging Area
```powershell
git status --short 
git diff --cached --stat
```
**If nothing is staged:** Inform the user and stop. Do not proceed.

### Step 2: Analyze Staged Changes

```powershell
git diff --cached
git diff --cached --name-status
```

Determine:
- What was added, modified, or deleted
- The scope: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`, `build`
- Whether there are breaking changes
- The intent behind the changes from code context

### Step 3: Generate issues.md (if GENERATE_ISSUE)

**Skip this step if the user only requested a commit message.**

Write `issues.md` to the repository root using the Write tool with this structure:

```markdown
---
Title: <concise imperative title>
Labels: <enhancement|bug|documentation|refactor|etc.>
---

## Description
<What this issue addresses>

## Motivation
<Why the change is needed>

## Proposed Solution
<What the staged changes implement>

## Acceptance Criteria
- [ ] <testable criterion>
- [ ] <testable criterion>
- [ ] <testable criterion>
```

### Step 4: Generate commit.md (if GENERATE_COMMIT)

**Skip this step if the user only requested an issue.**

Write `commit.md` to the repository root using the Write tool:

```markdown
<type>(<scope>): <imperative summary, max 72 chars>

<body: what changed and why, wrapped at 72 chars>

Refs: #<issue reference if applicable>
```

**Rules for commit.md:**
- Type must be one of: feat, fix, refactor, docs, test, chore, style, perf, ci, build
- Summary in imperative mood ("add" not "added")
- Body explains reasoning, not just restates the diff

### Step 5: Present for Review

**MANDATORY STOPPING POINT**: Present the generated file(s) to the user for review before finalizing.

## Error Handling

| Situation | Action |
|-----------|--------|
| No staged files | Inform user, stop |
| Binary files staged | Skip them, note in issue under Additional Context |
| Ambiguous intent (issue vs commit vs both) | Ask user which output they want |
| Ambiguous change intent | Generate best interpretation, flag for human review |

## Stopping Points

- After Step 5: user must approve or request changes before files are finalized