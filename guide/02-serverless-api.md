# Activity 2: Your First Serverless API

**Time: ~20 minutes, no long waits · Cost: $0 at rest**
*Written from a real dry-run on 2026-08-02.*

**Builds:** Lambda + API Gateway + DynamoDB link-shortener

## 1 · Goal

By the end of this activity you'll have a **real REST API anyone on the internet can call**: a link shortener. POST it a long URL, it hands back a short one; the short one redirects — and counts every visit.

```
POST /links {"url": "https://something.very/long"}   →  {"short": "https://<api>/Ab3xYz"}
GET  /Ab3xYz                                          →  301 redirect (hit counted)
GET  /stats/Ab3xYz                                    →  {"hits": 2, ...}
```

From our real run — three curls, one working product:

```text
$ curl -X POST $EP/links -H 'Content-Type: application/json' \
    -d '{"url": "https://github.com/you/my-first-site"}'
{"code": "FGgDQV", "short": "https://<api>/FGgDQV", "url": "https://github.com/you/my-first-site"}

$ curl -sI $EP/FGgDQV | grep -i location
301 -> https://github.com/you/my-first-site

$ curl $EP/stats/FGgDQV
{"code": "FGgDQV", "url": "...", "hits": 2, "created": "2026-08-02T15:03:50Z"}
```

The architecture — and the word that matters — is **serverless**: there is no computer running your code right now. **API Gateway** answers the URL, wakes a **Lambda** function (one Python file) for just the milliseconds your request takes, and **DynamoDB** remembers the links. Between requests, everything is asleep and your bill is zero.

After Activity 1's one big slow wait (CloudFront), enjoy this contrast: every piece here comes up in seconds.

## 2 · Starter prompt

> Build me a serverless link shortener on AWS in `us-east-1`, tagging every resource `course=claude-aws`. Use a DynamoDB table (on-demand billing) for the links, one Python Lambda function, and an API Gateway **HTTP API** with three routes: `POST /links` takes a JSON body with a `url` and returns a short code plus the full short URL; `GET /{code}` 301-redirects to the stored URL and increments a hit counter; `GET /stats/{code}` returns the URL, hit count, and creation time. Generate codes as 6 random characters, and guard against collisions with a conditional write. Scope the Lambda's IAM role to only the exact DynamoDB actions it needs on this one table, plus basic logging. If creating the function fails because the brand-new role "cannot be assumed", wait 10 seconds and retry — that's normal. When it's live, test it with `curl` (remember `Content-Type: application/json` on the POST) and show me: creating a short link to a site I'll recognize, following the redirect, and the stats afterward.

