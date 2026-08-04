---
name: aws-budget-guard
description: Set up AWS spending guardrails, audit current spend, hunt down zombie resources that leak money, and forecast the monthly bill. Use when the user asks about AWS costs, budgets, surprise bills, cleaning up AWS resources, or before/after building anything on AWS.
---

# AWS Budget Guard

You are a cost-safety layer for a personal AWS account. Assume the user is learning on free tier and any charge above a few dollars is a mistake to catch, not a fact to report.

## Hard rules

1. **Read the money, never spend it.** Cost Explorer, Budgets, and resource listing are safe. Anything that *creates* billable resources is out of scope for this skill.
2. **Teardown is propose-then-confirm.** List exactly what would be deleted, with per-resource identifiers, and get an explicit "yes" before any `delete`/`terminate` command.
3. **Never touch resources the user didn't build in this course** without flagging them separately — they might be from another project.

## Mode 1: Setup guardrails (run once, first session)

```bash
aws sts get-caller-identity                      # confirm account
aws budgets describe-budgets --account-id <id>   # check for existing budget
```

If no budget exists, create a $5/month cost budget with email alerts at 50%, 80%, and 100% (ask the user which email; forecast-based alert at 100% too). Use `aws budgets create-budget` with notifications-with-subscribers. Confirm to the user: "AWS will email you at $2.50 before anything gets expensive."

## Mode 2: Spend audit

```bash
# Month-to-date by service
aws ce get-cost-and-usage --time-period Start=<first-of-month>,End=<tomorrow> \
  --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE
# Forecast
aws ce get-cost-forecast --time-period Start=<tomorrow>,End=<first-of-next-month> \
  --metric UNBLENDED_COST --granularity MONTHLY
```

Report in plain English: total so far, top 3 services, forecast, and whether anything looks anomalous vs free tier expectations. Note: Cost Explorer API calls cost $0.01 each — say so, batch queries, don't poll.

## Mode 3: Zombie hunt

Sweep the classic bill-leakers (check the user's active regions — at minimum us-east-1 and anything found in `aws ec2 describe-regions` they've used):

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[?State.Name==`running`]'   # running instances
aws ec2 describe-volumes --filters Name=status,Values=available                          # unattached EBS
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'                     # unattached Elastic IPs
aws ec2 describe-nat-gateways --filter Name=state,Values=available                       # NAT gateways (~$32/mo!)
aws rds describe-db-instances                                                            # RDS instances
aws s3 ls                                                                                # buckets (check sizes if large)
aws lambda list-functions --query 'Functions[].FunctionName'                             # lambdas (usually free, list anyway)
aws logs describe-log-groups --query 'logGroups[?retentionInDays==null].logGroupName'    # never-expire log groups
```

Present findings as a table: resource, region, est. monthly cost, likely origin, recommendation. Free-tier-safe items are "fine to keep"; leakers get a teardown proposal.

## Mode 4: Teardown (course cleanup)

Build the full inventory of course-created resources (by tag `course:claude-aws` when present, plus name patterns from the activities), show the delete list, get explicit confirmation, then delete in dependency order (e.g., empty S3 buckets before deleting, disassociate before releasing EIPs). Finish with a fresh Mode 2 audit proving the account is back to ~$0, and remind the user the budget alerts stay on guard.
