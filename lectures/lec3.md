# 📌 Lecture 3 — Secure Git: Signed Commits, Secret Scanning, and History Hygiene

---

## 📍 Slide 1 – 🔓 The 296,000 Records Behind a Stray Commit

* 🗓️ **October 2022** — Toyota discloses a 5-year leak: an access key for the **T-Connect** customer service backend was pushed to a **public GitHub repo by a development partner in December 2017**
* 🪜 Nobody noticed. The key sat there for almost five years. **296,019 customer records** (email + management number) accessed
* 🧠 Toyota's response: rotate keys, public apology, and *promise* to scan repos
* 🪜 None of the controls needed were exotic — every one of them lives in this lecture: signed commits (audit trail), pre-commit secret scan (would have caught it pre-push), org-level secret scanning (would have caught it post-push)
* 💸 The whole 5-year exposure could have been prevented by a **30-line `.gitleaks.toml` and a 2-line pre-commit hook**

> 🤔 **Think:** Recall from Lecture 2 that *Repudiation* is the R in STRIDE. If someone pushes a malicious commit using your name, how do you prove it wasn't you? That's what commit signing solves.

---

## 📍 Slide 2 – 🎯 Learning Outcomes

| # | 🎓 Outcome |
|---|-----------|
| 1 | ✅ Explain commit signing as a non-repudiation control, not just a green checkmark |
| 2 | ✅ Compare GPG vs SSH commit signing and choose the right one |
| 3 | ✅ Configure your local Git to sign every commit by default |
| 4 | ✅ Run **gitleaks** as a pre-commit hook and read its findings |
| 5 | ✅ Rewrite history with `git filter-repo` to purge a leaked secret correctly |

---

## 📍 Slide 3 – 🗺️ Where Lecture 3 Sits

```mermaid
graph LR
    L1["🏛️ L1<br/>DevSecOps culture"] --> L3
    L2["🎯 L2<br/>Threat model"] -.identifies.-> L3["🔐 L3<br/>Secure Git (here)"]
    L3 --> L4["🚀 L4<br/>CI/CD"]
    L3 -.feeds.-> L8["🔏 L8<br/>Supply chain"]

    style L3 fill:#FF9800,color:#fff
```

* 🪜 **Building on L2:** STRIDE-R (non-repudiation) and STRIDE-S (spoofing) both have Git-layer controls — this lecture implements them
* 🎯 **Lab 3 alignment:** Task 1 configures SSH commit signing + verifies provenance; Task 2 wires gitleaks into pre-commit; Bonus rewrites history with filter-repo to purge a planted secret

---

## 📍 Slide 4 – 🪪 What Commit Signing Actually Proves

> 💬 *"A commit is an unverified claim about who made a change."* — paraphrasing the Git documentation. Signing turns the claim into evidence.

| 🏷️ Property | ❌ Unsigned commit | ✅ Signed commit |
|---|---|---|
| Author name & email | Trivially forged (`git -c user.name=...`) | Same — strings still trivially settable |
| **But:** can we prove the commit was made by the key-owner? | No | Yes — cryptographic signature |
| Audit trail for incident response | "Some commit claims Alice wrote it" | "Alice's key signed this; either Alice signed it or her key is compromised" |

* 🪜 **Signing protects against repudiation, not against identity confusion.** The display name can still say anything. The signature is the proof
* 🧠 **GitHub shows "Verified" only if** the signing key is uploaded to GitHub under the matching account. Otherwise you get "Unverified" — the commit is signed but GitHub can't tell whose key it is

---

## 📍 Slide 5 – 🪪 GPG vs SSH Commit Signing

* 🪜 Git has supported **GPG signing since 2009**. SSH-key signing was added in **Git 2.34 (November 2021)** and rolled out to GitHub for verification in **August 2022**
* 🆚 Which to use?

| 🏷️ Aspect | 🔐 GPG | 🪪 SSH |
|---|---|---|
| Key infrastructure | Separate keypair, manage with `gpg` | Reuses your existing SSH key |
| Key management | Web-of-trust, keyservers | Your `~/.ssh/` directory + GitHub upload |
| Linux ecosystem | Native everywhere | Native everywhere |
| Hardware token | YubiKey via gpg-agent | YubiKey via ssh-agent (FIDO/U2F) |
| GitHub support | Since 2016 | Since August 2022 |
| Beginner friction | High (key gen + trust setup) | Low (you already have a key) |

* 🪜 **In this course we use SSH signing** for Task 1: zero new infrastructure, same key you push with, GitHub verifies. GPG is fully fine; just heavier
* 🧠 Most cloud-native teams (CNCF projects, the Go core, Kubernetes) moved to SSH signing in 2023–2024

---

