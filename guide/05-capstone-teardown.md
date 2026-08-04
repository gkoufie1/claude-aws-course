# Activity 5: Capstone — Build It All, Then Leave No Trace

**Time: ~30 minutes (plus one ~10-minute CloudFront wait) · Cost: ends at literally $0**
*Written from a real dry-run on 2026-08-02 — we really did delete everything.*

**Builds:** the finale in two acts — wire your services into one app, then tear the whole account back to zero.

## 1 · Goal

Two things happen in this activity, and they're both the point of the course:

**Act I — the capstone.** Your static site (Activity 1) gets an AI widget: a visitor types a topic, the page calls your Lambda (Activity 4), Claude writes a haiku, and the site displays it — with the cost of that exact request underneath. One user-visible app, powered by five things you built: S3, CloudFront, GitHub Actions, Lambda, Bedrock.

**Act II — the teardown.** You audit the account, see the bill (**ours read $0.00**), and then delete every course resource — cleanly, in the right order, with a final sweep proving nothing is left. Knowing how to *leave* a cloud account is the skill that separates people who've done tutorials from people who've run infrastructure.

From our real run — a visitor typed "IAM roles instead of API keys" into the widget and got back:

> *Identities bloom / Access control without keys / Permissions flow safe*
>
> `that haiku cost $0.000141 to generate`

## 2 · Act I — Starter prompt: connect the pieces

> Add an AI haiku widget to my `my-first-site` page: an input box and button that call my haiku API from the browser and display the haiku plus its `this_request_cost`. Match the page's existing style. My haiku API will need CORS configured to allow exactly my CloudFront origin — set that on the API, not `*`. Then commit and push, watch the deploy, and confirm the live site works.

> ✅ **Checkpoint:** the Actions deploy is green, and on your live site you can type "my cat" and get a haiku about your cat, with a price tag of ~$0.0002.
> ✅ **Checkpoint (the quiet one):** notice what just happened — a browser talked to your CDN, which served a page that called your API gateway, which invoked your function, which called a foundation model, and the whole round trip cost a sixtieth of a cent. You built every link in that chain.

## 3 · Act II — Audit first, then propose, then delete

**Never delete from memory — inventory from tags.** Every resource this course created carries the tag `course=claude-aws`, and now it pays off:

> Use the aws-budget-guard skill: audit my month-to-date AWS costs, then sweep every resource tagged `course=claude-aws` (plus IAM roles and log groups, which the tag sweep misses) and show me the full list with what each would cost to keep. Don't delete anything yet.

> ✅ **Checkpoint:** a table of everything you built, and a month-to-date bill at or near **$0.00**. (Cost Explorer lags ~24h, so today's fractions of a cent appear tomorrow. Also: each Cost Explorer API call itself costs $0.01 — the one deliberate cent this course ever spends.)

Then, the confirmation ritual — the skill requires your explicit yes:

> Tear it all down. Keep my IAM user, my budget guardrail, and don't touch anything not on the course list.

> ✅ **Checkpoint:** Claude re-lists exactly what will be deleted and waits for you to say yes. **This pause is not bureaucracy — it's the habit that prevents the resume-generating event.** Say yes.
> ✅ **Checkpoint:** after the run (CloudFront takes ~10 minutes to disable before it can be deleted — everything else is seconds), a fresh tag sweep comes back **empty**.

## 4 · What Claude actually ran

<details>
<summary>The teardown, in dependency order (click to expand)</summary>

```bash
# 1. CloudFront first — it's the slow one. Disable, then delete after it settles.
aws cloudfront update-distribution --id <DIST_ID> ... Enabled=false
#    (~10 min wait; do everything else meanwhile)

# 2. Front doors — deleting an API takes its routes, integrations, and stages with it
aws apigatewayv2 delete-api --api-id <haiku-api-id>
aws apigatewayv2 delete-api --api-id <shortlink-api-id>

# 3. The bot's alarm clock, then all three functions
aws scheduler delete-schedule --name daily-digest-8am
aws lambda delete-function --function-name shortlink-api   # + daily-digest, haiku-api

# 4. Data and identities
aws dynamodb delete-table --table-name shortlinks
aws sesv2 delete-email-identity --email-identity <you@example.com>

# 5. The diaries
aws logs delete-log-group --log-group-name /aws/lambda/<each function>

# 6. Permission slips — detach managed policies, delete inline ones, then the role
aws iam detach-role-policy ... && aws iam delete-role-policy ... && aws iam delete-role ...
#    (first-site-deploy, shortlink-api-role, daily-digest-role,
#     daily-digest-scheduler-role, haiku-api-role)
aws iam delete-open-id-connect-provider --open-id-connect-provider-arn <github-oidc-arn>

# 7. The site — empty the bucket before deleting it
aws s3 rm s3://first-site-<ACCOUNT_ID> --recursive
aws s3api delete-bucket --bucket first-site-<ACCOUNT_ID>

# 8. Once CloudFront reports Deployed+disabled: delete it, then its access control
aws cloudfront delete-distribution --id <DIST_ID> --if-match <etag>
aws cloudfront delete-origin-access-control --id <OAC_ID> --if-match <etag>

# 9. Prove it
aws resourcegroupstaggingapi get-resources --tag-filters Key=course,Values=claude-aws
#    → empty. Done.
```

</details>

## 5 · What survives (on purpose)

| Kept | Why |
|---|---|
| **All five GitHub repos** | Teardown never touches GitHub. Your profile still shows a deployed site, two APIs, a bot, and this course's story — that's your portfolio. |
| **The $5 budget guardrail** | It costs nothing and watches forever. If anything ever starts spending in this account again, you get the email. |
| **Your `claude-course` IAM user** | It's how your CLI works. If you're truly done with AWS, sign in as root and delete it — the final door closed. |

And the claim that makes the teardown painless: **everything you deleted is rebuildable in about an hour** — the chapters *are* the runbook. Re-run activities 1–4 and every service comes back. Infrastructure you can recreate from instructions is infrastructure you own; infrastructure you're afraid to delete owns you.

## 6 · When it breaks (from our run)

- **CloudFront won't delete:** it must be *disabled* first, and the disable takes ~10 minutes to reach `Deployed` status. The OAC can't be deleted until the distribution is gone. Patience, not errors.
- **`delete-role` fails with policies attached:** detach managed policies and delete inline policies first — the commands above do it in order.
- **The site repo's Actions runs fail after teardown:** expected — the deploy role no longer exists. The repo is now an archive of what you built, not a live pipeline.
- **Cost Explorer still shows $0.01–0.30 tomorrow:** that's the Cost Explorer API calls themselves and today's Bedrock pennies posting late. The budget alarm stays silent.

## 7 · You're done

Five services shipped. One account, back to zero, guardrail still armed. A GitHub profile that shows all of it. And the actual skill — the one this course was always about — is now yours: **you know how to direct.**

**One last thing — take the final exam.** Tell Claude: *"quiz me."* Twenty questions, one at a time, each one something you actually built this weekend. It's not a gate; it's how the knowledge sets.

*Where next: re-run any chapter and extend it. Give the shortlinks API auth. Put a custom domain on the site. Swap the haiku model for a smarter one. You have the method — paste the error, describe the goal, direct the work.*
