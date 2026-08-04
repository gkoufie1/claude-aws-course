# Activity 4: Ship an AI-Powered Service

**Time: ~25 minutes (includes one ~15-minute AWS wait) · Cost: pennies, on the AWS bill you already watch**
*Written from a real dry-run on 2026-08-02 — including a full redesign mid-build when the first plan cost too much.*

**Builds:** Lambda + Amazon Bedrock (Claude Haiku) + a throttled public API — with **zero API keys**

## 1 · Goal

By the end of this activity you'll have a real AI service on the public internet: **haiku-as-a-service**. POST any topic; Claude writes you a haiku; the response tells you exactly what that request cost (spoiler: about a sixtieth of a cent).

```json
POST /haiku  {"about": "a five dollar budget that never got spent"}

{
  "haiku": "Five dollars sits still\nNever touched through all the days\nDreams unfulfilled wait",
  "model": "claude-haiku-4-5 (via Amazon Bedrock)",
  "this_request_cost": "$0.000157"
}
```

```
the internet ──▶ API Gateway (rate-limited) ──▶ Lambda ──▶ Amazon Bedrock ──▶ Claude
```

## 2 · Why Bedrock (the money lesson, learned the hard way)

Our first draft of this chapter used the Anthropic API directly — until we hit the wall you'd hit too: your **Claude Pro plan covers claude.ai and Claude Code, but not the API**. Direct API access needs separate credits (~$5 minimum) at console.anthropic.com.

**Amazon Bedrock** is AWS's service for serving AI models — including the same Claude Haiku — billed **per-token on your existing AWS bill**. No new account, no minimum purchase, and your Chapter 0 budget alarm already stands guard over it. For this activity that means roughly *2,000 haiku per dollar*, of which you'll use a few cents at most.

The bigger lesson rides along for free: **there is no API key in this build.** Not in the code, not in an environment variable, not in a secrets vault — the Lambda's IAM role *is* the credential, scoped to invoking exactly one model. In cloud security, the best secret is the one that doesn't exist.

> 💡 *If you ever do call an API that needs a real key (Stripe, OpenWeather, the Anthropic API itself), the AWS home for it is **SSM Parameter Store** as an encrypted SecureString, read by the Lambda at startup — never hardcoded, never in GitHub. Ask Claude to "store this key in Parameter Store and let my function read it" and you'll get exactly that pattern.*

## 3 · Starter prompt

One genuinely weird step lives in this build: the first time an AWS account uses Anthropic models on Bedrock, AWS requires a short **use-case form**. The console shows it as a web form — but it can be submitted from the CLI, and the prompt below includes the undocumented detail that makes that work (we found it the hard way).

> Build me an AI haiku service in `us-east-1`, tagging everything `course=claude-aws`. A Python Lambda (boto3 only, no extra packages) that calls **Claude Haiku 4.5 through Amazon Bedrock** using the `converse` API and the cross-region inference profile. Auth must be pure IAM — scope the role to `bedrock:InvokeModel` on just that model, no API keys anywhere. Put it behind an API Gateway HTTP API with one route, `POST /haiku` taking `{"about": "topic"}`, and **throttle the stage to 1 request/second**. Cap `maxTokens` at 200. Have the Lambda compute the actual cost of each request from the returned token counts and include it in the response as `this_request_cost`. Before testing: check whether this account has submitted the Anthropic use-case form (`aws bedrock get-use-case-for-model-access`); if not, submit it via `put-use-case-for-model-access` with my details — note the form's JSON needs `"intendedUsers": "0"` (a code, not a description) or it's rejected as invalid. If the model still returns ResourceNotFoundException after submitting, wait and retry — AWS says up to 15 minutes. Then show me a haiku, with its cost.

> ✅ **Checkpoint:** Claude confirms the use-case form is submitted (or was already).
> ✅ **Checkpoint:** within ~15 minutes (ours took 1), the endpoint returns a real haiku with a `this_request_cost` around `$0.0002`.
> ✅ **Checkpoint:** your budget guardrail didn't even flinch.

## 4 · Poke at it

> Ask my haiku API for haiku about three things I care about, and add up what all three cost.

Then try to break it — that's the point of the throttle:

> Fire 30 requests at my haiku endpoint at once and show me the count of status codes.

You should see mostly `200`s and a handful of `429 Too Many Requests` — the gate slamming. (In our run: 24 served, 6 rejected.)

