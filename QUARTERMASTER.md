# Quartermaster — deterministic pre-commit guard for the fleet

A Macie-*like* gate for git repositories, with one deliberate difference:
**no machine learning, no confidence scores, no guessing.** AWS Macie
answers "probably PII (87% confident)". Quartermaster answers only what it
can prove by shape: *this exact character sequence matches rule X, which is
defined as Y, and here is the fixed explanation and remediation.* Every
refusal names the file, the line, the rule id, and the reason.

The name is the job description: the quartermaster inspects every crate
before it goes aboard the ship.

## Enforcement matrix — what runs even if nobody remembers

| layer | trigger | can it be forgotten? |
|---|---|---|
| `.githooks/pre-commit` | every local commit | one-time per clone: `git config core.hooksPath .githooks` |
| `quartermaster.yml` (CI) | every push and PR | **no** — it ships in the repo, runs on GitHub's side |
| Quartermaster step in `lane-promoter.yml` | every blue → master promotion | **no** — wired into the promotion you already require |
| branch protection (recommended, one-time) | even direct pushes to master | **no** — see below |

AGENTS.md is the *awareness* layer, not enforcement: it is how AI
collaborators learn to run the scan before staging anything, and that
bypassing is forbidden. Enforcement is code. The briefing is `AGENTS-MD-SECTION.md`.

**Recommended one-time hardening** (Settings → Branches → protection rule
for `master` → Require status checks): select the `scan` job from the
quartermaster workflow. After that, nothing reaches master without a green
Quartermaster run — not even a direct push that skips blue.

Because the lane step fails the `promote` job, a Quartermaster refusal
**also** triggers the `report-failure` job: an issue opens, you get
notified, and your AI collaborator sees it on the next sync via
`gh issue list --label lane-refusal`. The falling tree has ears.

## Install (once per repo, ~2 minutes)

```bash
# from the repo root
cp -R <this folder>/tools/pii tools/pii
cp <this folder>/.githooks .githooks
cp <this folder>/.github/workflows/quartermaster.yml .github/workflows/
cp <this folder>/.pii-allowlist.json .
git config core.hooksPath .githooks          # activate the local hook
```

Then:

1. **AGENTS.md** — append the section from `AGENTS-MD-SECTION.md` (and
   `CLAUDE.md` too, if you keep one). Without this, AI collaborators
   write blind.
2. **lane-promoter** — paste the step from `LANE_PROMOTER_PATCH.yml` into
   the promote job, before promotion.
3. **baseline** — `python3 tools/pii/scan_pii.py --all`, fix or allowlist
   what it finds.
4. Commit, push to blue, and watch the gate bootstrap through the lane.

## Rules

### BLOCK — stops the commit / fails CI

| rule id | matches | why it is absolute |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | 20-char identifiers beginning AKIA/ASIA/ABIA/ACCA | grants AWS API access when paired with its secret |
| `AWS_SECRET_KEY` | `aws…secret…` field assigned a 40-char base64-style value | the secret itself, not a reference |
| `GITHUB_TOKEN` | `ghp_/gho_/ghu_/ghs_/ghr_`, `github_pat_` tokens | repo/account access until revoked |
| `SERVICE_API_KEY` | `sk-` + 20+ chars (OpenAI/Anthropic shape) | spends the account it belongs to |
| `GOOGLE_API_KEY` | 39 chars beginning `AIza` | bills your GCP project |
| `SLACK_TOKEN` | `xox[baprs]-` tokens | reads/posts to the workspace |
| `JWT` | `eyJ…` three-segment tokens | a signed, unexpired credential |
| `PRIVATE_KEY` | `-----BEGIN … PRIVATE KEY-----` | the credential material itself |
| `SSN` | strict 3-2-4 US SSN shape, invalid areas excluded | highest-value identity-theft target |
| `CREDIT_CARD` | 13–19 digits passing the Luhn checksum | the defining property of a card number |
| `KEY_VALUE_CREDENTIAL` | field named api key/secret/password/token + 16+ char opaque value | the name says what it is |

### WARN — reported loudly, does not stop the flow

| rule id | matches | escape hatch |
|---|---|---|
| `EMAIL` | any email address | allowlist intentional public contacts |
| `PHONE_US` | North-American phone shape | allowlist intentional contact numbers |
| `DOB` | labeled date of birth | no intentional-public case; remove |
| `STREET_ADDRESS` | house number + street type | allowlist business addresses |
| `IP_ADDRESS` | IPv4 literals | use doc-reserved ranges (192.0.2.x etc.) |
| `BEARER_TOKEN` | `Bearer <20+ chars>` | placeholders in tutorials; allowlist fakes |

Severity is per-rule and permanent — the same rule always yields the same
verdict. That is the no-guessing guarantee.

## The allowlist (allowances are documented, never silent)

`.pii-allowlist.json` — exact-value matching only, every entry **must**
carry a `reason`:

```json
{
  "entries": [
    {
      "value": "ops@example.com",
      "rule": "EMAIL",
      "reason": "EXAMPLE — replace: an address you deliberately publish as a public contact."
    }
  ]
}
```

- Missing `reason` → the gate **exits 2 and refuses to run**. A silent
  allowance would be guessing by another name.
- Allowlisted matches still print, marked `allow` with their reason —
  the decision stays visible in every run.

## Baseline (first run on an existing repo)

```bash
python3 tools/pii/scan_pii.py --all          # whole tree; fix or allowlist what it finds
```

Note: this scans the **working tree**. Content already in git **history**
was public the moment it was pushed — Quartermaster prevents new leaks; it
cannot un-leak old ones. For history: `git filter-repo` + rotate every
credential involved.

## Private/sealed repos

```bash
python3 tools/pii/scan_pii.py --staged --private
```

`--private` suppresses the identity WARNs (email/phone/DOB/address/IP) but
keeps every credential rule at BLOCK. Secrets do not ride even sealed repos.
Add `--private` to the hook line in `.githooks/pre-commit` for sealed repos.

## Self-test (deterministic, repeatable)

```bash
printf 'key = AKIAIOSFODNN7EXAMPLE\n' > /tmp/t.txt
python3 tools/pii/scan_pii.py /tmp/t.txt; echo "exit=$?"   # exit=1, BLOCK
```

Exit codes: `0` clean/allowlisted · `1` BLOCK findings · `2` config error
(including an allowlist entry missing its reason).
