# AGENTS.md — drop-in section for Quartermaster
#
# Paste everything below the line into the file your AI collaborators
# actually read: AGENTS.md at the repo root (and CLAUDE.md, if you keep
# one — same text). This is the AWARENESS layer: the agents learn the gate
# exists, run it before staging, and never bypass it. Enforcement itself
# does NOT depend on this file — the hook and the two CI layers run
# regardless. If the agents can't see the gate, though, they will generate
# content that trips it. This is how they see it.

───────────────────────────────────────────────────────────────────────────────

## Quartermaster — mandatory PII gate

Every change that lands in this repository passes Quartermaster
(`tools/pii/scan_pii.py`): a deterministic PII/secret gate. No ML, no
confidence scores, no guessing — every BLOCK cites a rule id, file:line,
and a fixed written explanation.

Run before committing anything (AI collaborators included):

    python3 tools/pii/scan_pii.py --staged

Run before pushing a branch:

    python3 tools/pii/scan_pii.py --git-diff "origin/master...HEAD"

Hard rules for every agent and human:

1. Never bypass. `--no-verify`, weakening a rule to erase a finding, or
   splitting a commit to dodge the diff window are all forbidden.
2. If BLOCKED: read the rule's `why:` and `fix:` lines, then fix the
   content. Do not argue with the gate; fix the cargo.
3. If a finding is intentional (e.g. the public contact email), add the
   exact value to `.pii-allowlist.json` WITH a reason. Entries without a
   reason make the gate refuse to run (exit 2). That is by design.
4. New allowlist entries are published decisions, not conveniences —
   prefer removing the data over documenting its release.
5. When generating content for this repo (READMEs, manifests, code
   examples), generate it clean: no real keys, no third-party personal
   data, no example values that match BLOCK rules.

Exit codes: 0 = clean or allowlisted · 1 = BLOCK, commit/push must not
proceed · 2 = config error, fix `.pii-allowlist.json` first.

Enforcement (independent of this file): the pre-commit hook in
`.githooks/` (enable once per clone with
`git config core.hooksPath .githooks`), `quartermaster.yml` on every
push/PR, and the Quartermaster step inside `lane-promoter.yml`, which
refuses the promotion. A CI refusal also opens a `lane-refusal` issue
automatically.

Full contract: `QUARTERMASTER.md`
