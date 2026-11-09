# Viktor's OpenClaw distribution

This fork maintains automatic quota resumption independently of whether upstream
accepts [PR #195884](https://github.com/openclaw/openclaw/pull/195884).

## Branches and initial baseline

- Distribution: `viktorjonzon/openclaw`, branch `dist/viktor`.
- Upstream contribution: `feat/quota-auto-resume-main`. Keep distribution-only
  changes off this PR branch.
- Initial upstream base: `94714fdce8e` (development main, package version
  `2026.11.1`), plus feature commits `7f53d22b0f1` and `3e0b8808350`.
- This is a maintained source branch, **not an installed or release-qualified
  binary**. Creating it does not change a running Gateway.
- The older `2026.10.2` quota-resume tarball is a separate build. It does not
  include the later main-branch hardening and is not an artifact of this branch.

## Update policy

Never reset this branch to upstream or use GitHub's discard/sync operation on it.
Merge upstream changes on a candidate branch; retain our patches and tests.
Prefer reviewed release cuts for future installation targets. A release tag on
an older release line does not turn this development baseline into a stable
release: review ancestry and migrations before choosing a target.

Updates are operator-driven, not scheduled. No unattended merge, registry
publication, or installation is configured. Failed checks or conflicts leave
the accepted distribution branch and running installation unchanged.

### 1. Prepare an isolated candidate

In a clone with `fork` pointing to `viktorjonzon/openclaw` and `origin` pointing to
`openclaw/openclaw`, choose a reviewed upstream tag or full commit SHA:

```bash
git fetch fork dist/viktor
git fetch origin <reviewed-upstream-ref>
git rev-parse FETCH_HEAD
git worktree add -b dist/candidate-<id> ../openclaw-candidate-<id> fork/dist/viktor
cd ../openclaw-candidate-<id>
git merge --no-ff <resolved-upstream-sha>
```

Resolve conflicts explicitly. Never resolve conflicts by dropping quota resume
or weakening its tests. If upstream implements an equivalent feature, compare
behavior and migration before retiring the local implementation. If integration
cannot be completed, preserve the candidate for repair; do not promote it.

### 2. Validate the exact candidate

Run as a normal user, using the Node and pnpm versions required by the candidate.
Follow its `AGENTS.md` instructions and current test runner configuration. The
initial baseline uses these commands (run tests serially):

```bash
pnpm install --frozen-lockfile
pnpm tsgo:core
pnpm tsgo:core:test
pnpm test src/agents/quota-resume-policy.test.ts src/gateway/server-quota-resume-runtime.test.ts src/gateway/server-runtime-services.test.ts src/agents/session-suspension.test.ts
pnpm build
```

Also run checks for any other changed integration surface and the repository's
required review. Inspect the final diff. Reject failures and missing/empty test
runs; do not use `--passWithNoTests`. Commit all candidate fixes before packaging.

For deployment qualification, use an isolated Gateway/state directory to verify
quota recovery, restart persistence, stop/new-user-turn cancellation and absence
of duplicate launches. Test migration from the currently installed version with
a private copy of its state. Unit tests alone do not establish live readiness.

### 3. Package, then promote

Package only a clean, validated commit. Choose an output directory outside the
checkout and include the full candidate SHA in the filename:

```bash
git status --short
git rev-parse HEAD
node scripts/package-openclaw-for-docker.mjs --skip-build --allow-unreleased-changelog --output-dir <artifact-directory> --output-name openclaw-viktor-<candidate-sha>.tgz
sha256sum <artifact-directory>/openclaw-viktor-<candidate-sha>.tgz
```

`--skip-build` is valid only immediately after a successful build of that same
source. Keep the artifact, checksum and validation log together, recording the
full source SHA, upstream base, Node/pnpm versions and any remaining limitations.
Keep the last working artifact and its verified full-state backup for recovery.

After successful qualification, promote with a normal, non-forced push:

```bash
git push fork HEAD:refs/heads/dist/viktor
git ls-remote fork refs/heads/dist/viktor
```

Compare the remote SHA with the validated local SHA. If another update moved
the distribution branch, the push must fail; integrate and revalidate instead of
force-pushing. Branch protection prevents force-pushes and deletion; validation
and promotion remain an operator responsibility, not a hosted CI enforcement.

### 4. Install through the supported updater

Before the first deployment, create and verify a full-state backup and disable
registry auto-installation using the installed version's supported configuration
(`update.auto.enabled: false` where supported). Do not disable ordinary security
monitoring. Inspect any external update jobs as well.

From an operator terminal outside the Gateway, use the supported updater with
the exact validated local artifact, first inspecting its plan:

```bash
openclaw update --tag <absolute-artifact-path> --dry-run
openclaw update --tag <absolute-artifact-path>
```

Verify service health, the installed artifact identity and actual quota resume.
An unchanged `openclaw --version` is not proof: fork and upstream packages can
share a version. Never use unqualified `/update`, registry `openclaw update`, or
`npm install -g openclaw` for this deployment; they may replace the fork's code.
An intentional operator registry update can still replace it; this policy is
not a CLI-level lock against that action.

Rollback must follow the candidate's supported recovery documentation. Restore
the matching state backup if migrations make an older binary incompatible; do
not assume reinstalling an old tarball alone is safe.

## Ownership

Viktor owns acceptance and distribution of this branch. Upstream maintainers
control only upstream acceptance. The upstream PR can remain open independently.
Maintaining this branch does not make future upstream changes conflict-free or
remove the need to apply security fixes.