## 📍 Slide 6 – 🪛 Configuring SSH Signing in 4 Commands

```bash
# 1. Tell Git to use SSH for signing
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub

# 2. Sign every commit by default
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# 3. Optional but recommended: local verification
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
echo "you@example.com namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" \
  >> ~/.config/git/allowed_signers

# 4. Upload the same .pub to GitHub → Settings → SSH and GPG keys
#    → Add new → choose 'Signing Key' (not 'Authentication Key')
```

* 🧠 The **`Signing Key` vs `Authentication Key`** distinction matters — same key bytes, two different roles. GitHub will show "Verified" only if you uploaded as **Signing**
* 🛠️ Local verification via `allowed_signers` means `git log --show-signature` will print "Good signature" — even offline

---

## 📍 Slide 7 – 🏢 Enforcing Signatures on the Server

* 📜 **Branch protection rules** in GitHub let you set "**Require signed commits**" — unsigned pushes are rejected
* 🪜 **Required for:**
  * `main` / `release/*` (always)
  * `production` branches in regulated environments
* 🪜 **Trade-off:** every contributor has to configure signing. The OWASP Security Champion (Lecture 1) usually owns the rollout
* 🧠 Once enforced, the audit trail in `git log --show-signature` becomes evidence — the compliance team will ask for it during SOC 2 / ISO 27001 review (Lecture 9 covers the frameworks)

> 💬 *"A 'Verified' badge on every commit is the cheapest non-repudiation control a software org can ship."* — paraphrased from the GitHub security engineering blog (2023)

---

## 📍 Slide 8 – 🗝️ How Secrets End Up in Git

* 🚦 The four classic patterns (every one of them is in OWASP Top 10:2025 A03 *Cryptographic Failures*):
  1. **Hardcoded** — `password = "admin123"` in a config file
  2. **Forgotten committed** — `aws-credentials.json` left in repo root
  3. **Pasted into history** — debugging session, never cleaned up
  4. **Leaked into logs** — Cloud Function prints env vars on error

* 📊 **GitGuardian's 2024 State of Secrets Sprawl** counted **23.8 million secrets** leaked across public GitHub in 2023 alone
* 🪜 **The killer property:** Git **never forgets**. Deleting a file in the next commit leaves the secret in `git log -p` forever. This is why rotation > deletion

---

## 📍 Slide 9 – 🔍 The Secret-Scanner Field

```mermaid
graph TB
    SS[🔍 Secret scanners]
    SS --> GL[Gitleaks<br/>Go, regex<br/>MIT, fast]
    SS --> TH[TruffleHog<br/>Go, regex + verify<br/>AGPL-3.0]
    SS --> GS[git-secrets<br/>AWS Labs<br/>shell, basic]
    SS --> GG[GitGuardian<br/>SaaS, 800+ detectors<br/>free tier]
    SS --> GHN[GitHub Secret Scanning<br/>org-native<br/>free for public repos]

    style GL fill:#4CAF50,color:#fff
    style TH fill:#2196F3,color:#fff
    style GHN fill:#9C27B0,color:#fff
```

| 🔍 Tool | 🎯 Best for | 🚦 Catch |
|---|---|---|
| **gitleaks** | Pre-commit hooks (sub-second on most repos) | Pure regex — fast, no network |
| **TruffleHog** | CI/CD scanning + **live verification** (does the secret still work?) | 800+ secret types, AGPL — same family of regex but with verifiers |
| **GitHub Secret Scanning** | Org-wide, post-push, partner-verified secrets (AWS, Stripe, etc.) | Free for public repos; **Advanced Security** for private |
| **git-secrets** (AWS Labs) | Lightweight; AWS-focused; predates the others | Limited rule set; still useful for `aws_access_key_id` style finds |

* 🪜 **In this course:** gitleaks for pre-commit (Lab 3 Task 2) + TruffleHog optional in CI (mentioned, not required)

---

## 📍 Slide 10 – ⚡ Gitleaks in 60 Seconds

```bash
# Install once (Go binary)
brew install gitleaks         # or: docker run zricethezav/gitleaks
gitleaks --version            # course pins v8.x (April 2026 stable)

# Scan working tree (no Git history)
gitleaks dir .

# Scan full history (slower but catches forgotten commits)
gitleaks git .

# Pre-commit hook output (this is what you'll add in Lab 3)
gitleaks protect --staged --redact
```

| 🚨 Subcommand | 🎯 When to use |
|---|---|
| `protect` | Pre-commit — scans staged changes only |
| `dir` | Pre-push — fast directory scan |
| `git` | CI nightly — full history audit |

* 🧠 Gitleaks is **regex + entropy** — it detects high-entropy strings even without a known prefix. False positives are normal; tune via `.gitleaks.toml`
* 🪜 Course pins **gitleaks v8.x** (latest stable as of April 2026)

