---
name: course-quiz
description: Administer the course's 20-question final quiz on Claude Code and AWS. Use when the student asks to be quizzed, tested, or examined, says "quiz me", or has just finished Activity 5 and accepts the offer of the final exam.
---

# The Final Exam — 20 Questions

You are administering this course's final quiz. The goal is not gatekeeping — it's cementing. Every question tests something the student actually did, and every explanation reconnects the fact to the moment they lived it.

## How to run it (strict)

1. **One question at a time.** Show the question number (e.g. `Question 7 of 20`), the question, and options if multiple-choice. Then STOP and wait for their answer. Never show more than one question per message. Never dump the question list.
2. **Grade immediately, teach briefly.** Right or wrong, give the answer plus the one-paragraph *why*, and name the chapter it came from. Wrong answers get warmth, not consolation prizes — "not quite, and here's the thing to remember" beats "good try!"
3. **Keep score silently.** No running score announcements — just a checkmark or cross per question. Reveal the total only at the end.
4. **Adapt the ending.** After question 20 give: score out of 20, a one-line verdict by band, and a personalized review list — for each miss, the chapter section to reread. Offer a retake of only the missed questions.
5. **Bands:** 18–20 *"Ship it. You didn't just take a course; you ran infrastructure."* · 14–17 *"Solid — review the misses and you're there."* · 10–13 *"The method stuck; some machinery didn't. Reread the chapters below."* · <10 *"Rerun an activity or two — the doing is what makes it stick."*
6. If they ask for a hint, give one — the goal is learning. Score a hinted correct answer as half right (round generously in their favor).

## The questions

**Q1 (MCQ, Ch0).** Chapter 0 has you create a `claude-course` IAM user instead of working as the root account. Why?
A) Root can't use the CLI · B) Root is the master key — if its credentials leak, everything is lost, so daily work happens on a scoped, deletable identity · C) IAM users are free, root costs money · D) AWS requires it
**Answer: B.** Root stays locked away for break-glass moments; the working user can be deleted and re-made. (Ch0 §4b)

**Q2 (MCQ, Ch0).** What makes `aws login` safer than the old access-key method?
A) It uses a longer password · B) It's encrypted, keys aren't · C) It issues temporary, auto-refreshing credentials through your browser session — there's no permanent secret file to leak · D) It requires MFA
**Answer: C.** Old tutorials have you download a CSV of forever-keys; `aws login` credentials expire on their own. (Ch0 §4c)

**Q3 (MCQ, Ch0).** Your $5 budget alarm crosses 50%. What actually happens?
A) AWS shuts down your services · B) AWS blocks new resources · C) You get an email at ~$2.50 — and nothing else; budgets inform, they don't act · D) Your card is charged $5
**Answer: C.** A budget is a smoke detector, not a sprinkler system. Knowing early is the protection. (Ch0 §4d)

**Q4 (MCQ, Ch1).** Your website is public, but its S3 bucket is private. How does anyone see the site?
A) CloudFront has a signed pass (Origin Access Control) to read the bucket and serves it to the world — the CDN is the only front door
B) The bucket becomes public at request time · C) GitHub serves the files · D) Lambda copies files to viewers
**Answer: A.** Nobody can bypass the CDN or hit the bucket directly — that's the "public bucket horror story" prevention. (Ch1)

**Q5 (short, Ch1).** GitHub Actions deploys your site to AWS with zero stored keys. What's the mechanism called, in one word-ish?
**Answer:** OIDC (OpenID Connect federation) — GitHub proves the workflow's identity with a signed token; an IAM role trusts exactly that repo and branch. (Ch1)

**Q6 (T/F, Ch1).** True or false: your site redeploys on push because AWS watches your GitHub repo for changes.
**Answer: False.** AWS watches nothing. *Your* GitHub Actions workflow runs on push, logs into AWS, syncs the files, and invalidates the cache. The robot is yours. (Ch1)

**Q7 (MCQ, Ch1).** In our dry-run, the first deploy failed with `Not authorized to perform sts:AssumeRoleWithWebIdentity`. The cause?
A) Wrong AWS password · B) GitHub's OIDC `sub` claim now embeds numeric account/repo IDs, so the classic `repo:owner/name` trust policy never matches · C) The bucket was full · D) IAM was down
**Answer: B.** Every old tutorial shows the pre-ID format. The fix: read the token's real claims, match exactly. (Ch1, troubleshooting)

**Q8 (short, Ch2).** "Serverless" doesn't mean there are no servers. What does it actually mean for your wallet?
**Answer:** Nothing runs (or bills) between requests — the function exists for milliseconds per call, so at rest the cost is zero. Pay-per-use, not pay-for-idle. (Ch2)

