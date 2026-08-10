---
uid: bug-6fc3b0f5
id: BUG-1
type: bug
title: 'xgd init: transient SSL failure on machine-branch push aborts init, leaving
  workspace incomplete'
created_by: xgd
created_at: '2026-08-10T21:02:22.149335+00:00'
updated_at: '2026-08-10T21:05:49.946152+00:00'
completed_at: null
last_field_updated: title
status: draft
fields:
  auto_merge_back: true
  needs_review: false
  priority: medium
---

## Symptom

`xgd init` reported:

```
Registering machine identity...
  ✓ Generated machine identity: b312ec0e
  ✗ Failed to push machine branch xgd-working-b312ec0e: fatal: unable to access
    'https://github.com/TayeWalker/vibe_proj/': SSL certificate problem: self signed certificate

✗ Branch topology setup failed. See messages above.
```

## Root cause

Two separate things:

1. **The trigger was transient, not configuration.** The push of the machine branch hit
   `SSL certificate problem: self signed certificate`. This was NOT a persistent misconfiguration:
   - `git config --list` shows no `http.sslVerify`, `http.sslCAInfo`, or proxy settings.
   - No `HTTPS_PROXY` / `SSL_CERT_FILE` / `NODE_EXTRA_CA_CERTS` env vars are set.
   - The three preceding branch pushes (`main`, `xgd-stable`, `xgd-working`) succeeded seconds earlier.
   - `git ls-remote` and `git push --dry-run origin xgd-working:xgd-working-b312ec0e` both
     succeed now, exit 0.

   Consistent with intermittent TLS interception (corporate proxy / VPN / network filter)
   or a momentary network blip — an environment condition, not an XGD or repo problem.

2. **The real damage is that init is not resumable.** `_setup_machine_identity`
   (`xgd_source/core/workspace.py:604`) returns 1, which propagates through
   `_setup_branches_worktrees_and_remote` to `cmd_init` (`workspace.py:1737`), which returns
   early. Two steps after it never ran:
   - `_create_governance_stubs()` (`workspace.py:1748`) — the Architecture Policy,
     Security Policy, and Interface Design Policy DOC tickets.
   - `_register_project_for_cross_project_refs()` (`workspace.py:1752`).

   And `xgd init` cannot simply be re-run to finish the job: the pre-flight check at
   `workspace.py:1575-1579` hard-fails with `✗ Branch or worktree "xgd-working" already exists.`
   So a transient network error during the last step of init leaves the workspace permanently
   half-initialized with no supported recovery path.

## Verified current state

Healthy (init got this far correctly):
- Branches `main`, `xgd-working`, `xgd-stable` exist locally and on origin.
- Worktrees for `main` and `xgd-stable` provisioned under `~/.xgd/worktrees/`.
- `.xgd/machine_id` (`b312ec0e`) and `.xgd/machines.yaml` written and committed (`16af57a`).
- Merge driver `xgd-ticket-recent` registered; `gh` authenticated as TayeWalker.
- `unique_counters` branch and local `main` being one ticket-commit ahead of `origin/main`
  are both normal XGD bookkeeping, not corruption.

Missing:
- Remote branch `xgd-working-b312ec0e` — absent from `git ls-remote`.
- All three governance DOC tickets — `xgd ticket list --type doc` returns none.

Unrelated non-issue: the bare `255` printed by `xgd dashboard start` is the dashboard PID
(confirmed against `.xgd/logs/dashboard_postmortem.json`), not an exit code. The dashboard
is running normally.

## Fix

Manual completion of the two skipped steps:

1. `git push origin xgd-working:xgd-working-b312ec0e`
2. Create the three governance DOC stubs, matching `_create_governance_stubs`
   (`type=doc`, empty body, `fields.doc_kind` in
   `architecture_policy` / `security_policy` / `interface_design_policy`).

Both target steps are idempotent by design, so this converges on the same state a clean
`xgd init` would have produced.

## Upstream follow-up (XGD tool, not this project)

`xgd init` should either retry/soft-fail the machine-branch push (it is the last and least
critical step) or support resuming an interrupted init rather than refusing on
already-existing branches. Filed here for the record; the fix belongs in the xgd package.