---

## 📍 Slide 11 – 🪝 Pre-Commit Hooks: The Right Place to Catch It

> 💬 *"The cheapest place to fail is on the developer's laptop, before the push."* — Bruce Schneier's economics applied to Git

* 🪜 **The pre-commit framework** (`pre-commit.com`, Anthony Sottile, 2017) standardizes hooks: declarative `.pre-commit-config.yaml`, language-agnostic
* 🪜 **Why it matters:** without a framework, each developer would `cp .git/hooks/...` manually — fragile, non-shareable

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
```

* 🧠 Pre-commit hooks are **opt-in per dev** by default. The reliable approach: pair pre-commit with **CI re-scan** (Lecture 4) — defense in depth

---

## 📍 Slide 12 – 🧹 When the Secret Already Shipped: History Rewrite

You **must** rewrite history when:
* 🪪 A credential is in Git history (current commits or past)
* ⚠️ You can't just delete the file — `git log -p` still shows it

| 🛠️ Tool | 📅 Era | 🪜 Status |
|---|---|---|
| `git filter-branch` | Built-in, since 2007 | ⚠️ Deprecated since Git 2.22 (2019) — slow, race-prone |
| **`git filter-repo`** | Python, by Elijah Newren (Git maintainer), 2018 | ✅ Current recommended tool |
| **BFG Repo Cleaner** | Java, by Roberto Tyley, 2012 | ✅ Faster for large repos, slightly less flexible |

* 🪜 **Critical**: rewriting history changes commit hashes. Everyone with an existing clone must re-clone or rebase. **Coordinate with the team before pushing.**
* 🚨 **Rotation > rewriting.** The secret has been on the public internet; assume it's been scraped. Rewrite to clean up; **rotate to actually secure.**

---

## 📍 Slide 13 – 🧹 filter-repo in Practice

```bash
# Install
pip install git-filter-repo                   # course pins v2.45+

# Always work on a fresh clone (filter-repo refuses dirty repos)
git clone https://github.com/you/repo.git repo-clean
cd repo-clean

# Replace a leaked AWS key with [REDACTED] across all history
echo 'AKIAIOSFODNN7EXAMPLE==>[REDACTED]' > /tmp/replacements.txt
git filter-repo --replace-text /tmp/replacements.txt

# Push the rewritten history (will force-push automatically)
git remote add origin git@github.com:you/repo.git
git push --force --all
git push --force --tags