**Q9 (MCQ, Ch2).** The link shortener generates random 6-character codes. Why write with `attribute_not_exists(code)`?
A) Speed · B) It's required by DynamoDB · C) Collision guard — if two requests draw the same code, the second write fails instead of silently overwriting someone's link · D) Cheaper writes
**Answer: C.** A conditional write turns "rare disaster" into "harmless retry." (Ch2)

**Q10 (MCQ, Ch2).** Your `curl` POST returns `{"error": "body must be JSON"}` but the body IS valid JSON. Most likely cause?
A) DynamoDB is down · B) You forgot `-H 'Content-Type: application/json'`, so API Gateway base64-encoded the body before the Lambda saw it · C) The JSON has too many keys · D) Rate limiting
**Answer: B.** We hit this in the dry-run on purpose; everyone hits it by accident. (Ch2, troubleshooting)

**Q11 (T/F, Ch2).** True or false: the route `GET /{code}` swallows every GET request, so `GET /stats/{code}` can never match.
**Answer: False.** API Gateway matches the most specific route first — real routing without writing a router. (Ch2)

**Q12 (MCQ, Ch3).** Why EventBridge **Scheduler** instead of the classic cron rules in older tutorials?
A) It's cheaper · B) Classic rules only speak UTC — your 8:00 AM email would shift an hour twice a year; Scheduler takes a timezone and handles DST · C) Cron is deprecated · D) It runs faster
**Answer: B.** `cron(0 8 * * ? *)` in `America/New_York` means 8 AM *your* time, always. (Ch3)

**Q13 (MCQ, Ch3).** The digest bot needed TWO IAM roles — one for the Lambda, one for the scheduler. Why does the scheduler need its own?
A) AWS billing requires it · B) Because even AWS's own services get no free pass — the scheduler must hold explicit permission to invoke your function, scoped to just that function
C) Lambda roles expire nightly · D) It doesn't; it was optional
**Answer: B.** Least privilege applies to robots too. (Ch3)

**Q14 (MCQ, Ch3).** SES sandbox mode means your account can only email verified addresses. The course calls this a feature. Why?
A) It's faster · B) Verified mail skips spam filters · C) Your AWS account is structurally incapable of spamming anyone — a perfect property for a personal bot · D) It's free
**Answer: C.** Tutorials treat sandbox as a wall to climb; for a self-mailing bot it's a guarantee. (Ch3)

**Q15 (short, Ch4).** The haiku API calls Claude with **zero API keys** anywhere. How does the Lambda authenticate to Bedrock?
**Answer:** Its IAM role — scoped to `bedrock:InvokeModel` on exactly one model. The credential is the identity itself; nothing to store, nothing to leak. (Ch4)

**Q16 (MCQ, Ch4).** Why does the AI endpoint get a 1 request/second throttle when the other APIs didn't?
A) Bedrock requires it · B) Every call to this endpoint spends real money on your bill — the throttle is the wallet's bodyguard against a flood · C) Haiku are slow to write · D) Lambda can't scale
**Answer: B.** Free-at-rest APIs can shrug off floods; pay-per-call APIs can't. (Ch4)

**Q17 (short, Ch4).** Our 30-request burst saw 24 pass and only 6 rejected — the throttle is "best-effort." Name at least two OTHER layers protecting the bill.
**Answer (any two):** tiny `maxTokens` cap (200) · the cheapest model (Haiku) · the $5 budget alarm · (also acceptable: input length validation). Defense in depth — no single wall, four short ones. (Ch4)

**Q18 (MCQ, Ch5).** Why did every resource in this course get the tag `course=claude-aws`?
A) AWS requires tags · B) Tags make resources cheaper · C) So one tag-sweep query can inventory — and later verify the deletion of — everything the course created, no memory required · D) For prettier console screens
**Answer: C.** The teardown's proof-of-empty is a tag query returning `[]`. (Ch5)

**Q19 (MCQ, Ch5).** In the teardown, why does CloudFront go first?
A) It's the most expensive · B) Alphabetical order · C) It must be *disabled* and settle (~10 min) before it can be deleted — start the slow thing, do the quick deletes while it drains · D) AWS mandates the order
**Answer: C.** Dependency-aware ordering: slow async first, quick deletes in parallel, dependents before dependencies (empty bucket → delete bucket; distribution → then its OAC). (Ch5)

**Q20 (short, Ch5).** Three things deliberately survive the teardown. Name two, and say why in a phrase each.
**Answer (any two):** the GitHub repos (your portfolio — teardown never touches GitHub) · the $5 budget alarm (costs nothing, guards forever) · the `claude-course` IAM user (your CLI access; delete it from root only when done with AWS entirely). (Ch5)

## After the quiz

If they scored under 18, remind them: every deleted service rebuilds in about an hour by re-running the chapters — and rebuilding from scratch is the single best review session that exists.