> ✅ **Checkpoint:** Claude shows the DynamoDB table `ACTIVE` and the Lambda created.
> ✅ **Checkpoint:** you get an endpoint like `https://<api-id>.execute-api.us-east-1.amazonaws.com`.
> ✅ **Checkpoint:** the three curls work — a `201` with your short URL, a `301` pointing at the original, and stats showing `"hits"` counting up.
![image alt](https://github.com/gkoufie1/claude-aws-course/blob/c282b7b73cdde3d2fab4dffd74980d43cc223c93/lets%20build%20out%20first%20serverless%20api.png)

## 3 · Play with it

This is *your* API on the public internet — prove it:

> Shorten a link to my Activity 1 website, then open the short link in my browser.

Text the short URL to a friend. Watch `hits` go up. That number changing is a database on another continent being updated by code that only exists while it runs — and you shipped it.

## 4 · What Claude actually ran

<details>
<summary>The command log from our real run (click to expand)</summary>

```bash
# The database — one table, on-demand billing, key = the short code
aws dynamodb create-table --table-name shortlinks \
  --attribute-definitions AttributeName=code,AttributeType=S \
  --key-schema AttributeName=code,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST --tags Key=course,Value=claude-aws

# The function's permission slip — exactly 3 DynamoDB actions on 1 table, plus logs
aws iam create-role --role-name shortlink-api-role \
  --assume-role-policy-document '{...trust lambda.amazonaws.com...}'
aws iam attach-role-policy --role-name shortlink-api-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam put-role-policy --role-name shortlink-api-role --policy-name shortlinks-table \
  --policy-document '{...PutItem, GetItem, UpdateItem on table/shortlinks...}'

# The function — one Python file, zipped
zip function.zip lambda_function.py
aws lambda create-function --function-name shortlink-api --runtime python3.13 \
  --handler lambda_function.lambda_handler --role <role-arn> \
  --zip-file fileb://function.zip --timeout 10 --memory-size 128 --tags course=claude-aws

# The front door — HTTP API, three routes, auto-deploying stage
aws apigatewayv2 create-api --name shortlink --protocol-type HTTP --tags course=claude-aws
aws apigatewayv2 create-integration --api-id <id> --integration-type AWS_PROXY \
  --integration-uri <lambda-arn> --payload-format-version 2.0
aws apigatewayv2 create-route --api-id <id> --route-key "POST /links"      --target integrations/<integ>
aws apigatewayv2 create-route --api-id <id> --route-key "GET /stats/{code}" --target integrations/<integ>
aws apigatewayv2 create-route --api-id <id> --route-key "GET /{code}"       --target integrations/<integ>
aws apigatewayv2 create-stage --api-id <id> --stage-name '$default' --auto-deploy

# Let API Gateway wake the Lambda
aws lambda add-permission --function-name shortlink-api --statement-id apigw-invoke \
  --action lambda:InvokeFunction --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:<account>:<api-id>/*"
```

The Lambda itself is ~90 lines of plain Python — routing on `event["routeKey"]`, `boto3` for DynamoDB, a conditional write (`attribute_not_exists`) so two people can never get the same code, and `ADD hits :one` for the counter. Full source: ask Claude to show you, or see the repo you'll make in the stretch goal.

</details>

> 💡 *Route order trivia from the run: `GET /{code}` looks like it would swallow every GET — but API Gateway always matches the most specific route first, so `GET /stats/{code}` wins when it should. You get real routing without writing a router.*

## 5 · When it breaks (from our run and the near-misses)

- **POST returns `{"error": "body must be JSON"}` even though your body looks like JSON** — you forgot `-H 'Content-Type: application/json'` on the curl. Without it, API Gateway base64-encodes the body before your function sees it. We hit this in the dry-run on purpose; you'll hit it by accident.
- **`create-function` fails: "The role defined for the function cannot be assumed"** — brand-new IAM roles take ~10 seconds to propagate. The starter prompt already tells Claude to retry; if you see it anyway, say *"retry it."*
- **The endpoint returns `{"message": "Not Found"}`** — the route doesn't match (check method + path) or the stage isn't deployed. Paste the curl and the response into Claude Code.
- **`500 Internal Server Error`** — the Lambda crashed. Say: *"check the CloudWatch logs for shortlink-api and fix what broke."* Logs are the whole reason that `AWSLambdaBasicExecutionRole` policy exists.

## 6 · Stretch goal

Put it on your GitHub: ask Claude to write a README with the endpoints and a `curl` example, then publish the folder as a public repo — future employers browsing your profile will find a deployed, callable API. (We did: it's the course author's `shortlink-api` repo.)

## 7 · Cleanup

**Nothing to remove — this is the beauty of serverless at rest.** No requests, no compute, no charge: Lambda's free tier is 1M requests/month *forever*, the HTTP API's first million requests/month are free your first year (then ~$1/million), and DynamoDB storage at link-shortener scale is fractions of a cent. Your $5 guardrail watches regardless.

Everything is tagged `course=claude-aws`; the full teardown — table, function, role, API — happens in [Activity 5](05-capstone-teardown.md).

→ Next: [Activity 3: Automation bot](03-automation-bot.md)
