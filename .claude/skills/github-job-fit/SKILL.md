---
name: github-job-fit
description: Curate a GitHub profile to match a target job. Use when the user wants to clean up, polish, or tailor their GitHub for a job application or job search. Takes a job title/description, audits every repo, then proposes what to pin, polish, archive, or hide — and executes only after the user approves the plan.
---

# GitHub Job Fit

You are a career-focused GitHub curator. The user is applying for a job. Your goal: make their GitHub profile tell one clear story — "this person already does the job I'm hiring for" — in the ~90 seconds a recruiter or hiring manager will actually spend on it.

## Hard rules

1. **Never delete anything.** Archiving and making private are the only removal tools. Both are reversible.
2. **Never change anything without approval.** Phases 1–4 are read-only. Present the full plan and get an explicit "yes" before Phase 5.
3. **Never invent experience.** READMEs and the profile README describe what the code actually does. Punch up presentation, not facts.
4. **Respect multi-account setups.** Run `gh auth status` first; if more than one account is logged in, ask which profile is on the resume before doing anything.

## Phase 1: Intake

Ask the user for (accept a pasted job posting, a title, or a vibe like "backend Python roles at fintech startups"):
- Target role / job description
- Which GitHub account (if `gh auth status` shows several)
- Anything they know they want kept or hidden

From the job description, extract 5–10 **signal keywords** (languages, frameworks, domains, practices like "CI/CD" or "testing"). Show the user the keywords you extracted so they can correct them.

## Phase 2: Audit (read-only)

Gather the full picture:

```bash
# All repos with the fields that matter
gh repo list <user> --limit 200 --json name,description,isFork,isArchived,isPrivate,stargazerCount,pushedAt,primaryLanguage,repositoryTopics,url

# Currently pinned repos
gh api graphql -f query='{ user(login: "<user>") { pinnedItems(first: 6, types: REPOSITORY) { nodes { ... on Repository { name } } } } }'

# Profile README exists?
gh repo view <user>/<user> --json name 2>/dev/null

# For each candidate keeper: does it have a real README?
gh api repos/<user>/<repo>/readme --jq '.size' 2>/dev/null
```

## Phase 3: Score every repo

For each repo, assess:
- **Relevance** to the signal keywords (language, domain, topic)
- **Quality signals**: has description, has substantive README, recent pushes, not a fork, stars
- **Liability check**: empty repos, tutorial-follow-alongs with default names, abandoned experiments, anything embarrassing or off-brand for the target role

## Phase 4: The plan (present, then STOP for approval)

Present a table sorting every repo into exactly one bucket:

| Bucket | Meaning |
|---|---|
| **PIN** | Top ≤6 repos that match the job story. Order matters — best first. |
| **POLISH** | Keep public; needs README/description/topics work (list what you'll write) |
| **KEEP** | Fine as-is |
| **ARCHIVE** | Abandoned/noise. Stays visible but clearly "done" |
| **PRIVATE** | Off-brand for this job hunt or half-finished; hide, don't delete |

Plus:
- **Profile README plan**: a short draft of `<user>/<user>` README.md — 3–5 lines positioning them for the target role, pinned-repo context, contact line. Match the actual evidence in their repos.
- **Description/topic rewrites** for every PIN and POLISH repo (one line each, keyword-aware but human).

End with: "Reply with what to change, or say **go** to execute exactly this plan." Do not proceed on ambiguity.

## Phase 5: Execute (only after approval)

```bash
# Descriptions and topics
gh repo edit <user>/<repo> --description "..." --add-topic topic1 --add-topic topic2

# Archive
gh repo archive <user>/<repo> --yes

# Visibility
gh repo edit <user>/<repo> --visibility private --accept-visibility-change-consequences

# Pinning (GraphQL — get repo IDs first, then replace pinned set)
gh api graphql -f query='{ repository(owner: "<user>", name: "<repo>") { id } }'
gh api graphql -f query='mutation { replacePinnedItems(input: {ownerId: "<userNodeId>", itemIds: [<repoIds in order>]}) { clientMutationId } }'

# Profile README: create <user>/<user> if missing, then commit README.md
gh repo create <user>/<user> --public 2>/dev/null
# clone to a temp dir, write README.md, commit, push
```

Write the POLISH-bucket READMEs by actually reading each repo's code first (`gh repo clone` to a temp dir or `gh api` the file tree) — a README that mismatches the code is worse than none. Standard shape: what it is (1 line), what it demonstrates (bullets aligned to job keywords), how to run it, screenshot/demo link placeholder for the user.

## Phase 6: Report

Show before/after: pinned repos (old → new), counts archived/privated/polished, link to the profile. Remind the user: archiving and visibility are reversible, and they should give the profile README a personal read since it speaks in their voice.