## 5 · What Claude actually ran

<details>
<summary>The command log from our real run (click to expand)</summary>

```bash
# The permission slip — IAM instead of an API key
aws iam create-role --role-name haiku-api-role --assume-role-policy-document '{...lambda.amazonaws.com...}'
aws iam attach-role-policy --role-name haiku-api-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam put-role-policy --role-name haiku-api-role --policy-name invoke-claude-haiku \
  --policy-document '{...bedrock:InvokeModel on the Haiku 4.5 foundation-model + inference-profile ARNs only...}'

# The one-time Anthropic use-case form — CLI-submittable, undocumented schema
aws bedrock get-use-case-for-model-access                  # check first
aws bedrock put-use-case-for-model-access --form-data "$(printf '%s' \
  '{"companyName":"<you>","companyWebsite":"https://github.com/<you>","intendedUsers":"0",
    "industryOption":"Education","otherIndustryOption":"","useCases":"Personal learning project..."}' | base64)"
#   ^ intendedUsers MUST be the string "0" — prose like "Internal employees" is rejected

# The function — ~70 lines: validate input, bedrock.converse(), compute cost, respond
zip function.zip lambda_function.py
aws lambda create-function --function-name haiku-api --runtime python3.13 \
  --handler lambda_function.lambda_handler --role <role-arn> --zip-file fileb://function.zip \
  --timeout 30 --tags course=claude-aws

# The front door — one route, throttled
aws apigatewayv2 create-api --name haiku --protocol-type HTTP --tags course=claude-aws
aws apigatewayv2 create-integration --api-id <id> --integration-type AWS_PROXY \
  --integration-uri <lambda-arn> --payload-format-version 2.0
aws apigatewayv2 create-route --api-id <id> --route-key "POST /haiku" --target integrations/<integ>
aws apigatewayv2 create-stage --api-id <id> --stage-name '$default' --auto-deploy \
  --default-route-settings '{"ThrottlingRateLimit": 1, "ThrottlingBurstLimit": 2}'
aws lambda add-permission --function-name haiku-api ... --principal apigateway.amazonaws.com
```

The Lambda's Bedrock call, in essence:

```python
bedrock = boto3.client("bedrock-runtime")
result = bedrock.converse(
    modelId="us.anthropic.claude-haiku-4-5-20251001-v1:0",
    system=[{"text": "You write haiku. Reply with ONLY the haiku..."}],
    messages=[{"role": "user", "content": [{"text": f"Write a haiku about: {topic}"}]}],
    inferenceConfig={"maxTokens": 200},
)
# result["usage"] has inputTokens/outputTokens -> compute this_request_cost
```

</details>

## 6 · When it breaks (from our run)

- **`ResourceNotFoundException: Model use case details have not been submitted`** — the form isn't in yet, or the 15-minute clock is still ticking. Confusingly, an occasional request may *succeed* before enforcement propagates — ours did once, then the door closed. Submit the form, wait, retry.
- **`ValidationException: Invalid form data`** on the form submission — the JSON schema is undocumented and picky: `intendedUsers` must be `"0"`, and all six fields must be present. The starter prompt bakes this in.
- **`AccessDeniedException`** — the Lambda role's Bedrock policy doesn't cover the model ARN being invoked. Cross-region inference profiles need *both* the inference-profile ARN and the wildcard-region foundation-model ARN.
- **Your burst test returns all `200`s** — AWS enforces API Gateway throttles *best-effort* across its fleet, so small bursts can slip through (our first burst of 8 all passed). It clamps sustained floods, not trickles. Your layered defense is: throttle + tiny `maxTokens` + the cheapest model + the Chapter 0 budget alarm. Defense in depth, not one perfect wall.

## 7 · Stretch goal

Wire yesterday's bot into today's AI: ask Claude to make your Activity 3 daily digest email **open with a haiku about your link stats**. Two services you built, talking to each other — that's a system, and it's also the warm-up for the capstone.

## 8 · Cleanup

**$0 at rest.** Bedrock has no standing charge — no requests, no cost. The Lambda, API, and role are free at rest as in Activity 2, and model access itself costs nothing. Everything is tagged `course=claude-aws` for [Activity 5](05-capstone-teardown.md)'s teardown.

→ Next: [Activity 5: Capstone + teardown](05-capstone-teardown.md)
