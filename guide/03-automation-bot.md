# Activity 3: A Bot That Works While You Sleep

**Time: ~20 minutes (mostly waiting for one email) · Cost: $0**
*Written from a real dry-run on 2026-08-02 — including the email that didn't arrive.*

**Builds:** EventBridge cron + Lambda + SES daily email digest

## 1 · Goal

Every morning at 8:00 AM, an email lands in your inbox: how many short links you've made, how many times they've been clicked, and your top performers — stats pulled live from the Activity 2 database. You will not be running anything. No computer of yours will be on. The cloud does it, on schedule, forever, for free.

The email that lands, from our real run:

```text
Subject: Link digest: 1 links, 2 clicks

Good morning! Your link shortener as of 2026-08-02 15:10 UTC:

  Links created : 1
  Total clicks  : 2

  Top links:
    /FGgDQV     2 clicks  ->  https://github.com/you/my-first-site

- your daily-digest bot (Lambda + EventBridge + SES)
```

```
EventBridge Scheduler ──8:00 AM──▶ Lambda ──scan──▶ DynamoDB
                                     │
                                     └──send──▶ SES ──▶ your inbox
```

Three new words: **EventBridge Scheduler** (the cloud's alarm clock — timezone-aware, so daylight saving just works), **SES** (Simple Email Service — AWS's email sender), and **CloudWatch** (where your bot's diary lives — every run, every crash, every millisecond billed).

## 2 · Starter prompt

The one step Claude can't do: AWS won't send email *as you* until you prove you own your address, by clicking a link it emails you. So the prompt front-loads that, and the build happens while you wait for the click:

> Build me a daily email bot in `us-east-1`, tagging everything `course=claude-aws`. **First**, create an SES email identity for my address `<your-email>` so the verification email is on its way — tell me to go click it, and poll the verification status in the background. While that's pending, build the bot: a Python Lambda that scans my `shortlinks` DynamoDB table and emails me a digest — link count, total clicks, top 5 — via SES, with the role scoped to exactly that one Scan and one SendEmail. Schedule it with EventBridge Scheduler at 8:00 AM in my timezone (not UTC). The scheduler needs its own small role that's only allowed to invoke this function. The moment my address verifies, do a test invoke, show me the CloudWatch logs, and confirm the email was sent.

> ✅ **Checkpoint:** Claude says the verification email is sent, and it's in your inbox. **If it isn't, see the first troubleshooting entry — this happened to us.**
> ✅ **Checkpoint:** after you click, the test invoke returns something like `{"sent": true, "links": 1}` and the log shows a `REPORT` line — that's CloudWatch telling you your bot ran for ~600ms and used ~96 MB.
> ✅ **Checkpoint:** the digest is in your inbox, with real numbers from *your* table.

> 💡 *SES starts every account in "sandbox" mode: it can only send to verified addresses. Tutorials treat this as a limitation to escape. For a personal bot, it's a feature — your AWS account literally cannot spam anyone.*

## 3 · Trust the schedule

The test invoke proves the pipeline; the schedule proves itself tomorrow at 8:00 AM. Nothing to do tonight — that's the entire point. If you're impatient:

> Change my digest schedule to fire 3 minutes from now, then put it back to 8:00 AM daily after it sends.

## 4 · What Claude actually ran

<details>
<summary>The command log from our real run (click to expand)</summary>

```bash
# Prove you own your address (the click lives on your side)
aws sesv2 create-email-identity --email-identity <you@example.com> --tags Key=course,Value=claude-aws
aws sesv2 get-email-identity --email-identity <you@example.com> \
  --query VerifiedForSendingStatus        # poll until True
aws ses verify-email-identity --email-address <you@example.com>   # RE-send, if the first vanishes

# The bot's permission slip: one Scan, one SendEmail, plus logs
aws iam create-role --role-name daily-digest-role --assume-role-policy-document '{...lambda.amazonaws.com...}'
aws iam attach-role-policy --role-name daily-digest-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam put-role-policy --role-name daily-digest-role --policy-name digest-permissions \
  --policy-document '{...dynamodb:Scan on table/shortlinks, ses:SendEmail on identity/<you>...}'

# The bot itself — one Python file: scan, format, send
zip function.zip lambda_function.py
aws lambda create-function --function-name daily-digest --runtime python3.13 \
  --handler lambda_function.lambda_handler --role <role-arn> --zip-file fileb://function.zip \
  --environment 'Variables={TABLE=shortlinks,SENDER=<you>,RECIPIENT=<you>}' --tags course=claude-aws

# The alarm clock — its own tiny role (even AWS's scheduler needs a permission slip)
aws iam create-role --role-name daily-digest-scheduler-role \
  --assume-role-policy-document '{...scheduler.amazonaws.com...}'
aws iam put-role-policy --role-name daily-digest-scheduler-role --policy-name invoke-digest \
  --policy-document '{...lambda:InvokeFunction on function:daily-digest...}'
aws scheduler create-schedule --name daily-digest-8am \
  --schedule-expression "cron(0 8 * * ? *)" \
  --schedule-expression-timezone "America/New_York" \
  --flexible-time-window Mode=OFF \
  --target '{"Arn": "<lambda-arn>", "RoleArn": "<scheduler-role-arn>"}'

# The proof
aws lambda invoke --function-name daily-digest --payload '{}' /dev/stdout
aws logs tail /aws/lambda/daily-digest --since 2m
```

</details>

> 💡 *Why EventBridge **Scheduler** and not the older EventBridge cron rules you'll see in tutorials? Timezones. Classic rules speak only UTC, and your 8:00 AM email would silently shift an hour twice a year. Scheduler takes `America/New_York` and handles daylight saving itself.*

## 5 · When it breaks (from our run)

- **The verification email never arrives.** This happened in our dry-run. University and corporate mailboxes (Microsoft 365 especially) love to eat AWS mail. In order: check **Junk**, check the **Other** tab, search for "aws", wait 3 minutes. Then tell Claude: *"resend my SES verification email"* (that's the `verify-email-identity` command — it worked for us on the second send). Still nothing? Your mail gateway is quarantining it — use a personal Gmail-type address for the bot instead; that's the better home for AWS notifications anyway.
- **The test invoke fails with `Email address is not verified`.** You haven't clicked the link yet (or you verified a different address than the one in the function's environment). `get-email-identity` must say `True` first.
- **The invoke works but no email lands.** Check spam first — your own bot's first email often starts there. Mark it "not spam" once and your mail provider learns.
- **Tomorrow's 8:00 AM email doesn't come.** Ask Claude: *"did my daily-digest schedule fire? Check the schedule state and the CloudWatch logs from 8 AM."* The logs will say whether it ran and crashed, or never ran (usually the scheduler role's permission).

## 6 · Stretch goal

Make the digest yours: ask Claude to add a random inspirational quote, tomorrow's weather via a free API, or — the power move — **yesterday's AWS spend** from Cost Explorer, so your bot becomes your own budget guard. (One honest cost note if you do: each Cost Explorer API call bills $0.01 — a daily check is ~$0.30/month, the only non-free thing in this course. Your call whether the peace of mind is worth thirty cents.)

Then publish the bot as a public repo like Activity 2 — a scheduled, production cron job is a legitimately good portfolio piece.

## 7 · Cleanup

**What keeps running: your bot — on purpose.** That's the deliverable. At rest it's $0: EventBridge Scheduler's free tier is 14 *million* invocations a month (you'll use 30), Lambda's free tier covers a 600ms daily run thousands of times over, and SES's ~30 emails/month cost either nothing (first-year free tier) or about a third of a cent after.

Want it quiet for a while? One command pauses it without deleting anything: ask Claude to *"disable my daily-digest schedule"* (`aws scheduler update-schedule --state DISABLED`). Full teardown, as always, is [Activity 5](05-capstone-teardown.md) — everything here is tagged `course=claude-aws`.

→ Next: [Activity 4: Claude-powered service](04-claude-powered-api.md)
