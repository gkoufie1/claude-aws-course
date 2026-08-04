# Claude Code × AWS
### Ship 5 real cloud services this weekend — by directing AI, not memorizing commands.

**Install one thing. Claude builds the rest. You learn to direct.**

By Sunday night you'll have: a live website on a global CDN, a public REST API with a database, a bot that emails you every morning, an AI-powered service, and — the part most courses skip — a clean **$0.00** AWS bill and the receipts to prove it. All of it on your GitHub, where employers can see it.

---

## What you'll ship

| # | Activity | You'll ship | You'll learn | Time |
|---|---|---|---|---|
| 0 | [Setup](guide/00-setup.md) | A fully-armed cloud toolchain | Claude Code, `aws login`, IAM, budget guardrails | 30 min |
| 1 | [Static site](guide/01-static-site.md) | A live website with a real URL | S3, CloudFront, GitHub Actions, OIDC | 30 min |
| 2 | [Serverless API](guide/02-serverless-api.md) | A link shortener anyone can call | Lambda, API Gateway, DynamoDB | 20 min |
| 3 | [Automation bot](guide/03-automation-bot.md) | A bot that emails you daily stats | EventBridge, SES, CloudWatch | 20 min |
| 4 | [AI service](guide/04-claude-powered-api.md) | Haiku-as-a-service (really) | Amazon Bedrock, IAM auth, cost control | 25 min |
| 5 | [Capstone + teardown](guide/05-capstone-teardown.md) | One connected app — then a spotless account | Architecture, auditing, leaving no trace | 30 min |

Every chapter was **written from a real dry-run** — we ran every command, hit every error, and the troubleshooting sections contain only failures that actually happened, with the exact fixes that actually worked. When something breaks for you, odds are it broke for us first.

## The method (and why it's worth learning)

This course teaches the skill that's replacing command memorization: **directing an AI agent through real infrastructure work**. Each chapter gives you battle-tested prompts to paste into Claude Code, checkpoints so you always know you're on track, and a collapsible log of what Claude actually ran — for the day you want to understand the machinery.

You will never copy a 40-line command from a blog post. You will learn why the bucket is private, why the bot has two IAM roles, why the AI endpoint has no API key — because understanding *why* is your job. Typing was never the valuable part.

## What's included

- **Six chapters** (setup + five builds), each dry-run validated end to end
- **Battle-tested starter prompts** — including the undocumented quirks we found the hard way (GitHub's new OIDC claim format; Bedrock's picky use-case form) pre-baked so your run goes smoothly
- **A 20-question final exam** — say "quiz me" and Claude administers it one question at a time, grades you, and builds a personalized review list from your misses
- **Three Claude Code skills that install automatically** when you open this folder:
  - `aws-budget-guard` — spending audits, zombie-resource hunts, and the propose-then-confirm teardown. The reason this course can promise a ~$0 bill.
  - `github-job-fit` — point it at a job posting and it reshapes your GitHub profile to match (pins, READMEs, archiving the noise). Your five new repos deserve good staging.
  - `course-quiz` — the final-exam engine: 20 questions drawn from what you actually built
- **The teardown runbook** — most courses leave you with a slowly-leaking cloud account; this one ends with a verified-empty tag sweep and a budget alarm that keeps watching

## What you need

- A **Claude account** (Pro or Max) — the one thing that powers everything
- A free **GitHub** account
- An **AWS** account (free tier; card required at signup — Chapter 0's first real act is arming a $5 budget alarm so you stay at ~$0)

**Total cost of everything in this course: $0–2.** No API credits to buy, no other subscriptions — the AI activity runs on Amazon Bedrock at fractions of a cent, on the same AWS bill your alarm watches. We measured our own dry-run: the bill read **$0.00**.

*Claude runs every command in this course. You keep exactly four jobs — signing into websites, clicking verification links, entering payment details, and approving logins. The things only a human should do.*

## FAQ

**Do I need to know how to code?** No. You need to be able to read English and paste prompts. The course explains every concept the moment it becomes relevant, in plain language.

**What if something breaks?** One rule, every chapter: paste the error into Claude Code and say "fix this." That's not a cop-out — learning to debug by directing is *the course*. Our troubleshooting sections cover every failure our own run hit.

**Is my card safe?** Chapter 0 arms a $5/month budget alarm before you build anything. Every resource the course creates is free-tier or fractions of a cent, tagged for cleanup, and Activity 5 deletes it all — we verify the empty account together.

**How long does it really take?** The chapters total about 2.5 hours of active work. A comfortable weekend pace: setup + site on Saturday morning, the rest on Sunday.

**What do I have when I'm done?** Five deployed-and-documented repos on your GitHub, working knowledge of the eight most-used AWS services, and the direct-an-agent workflow that transfers to every cloud provider and every future tool.

## Start here

→ **[Chapter 0: Setup](guide/00-setup.md)** — one install, then Claude sets up everything else. Fully guided, ~30 minutes.

---

*© Justin Laws. Licensed for individual purchasers — see [LICENSE.md](LICENSE.md). Questions or stuck? Open an issue in this repo — course updates are free forever.*
