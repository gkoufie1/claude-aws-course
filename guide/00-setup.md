# Chapter 0: Setup — one install, and Claude does the rest

**Time: ~30 minutes · Cost: $0**

Most cloud courses lose you here, in a maze of installers and config files. Not this one. You will manually install exactly **one** program — Claude Code. Then you'll tell Claude to install and configure everything else.

## What you need before starting

| Account | Cost | Why |
|---|---|---|
| [Claude](https://claude.ai) | Pro ($20/mo) or Max | Powers Claude Code — this is the "one thing" |
| [GitHub](https://github.com) | Free | Where your projects will live |
| [AWS](https://aws.amazon.com) | Free tier* | Where your services will run |

*AWS requires a credit card at signup even for free tier. Everything in this course fits in free tier, and in Step 4 we install a budget alarm that emails you long before anything could cost real money. Expected total course cost: **$0–2**.

## Step 1: Install Claude Code (the only manual install)

**Mac or Linux** — open Terminal (Mac: press `Cmd+Space`, type "Terminal", hit Enter) and paste:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows** — open PowerShell (Start menu → type "PowerShell") and paste:

```powershell
irm https://claude.ai/install.ps1 | iex
```

Then start it:

```bash
claude
```

A browser window opens → log in with your Claude account → you're in. No API keys, no config files.

> ✅ **Checkpoint:** your terminal shows the Claude Code welcome screen with a `>` prompt waiting for you.

## Step 2: Let Claude install the rest

From here on, everything in a quote box is a **prompt you paste into Claude Code** — not a command you run yourself. This is the whole method of the course: you direct, Claude executes.

> Check whether I have the GitHub CLI (`gh`) and AWS CLI (`aws`) installed. Install whichever are missing for my operating system (use Homebrew on Mac, winget on Windows, apt on Linux — and install the package manager first if I don't have it). If the AWS CLI is installed but older than version 2.32, upgrade it — this course uses the `aws login` command, which older versions don't have. When done, run `gh --version` and `aws --version` and show me the output so I can confirm both work.

Claude will detect your OS, install what's missing, and verify. This takes 2–5 minutes.

> 💡 **Already have some of these?** Great — Claude will detect that and skip them. "Already installed" counts as passing this step — with one exception: an AWS CLI older than 2.32 needs the upgrade, and the prompt above tells Claude to handle that too.

> 💡 **You may be asked for a password mid-install.** That's your *computer's* password (the one you log into your Mac/PC with) — installers need it. It's normal, and Claude never sees it: you type it directly into the system prompt.

> ✅ **Checkpoint:** you see version numbers for both `gh` and `aws`.

## Step 3: Connect GitHub (2 minutes)

> Run `gh auth login` and walk me through logging into GitHub. I want GitHub.com, HTTPS, and logging in through the browser.
> ![image alt](https://github.com/gkoufie1/claude-aws-course/blob/42f1405d87b1a4342c49e70ca2500c017ea785b4/gh%20auth%20login1.png)

The GitHub CLI shows a one-time code, opens your browser, you paste the code, done. No tokens to manage.

> ✅ **Checkpoint:** paste this prompt — *"Run `gh auth status` and tell me if I'm logged in."* — and Claude confirms your username.

> 💡 **Already a GitHub user?** `gh auth status` may show more than one account. The one marked `Active account: true` is the one this course will use — if that's not the account you want your projects on, tell Claude: *"switch my active gh account to \<username\>."*
> ![image alt](https://github.com/gkoufie1/claude-aws-course/blob/1c2f511a322ffe1b292b40badda4468b90428361/gh%20auth%20yes.png)

## Step 4: Connect AWS (the careful one — 10 minutes)

This is the only genuinely fiddly part of the whole course, so we go slow.

### 4a. Create your AWS account

Go to [aws.amazon.com](https://aws.amazon.com) → "Create an AWS Account". Choose the free **Basic support** plan. Yes, it asks for a card — see the note at the top of this chapter, and keep going: the guardrail comes in step 4d.

### 4b. Create your working user

Don't do day-to-day work as your root account (the one you just signed up with) — that's the master key to everything, and AWS itself warns against it. Instead, make a working user with its own console password:

1. In the AWS Console, search **IAM** → **Users** → **Create user**
![image alt](https://github.com/gkoufie1/claude-aws-course/blob/227f31f8273bd69a6fa0eb042b6a47c9d7587d68/IAM.png)
![image alt](https://github.com/gkoufie1/claude-aws-course/blob/82ff79d1bde4f6c136ad03792bd7b9ea246e9c25/create%20user.png)
3. Name: `claude-course`
![image alt](https://github.com/gkoufie1/claude-aws-course/blob/57946aeb65b4ef3694766e9ac0b90aeaac86b683/iam%20name.png)
5. Check **Provide user access to the AWS Management Console**. AWS will recommend Identity Center — for a personal account, pick **I want to create an IAM user** instead
6. Choose **Custom password** and set one, then uncheck **User must create a new password at next sign-in** (you're the only one using it) → Next
7. **Attach policies directly** → check **AdministratorAccess** → Next → **Create user**
![image alt](https://github.com/gkoufie1/claude-aws-course/blob/0d5aad6d11719ddff197265fd7aaa65492384ff6/select%20permissions.png)
9. On the success screen, copy the **Console sign-in URL** (it contains your 12-digit account number). Save **three things** in your password manager: that URL, the username `claude-course`, and the password you just set

> 💡 **You now have two AWS logins — don't mix them up.**
> - **Root** = the *email address* + password from 4a. Lock it away. You'll use it maybe twice a year.
> - **`claude-course`** = a *username* + password + your account's sign-in URL. This is your everyday login for the whole course.
>
> Quick tell: if an AWS sign-in page asks for an **email address**, it's the root form — not the one you want.

> 💡 *Notice what we **didn't** create: access keys. Older tutorials have you download a file of permanent secret keys — a file that works forever and that attackers love to find. The modern CLI signs in through your browser instead, with temporary credentials that expire on their own. Nothing to download, nothing to leak.*

> 💡 *Sidebar: real companies scope permissions way down from AdministratorAccess. For a personal learning account it's fine — and we delete this entire user in Activity 5.*

### 4c. Sign in — browser first, then CLI

Here's the mental model for this step: the `aws login` command doesn't ask for a password. It looks at **whoever your browser is signed in as** and hands the CLI that identity. So the whole trick is getting your browser signed in as `claude-course` *before* running it.

**Part 1 — in your browser:**

1. **Sign out of the AWS Console.** You're currently signed in as root (that's who created the user). Click your account name in the **top-right corner** → **Sign out**.
2. **Open the console sign-in URL you saved in 4b.** Because that URL contains your account number, AWS skips straight to the right form: **IAM user sign-in**, with the Account ID already filled in.
   - Wrong page? If it asks for an **email address**, that's the root form — look for **"Sign in to a different account"** or an **IAM user** option.
3. Enter username `claude-course` and the password from 4b → **Sign in**.
4. If AWS asks you to **choose a new password**, that's normal for a first sign-in — set one and update your password manager.

> ✅ **Checkpoint:** the AWS Console loads, and the **top-right corner** says `claude-course @ <your account number>`. That corner label is always how you check *who* you are in AWS.

**Part 2 — back in Claude Code**, paste:

> Set my AWS CLI default region to `us-east-1` and output format to `json`. Then run `aws login` — it opens my browser and waits for me to approve. When it finishes, run `aws sts get-caller-identity` and tell me who I'm signed in as. If the ARN ends in `:root`, warn me instead of continuing.

Your browser opens an AWS approval page → click **Allow** → the terminal fills in the rest. The CLI now holds temporary credentials that refresh themselves while you work. (The region gets set *before* the login on purpose — `aws login` refuses to start without one.)

> 🔒 **Security habit, learned early:** notice that no secret ever touched the chat window — and none ever should. Your password went into AWS's own sign-in page; the CLI got its credentials through the browser handshake. If a tutorial ever asks you to paste a secret key into an AI chat, close the tab.

> ✅ **Checkpoint:** `get-caller-identity` returns your account number and an ARN ending in `user/claude-course`.

**If something's off, it's one of these:**

- **ARN ends in `:root`** — your browser was still signed in as root when `aws login` ran. Sign out of the console, sign in as `claude-course`, then tell Claude: *"run `aws login` again — it will ask whether to overwrite the old root session; answer yes for me."* (We hit both of these ourselves in the dry-run of this chapter.)
- **`Invalid choice: 'login'`** — your AWS CLI is too old for `aws login`. Tell Claude: *"upgrade my AWS CLI, then retry the login."*
- **`Unable to locate credentials`** — the login never completed (browser closed too soon, or this step was skipped). Paste the prompt again.
- **An `ExpiredToken` error hours or days later** — sessions expire; that's them working as designed. Run `aws login` again and re-approve.

### 4d. Arm the budget guardrail

You cloned this course repo (or will in Activity 1), which means the `aws-budget-guard` skill is already available. Paste:

> Use the aws-budget-guard skill to set up my spending guardrails. Use a $5 monthly budget and my email address.

Claude creates a $5/month budget with alerts at 50%, 80%, and 100%. If a forgotten resource ever starts costing money, AWS emails you at **$2.50** — long before it matters.

> ✅ **Checkpoint:** Claude confirms the budget exists (and AWS sends a confirmation email).

## Step 5: The full pre-flight check

One last paste:

> Run a setup check: `claude --version`, `gh auth status`, and `aws sts get-caller-identity`. Tell me in plain English whether all three are ready — and confirm AWS shows the `claude-course` user, not root. If anything failed, help me fix it.

All green? **You're ready. Everything from here on is building.**

→ Next: [Activity 1: A live website in 20 minutes](01-static-site.md)

---

### Troubleshooting

One rule, for this chapter and every chapter after it: **paste the error into Claude Code and say "fix this."** Describe what you expected and what happened. That's not a workaround — learning to debug by directing Claude *is the course*.
