# Activity 1: A Live Website in 20 Minutes

**Time: ~20 minutes of work + one ~10-minute AWS wait · Cost: $0**
*Written from a real dry-run on 2026-08-02 — including the part where it broke.*

**Builds:** S3 static hosting + CloudFront + GitHub Actions auto-deploy

## 1 · Goal

By the end of this activity you'll have a web page of your own at a real `https://` URL, served worldwide by a CDN, that **redeploys itself every time you push to GitHub**. The pipeline you're building:

```
you edit → git push → GitHub Actions → S3 bucket (private) → CloudFront → the world
```

When ours went live it was a cream-colored page reading *"This site lives in an S3 bucket, is served worldwide by CloudFront, and redeploys itself every time I push to GitHub. I didn't memorize any of those words a week ago."* Yours will say whatever you want — that's rather the point.

Four words you'll own by the end: **bucket** (S3's folder-in-the-cloud), **distribution** (CloudFront's copy of your site on servers worldwide), **workflow** (the robot that redeploys), **OIDC** (how that robot logs into AWS without a stored password).

## 2 · Starter prompts

**First, the site itself:**

> Create a folder called `my-first-site` with a simple, tasteful one-page personal site — just `index.html`, no frameworks. Put my name on it. Then make it a git repo and publish it to GitHub as a public repo with `gh`.

> ✅ **Checkpoint:** `github.com/<you>/my-first-site` exists and shows your `index.html`.

**Then, the deployment — this is the big one:**

> Deploy `my-first-site` to AWS the modern way: a **private** S3 bucket served through CloudFront with Origin Access Control — do not make the bucket public. Region `us-east-1`, and tag every resource `course=claude-aws`. Then set up auto-deploy: a GitHub Actions workflow that syncs to the bucket and invalidates the CloudFront cache on every push to main, authenticating to AWS with **OIDC — no access keys stored in GitHub**. Careful with the IAM role's trust policy: GitHub's OIDC `sub` claim now embeds numeric IDs (`repo:OWNER@ownerid/REPO@repoid:ref:...`), so look up my repo's and my account's numeric IDs with `gh api` and use the exact claim. Scope the role to only this bucket and this distribution's invalidations. When everything is pushed and the first deploy run is green, give me my CloudFront URL.

While Claude works, you'll see the one genuinely slow AWS moment of this course: the CloudFront distribution takes **5–15 minutes** to copy your site to edge servers around the planet. Status goes `InProgress` → `Deployed`. Get a snack; Claude will keep wiring up the deploy robot meanwhile.

> ✅ **Checkpoint:** Claude reports the bucket exists and the distribution is created (status `InProgress` is fine at this point).
> ✅ **Checkpoint:** the GitHub Actions run is green — all steps ✓, including "Sync site to S3" and "Invalidate CloudFront cache".
> ✅ **Checkpoint:** your `https://<something>.cloudfront.net` URL loads your page, with the padlock.
![image alt](https://github.com/gkoufie1/claude-aws-course/blob/79e30e3e185474c392262767902b25cd7d6e2f69/Yes.png)

## 3 · Prove the loop

The URL working once is nice. The *loop* is the point:

> Change the headline on my site to say something new, commit, and push. Then watch the Actions run and tell me when the change is live at my CloudFront URL.

> ✅ **Checkpoint:** within about a minute of the push, a hard refresh (`Cmd+Shift+R` / `Ctrl+Shift+R`) shows your new headline. You now have a website that deploys itself. This exact loop — edit, push, live — is how you'll ship for the rest of the course.

## 4 · What Claude actually ran

<details>
<summary>The command log from our real run (click to expand)</summary>

```bash
# The bucket — private, tagged, site uploaded
aws s3api create-bucket --bucket first-site-<ACCOUNT_ID>
aws s3api put-public-access-block --bucket first-site-<ACCOUNT_ID> \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-tagging --bucket first-site-<ACCOUNT_ID> --tagging 'TagSet=[{Key=course,Value=claude-aws}]'
aws s3 sync my-first-site s3://first-site-<ACCOUNT_ID> --exclude ".git/*"

# CloudFront — Origin Access Control, then the distribution
aws cloudfront create-origin-access-control --origin-access-control-config \
  Name=first-site-oac,OriginAccessControlOriginType=s3,SigningBehavior=always,SigningProtocol=sigv4
aws cloudfront create-distribution-with-tags --distribution-config-with-tags file://cf-dist.json
#   (config: S3 origin + OAC, default root index.html, redirect-to-https, PriceClass_100)

# Let only this distribution read the private bucket
aws s3api put-bucket-policy --bucket first-site-<ACCOUNT_ID> --policy '{...Principal cloudfront.amazonaws.com,
  Condition AWS:SourceArn = <distribution ARN>...}'

# GitHub → AWS trust, no keys: OIDC provider + a tightly-scoped role
gh api repos/<you>/my-first-site --jq '.id'          # repo's numeric ID
gh api users/<you> --jq '.id'                        # your account's numeric ID
aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com --thumbprint-list <github thumbprints>
aws iam create-role --role-name first-site-deploy --assume-role-policy-document '{...
  "sub": "repo:<you>@<your-id>/my-first-site@<repo-id>:ref:refs/heads/main"...}'
aws iam put-role-policy --role-name first-site-deploy --policy-name deploy-site \
  --policy-document '{...s3 list/get/put/delete on the bucket, cloudfront:CreateInvalidation on the distribution...}'
```

And the workflow, `.github/workflows/deploy.yml`:

```yaml
name: Deploy site
on:
  push:
    branches: [main]
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/first-site-deploy
          aws-region: us-east-1
      - name: Sync site to S3
        run: aws s3 sync . s3://first-site-<ACCOUNT_ID> --exclude ".git/*" --exclude ".github/*" --delete
      - name: Invalidate CloudFront cache
        run: aws cloudfront create-invalidation --distribution-id <DIST_ID> --paths "/*"
```

</details>

> 💡 *Why is the bucket private if it's a public website? Because CloudFront is the front door — it has a signed pass (that's the Origin Access Control) to fetch from your bucket. Nobody can bypass your CDN, and you never have to think about "public bucket" horror stories.*

## 5 · When it breaks (all of these happened in our run)

- **The Actions run fails: `Not authorized to perform sts:AssumeRoleWithWebIdentity`.** Two known causes, both fixable in one prompt:
  1. *Brand-new role.* IAM changes take a minute to propagate. Tell Claude: *"rerun the failed workflow."*
  2. *The trust policy uses the old `sub` format.* Every tutorial on the internet says the claim is `repo:owner/name:ref:...` — **GitHub now embeds numeric IDs**, e.g. `repo:owner@123456/name@7891011:ref:refs/heads/main`, and the old format silently never matches. Tell Claude: *"add a temporary workflow step that prints the OIDC token's `iss`/`aud`/`sub` claims, fix the role's trust policy to match exactly, then remove the step."* That's precisely how we found it.
- **The CloudFront URL returns 403 or an XML error.** Either the distribution is still `InProgress` (wait it out) or the bucket policy isn't attached yet — paste the error into Claude Code.
- **Your change is deployed but the browser shows the old page.** That's caching doing its job — hard refresh. If it persists past a minute, ask Claude whether the invalidation step ran.
![image alt](https://github.com/gkoufie1/claude-aws-course/blob/d6fa2d96be146befd61bba51b960e5939bc6bfe0/s3.png)

## 6 · Stretch goal

Make the page actually *you*: real name, a line about what you're building, links to your GitHub. Push and watch it go live. (A custom domain like `yourname.com` is possible — CloudFront supports it with a free certificate — but a domain itself costs ~$12/year, so we park that until after the course.)

## 7 · Cleanup

**Nothing to clean up — this one runs free.** At rest, this whole activity costs approximately nothing: kilobytes in S3 (fractions of a cent), CloudFront's free tier covers 1 TB/month of traffic, Actions is free for public repos, and IAM is always free. Your budget guardrail from Chapter 0 is watching regardless.

Everything is tagged `course=claude-aws`, and the full teardown — bucket, distribution, OAC, role, OIDC provider — happens in [Activity 5](05-capstone-teardown.md).

→ Next: [Activity 2: Serverless API](02-serverless-api.md)
