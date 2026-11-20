# Compromise response runbook #

Operator plays for a compromised plugin, a bad release, or a
compromised catalog. Start at [Play 1](#play-1--triage-and-freeze) and
let the triage answer route you; don't improvise mid-incident.

| Symptom | Play |
| ------- | ---- |
| Plugin repo looks compromised, catalog pins unchanged | [Play 2](#play-2--hold-the-line) |
| A merged pin points at a bad or compromised version | [Play 3](#play-3--roll-back-a-pin) |
| A plugin can no longer be trusted at any version | [Play 4](#play-4--pull-a-plugin) |
| This repository itself is compromised | [Play 5](#play-5--catalog-repository-compromised) |
| Any of the above | finish with [Play 6](#play-6--communicate) |

## How exposure works ##

Three facts decide every play below:

- **Installers run what the pin names, nothing else.** Each catalog
  entry pins `ref` (tag) and `sha` (commit); Claude Code checks out the
  `sha`. Commits pushed to a plugin repository — including a force-moved
  tag — reach no one until a pull request moves the pin here.
- **Updates apply on version *difference*, not increase.** A user's
  copy is replaced whenever the computed version differs from what they
  have, so a rollback ships exactly like a release.
- **Propagation lags.** Auto-update is off by default for this
  marketplace. Users who enabled it pick up catalog changes at session
  start; everyone else waits until they run
  `claude plugin marketplace update felddy` or
  `claude plugin update <plugin>@felddy`. Both the damage and the fix
  move at that speed. A user who pinned the marketplace source to a SHA
  is frozen: unaffected by a bad pin, and unreached by the fix until
  they move their pin.

## Play 1 — Triage and freeze ##

1. **Freeze the catalog.** Merge nothing — especially Renovate pin-bump
   PRs. Close any open pin bump that touches the suspect plugin.
2. **Answer one question: did the catalog move, or only a plugin
   repo?**

   ```console
   git log --oneline -- .claude-plugin/marketplace.json
   ```

   For each entry, compare the pinned `sha` against what the plugin
   repository serves right now:

   ```console
   git ls-remote https://github.com/OWNER/REPO "refs/tags/TAG*"
   ```

   A tag commit that no longer matches the pinned `sha` means the tag
   moved upstream — suspicious, but users are still safe if the pin is
   unchanged.
3. **Snapshot evidence** before touching anything: `git ls-remote`
   output, catalog history, the suspect commits, open PR diffs.
4. **Route:** catalog history clean and pins pointing at reviewed
   commits → [Play 2](#play-2--hold-the-line). A merged pin is bad →
   [Play 3](#play-3--roll-back-a-pin). The plugin itself is
   untrustworthy → [Play 4](#play-4--pull-a-plugin). This repository's
   history, settings, or collaborators look wrong →
   [Play 5](#play-5--catalog-repository-compromised).

## Play 2 — Hold the line ##

A plugin repository is compromised but every catalog pin still names a
reviewed commit. **Users are not exposed**; the pin fetches by commit,
and moved or deleted tags change nothing.

1. Keep the freeze: no pin bumps for that plugin until its repository
   is trusted again.
2. Clean the plugin repository: rotate its credentials, revert or
   remove the malicious commits, move tags back to reviewed commits,
   audit collaborators, deploy keys, Actions secrets, and installed
   apps.
3. Review the window: check whether any pin bump merged while the
   repository was compromised (`git log` from Play 1). If one did,
   switch to [Play 3](#play-3--roll-back-a-pin).
4. Resume only after a clean audit, and verify the next Renovate PR's
   `sha` against the plugin repository by hand before merging.

## Play 3 — Roll back a pin ##

A bad version shipped: a merged catalog change points installers at a
commit that must not run.

1. Revert the catalog commit that moved the pin:

   ```console
   git revert <bad-catalog-commit>
   git push origin main   # via PR if branch protection requires one
   ```

   Rollback works because updates trigger on any version difference —
   users "update" back to the older pinned release.
2. Confirm what the restored pin serves: the `sha` in the reverted
   entry must match the last known-good release.
3. Users on auto-update recover at their next session start. Everyone
   else needs a nudge — go to [Play 6](#play-6--communicate) with the
   exposure window (merge time of the bad pin → merge time of the
   revert) and the recovery command:

   ```console
   claude plugin marketplace update felddy
   ```

4. Treat the bad version as a vulnerability in the plugin repository:
   advisory, yanked release notes, and a fixed release once the
   repository is clean.

## Play 4 — Pull a plugin ##

The plugin can't be trusted at any version and must come off user
machines, not just stop updating.

1. In `.claude-plugin/marketplace.json`:
   - Remove the plugin's entry from `plugins`.
   - Map its name to `null` in the top-level `renames` field. Treat
     `renames` as append-only history — add entries, never edit or
     remove old ones.
   - Set `"forceRemoveDeletedPlugins": true` at the top level.

   ```json
   {
     "renames": { "claudeaux": null },
     "forceRemoveDeletedPlugins": true
   }
   ```

2. Run `claude plugin validate . --strict` — it rejects a `renames`
   chain that doesn't resolve — then merge.
3. What happens on user machines, at each session start after their
   catalog refreshes: Claude Code compares installs against the catalog
   plus `renames`, uninstalls the plugin from the user, project, and
   local scopes, and lists it under **Needs attention** in `/plugin`.
4. Know the gaps: installs driven by managed settings are *not*
   removed — notify those administrators directly. Users who never
   refresh the catalog keep the plugin — [Play 6](#play-6--communicate)
   is what reaches them.
5. After the incident window, decide deliberately whether
   `forceRemoveDeletedPlugins` stays. Leaving it `true` means every
   future entry removal also uninstalls — reasonable policy, but make
   it a choice, not a leftover.

## Play 5 — Catalog repository compromised ##

The trust anchor itself is suspect: unexplained commits, changed
settings, or an attacker with write access.

1. **Lock the account:** rotate the GitHub password, re-verify 2FA
   devices, then revoke and reissue personal access tokens and SSH
   keys (`gh auth status` to see what's live).
2. **Audit the repository:** collaborators, deploy keys, Actions
   secrets, installed GitHub Apps, webhooks, and branch-protection
   changes. Remove anything unexplained.
3. **Record, then restore.** Note the bad commit SHAs for the
   advisory, then restore `main` to the last known-good commit (revert
   PRs, or a force push with protection temporarily lifted).
4. **Audit the blast radius:** every catalog commit and every pin bump
   since the last known-good state. Each bad pin that merged gets
   [Play 3](#play-3--roll-back-a-pin); each plugin that must die gets
   [Play 4](#play-4--pull-a-plugin).
5. **If trust in the catalog is burned,** tell users to remove and
   re-add it — and warn them that removing a marketplace uninstalls
   every plugin installed from it:

   ```console
   claude plugin marketplace remove felddy
   claude plugin marketplace add felddy/claude-plugins
   ```

## Play 6 — Communicate ##

Every play above ends here; propagation lag means silence leaves users
exposed.

- **GitHub Security Advisory** on the affected plugin repository (and
  on this repository if the catalog itself shipped the problem). State
  the affected versions by tag *and* commit SHA, the exposure window,
  and the action required.
- **README banner** at the top of this repository — users who
  investigate come here first.
- **Release notes** on the fixed release, pointing back at the
  advisory.
- Claude drafts; the operator publishes. Advisory publish and close are
  authority actions and stay human.

## Preparation checklist ##

Verify these hold *before* they're needed; each play assumes them.

- [ ] Branch protection on `main`: pull request required, no direct
      pushes.
- [ ] CODEOWNERS covers `.claude-plugin/`.
- [ ] 2FA on the owning account; collaborator list minimal.
- [ ] Renovate pin bumps are reviewed, never auto-merged; the diff to
      review is two lines — `ref` and `sha` — against the plugin
      repository's release.
- [ ] CI runs `claude plugin validate . --strict` on every PR.
- [ ] Catalog entries always pin `ref` and `sha` together; an unpinned
      entry silently re-exposes users to the plugin repository's
      default branch.
- [ ] This runbook is linked from the README and reachable from cold
      memory.