# THEN rotate the actual AWS key — the rewrite alone doesn't help
```

* 🧠 The **bonus task** in Lab 3 does exactly this on a small repo with a planted fake key — safe to experiment

---

## 📍 Slide 14 – 🪪 Identity Beyond Commit: Signed Tags & Releases

* 🏷️ Same `gpg.format = ssh` config makes **tags** signable: `git tag -s v1.0.0`
* 📦 Signed tags matter for **release authenticity** — the artifact you sign in Lecture 8 (Cosign) is usually built from a signed tag
* 🪜 **Reproducible signing chain you'll build in this course:**
  * **L3** — sign commits + tags
  * **L4** — CI gates on signed commits
  * **L8** — sign the built artifact (image, SBOM, provenance) with Cosign
  * **L9** — verify signatures at deploy + runtime

* 🪜 Each link is cheap on its own; together they form the supply-chain provenance backbone

---

## 📍 Slide 15 – 🔬 Case Study: Uber's S3 Bucket Keys (2016)

* 🗓️ **October 2016** — attackers find AWS credentials in a **private GitHub repo** belonging to Uber engineers (Uber later paid the attackers $100k under an "bug bounty" agreement that violated breach notification laws)
* 💾 Credentials gave access to an S3 bucket containing **57 million driver + rider records**
* 🧠 The credentials were in a private repo — *but private isn't the same as scanned*. If a single contributor's account is compromised, every secret in every private repo they touched is exfiltrated
* 🪜 **Controls that would have helped:** secret scanning on the private repo, **pre-commit hook** to prevent the original commit, **short-lived IAM credentials** instead of long-lived access keys (Lecture 6 covers this in IaC)

---

## 📍 Slide 16 – 🔬 Case Study: Codecov Bash Uploader (2021)

* 🗓️ **April 15, 2021** — Codecov discloses that attackers altered the `bash` uploader script (downloaded by `curl | bash` in thousands of CI pipelines) to **exfiltrate environment variables**
* 🌍 Affected: HashiCorp, Twilio, Mozilla, Rapid7, Linux Foundation, GoDaddy — anyone uploading coverage via the script
* 🧠 **Two intertwined lessons:**
  * **For Git/secrets (this lecture):** secrets in CI env vars are first-class secrets and need rotation too
  * **For supply chain (Lecture 8):** signing/verification of `bash` scripts pulled into CI matters as much as signing your own artifacts

* 🪜 The fix that mattered: rotate every secret that ever appeared in a Codecov-using CI pipeline. The fix that *should* have been in place: signed `curl | bash` payloads (Cosign blob signing — Lab 8 Bonus)

---

## 📍 Slide 17 – 🧠 Common Mistakes & Fixes

| 🚨 Mistake | 🛠️ Fix |
|---|---|
| `.env` committed once and then `.gitignore`-d | History rewrite + rotate secret. Adding it to `.gitignore` *now* doesn't remove it from history |
| "I'll just delete the commit" | `git reset --hard HEAD~1` doesn't delete remote — and doesn't touch any other branch where the commit landed |
| Pre-commit hook installed, but new dev didn't run `pre-commit install` | Add `pre-commit install` to your repo bootstrap script; CI re-scans as fallback |
| "We have GitHub Secret Scanning, we're safe" | GHSS catches **partner-verified** secrets (AWS, Stripe, etc.). Generic high-entropy strings: gitleaks or TruffleHog |
| Signed commits but unverified branch protection | `Require signed commits` must be set in the branch protection rule — otherwise unsigned commits can still merge |

---

## 📍 Slide 18 – 🪪 Organizational Habits That Scale

* 🧪 **Patterns that work** (from public security engineering blogs at GitHub, Mozilla, Datadog, Shopify):
  1. **Onboarding script** installs pre-commit + configures signing in one command
  2. **`main` branch** has *Require signed commits* + *Require pull request* turned on
  3. **GHSS + a custom pattern rule** for any company-internal secret format (e.g. `MYCO_TOKEN_*`)
  4. **Secret rotation runbook** documented before any incident — when (not if) a leak happens, the team knows what to do at 2am
* 🚫 **Anti-patterns:**
  * "We trust our devs" (the threat model in Lecture 2 includes insider mistakes, not malice)
  * Secret-scanning enabled but no-one reads the alerts (Target 2013 — same failure mode at a different layer)

---

## 📍 Slide 19 – ⏭️ What's Next + Lab Preview

* 🧪 **Lab 3** (this week):
  * Task 1 (6 pts): Configure SSH commit signing; verify with `git log --show-signature` and on GitHub
  * Task 2 (4 pts): Install pre-commit + gitleaks; plant a fake secret; observe the hook stopping the commit
  * Bonus (2 pts): Rewrite history with `git filter-repo` to purge a planted secret across all commits
* 🚀 **Lecture 4** (next week): **CI/CD Security & Build Hardening** — we extend pre-commit hooks into pipeline gates, add SBOM generation, and start treating the pipeline itself as an attackable system

> 💬 *"Trust is built in droplets and lost in bucketloads."* — Kevin Mitnick. Applies to Git history as much as to interpersonal trust.

---

## 📍 Slide 20 – 📚 Resources & Takeaways

**Books:**

| 📖 Book | ✍️ Why |
|---|---|
| *Pro Git* — Scott Chacon & Ben Straub (2nd ed., free PDF) | Chapter 7 covers signing in detail |
| *Application Security Program Handbook* — Derek Fisher (Manning, 2023) | Chapter 4 on secrets management is the most actionable single chapter |
| *The DevOps Handbook* — Kim, Humble, Debois, Willis (2nd ed., 2021) | Part IV on technical practices; secrets as a deployment concern |

**Talks & specs:**

* 🎥 *"How GitHub Builds GitHub"* — GitHub engineering, several talks across 2022–2024; signing rollout discussed
* 🎥 *"Secrets Sprawl: What 23M Secrets Tell Us"* — GitGuardian, AppSec EU 2024
* 📜 [Gitleaks Rule Reference](https://github.com/gitleaks/gitleaks/blob/master/config/gitleaks.toml)
* 📜 [GitHub: About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification)
* 📜 [git-filter-repo manual](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html)

**Takeaways:**

| # | 🧠 Insight |
|---|---|
| 1 | Signed commits are about **non-repudiation**, not just a green checkmark. The signature is evidence the audit team will ask for. |
| 2 | SSH signing is the lowest-friction modern approach. GPG remains valid; pick one and standardize. |
| 3 | Secrets in Git are **forever** — rewriting history is for cleanup; rotation is for actual remediation. |
| 4 | gitleaks pre-commit + GHSS/TruffleHog in CI = layered defense. Either alone has known gaps. |
| 5 | The Toyota leak (5 years, 296k records) needed two controls that fit on one page of YAML. Cheap insurance. |

> 💬 *"The price of liberty is eternal vigilance."* — Thomas Jefferson (sort of) — applies to your `git log -p` as much as to anything else.
