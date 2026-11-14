# AGENTS.md

## Quick Commands

```bash
gh infra validate github/                  # Validate manifests (what CI runs)
gh infra plan github/                      # Dry-run: show what would change
gh infra plan github/ --diff               # Dry-run with inline file diffs
gh infra apply github/ --auto-approve      # Apply (auto-runs on push to main)
gh infra apply github/ --force-secrets     # Re-apply secrets (values can't be diffed)
./actionlint <file.yml>                    # Lint a specific workflow template
```

> **Local gh-infra binary:** Build the same commit CI builds (pinned in `.gh-infra-ref`). The local checkout sits on `main`, so check out the SHA explicitly:
> `cd ~/Development/gh-infra && git fetch origin && git checkout "$(cat ~/Development/github-config/.gh-infra-ref)" && go build -o gh-infra ./cmd/gh-infra/`

## Managed Repos

| Repo | Visibility | Tier | FileSet overrides |
|------|-----------|------|------------------|
| ai-agent-rules | public | Full (CI+Deps+Release+Publish) | — |
| claude-code-status-line | public | Full (CI+Deps+Release+Publish) | — |
| pagerduty-mcp-server | public | Full (CI+Deps+Release+Publish) | — |
| JamBot | public | CI (CI+Deps+Justfile) | — |
| github-config | public | Self (CI self-managed; source only) | — |
| SNORE | public | CI+Deps+Justfile+Hooks+Release | `vars: web: "ui", web_pm: "pnpm"` on ci.yml + Justfile; `e2e: "true"` on ci.yml; `manual_only: "true"` on release.yml |
| shell-configs | public | CI+Deps+Justfile+Hooks+Release | `vars: shell: "true"` on ci.yml + Justfile; `e2e: "true"` on ci.yml |
| meowdb | public | CI+Deps+Justfile+Hooks+Release | — |
| recall | private | CI (CI+Deps+Justfile) | `vars: git_identity: "true"` on ci.yml |
| homelabconfigs | private | Deps (CI + Justfile self-managed) | — |
| syncify | private | CI+Deps+Hooks (Justfile self-managed) | `vars: web: "ui/web", web_pm: "npm"` on ci.yml |
| envsync | private | CI (CI+Deps+Justfile+Hooks) | `vars: system_packages: "libsqlcipher-dev"` on ci.yml |
| BOOTLEG | private | CI (CI+Deps+Justfile+Hooks) | — |
| chartright | private | Deps (CI + Justfile self-managed) | — |
| medical | private | Settings only (no managed files; Actions disabled) | — |
| financial | private | Settings only (no managed files; Actions disabled) | — |

## Project Structure

```
github/
  files-all.yaml       # renovate.json + auto-approve.yml + pull_request_template.md → 13 repos
                       # (all but github-config, medical, financial)
  files-ci.yaml        # ci.yml → 11 repos (github-config, homelabconfigs, chartright self-manage CI)
  files-full.yaml      # publish.yml → 3 PyPI repos
  files-hooks.yaml     # .hooks/pre-commit → 11 repos (github-config, homelabconfigs, chartright excluded)
  files-issues.yaml    # .github/ISSUE_TEMPLATE/ issue forms → 7 public repos (all but github-config)
  files-justfile.yaml  # Justfile → 10 repos (github-config, homelabconfigs, syncify, chartright excluded)
  files-release.yaml   # release.yml + release-please-config.json → 6 release-tier repos
  repos-public.yaml    # RepositorySet: 8 public repos. Defaults = shared settings + public profile
                       # (auto-merge, rulesets, fork PR approval, release-app vars/secret);
                       # the e2e ruleset is a per-entry YAML anchor
  repos-private.yaml   # RepositorySet: 8 private repos. Defaults = shared settings + private profile
  templates/           # ci-python.yml, Justfile, auto-approve.yml, publish.yml, pre-commit-hook,
                       # release.yml, release-please-config.json, renovate.json,
                       # pull_request_template.md, ISSUE_TEMPLATE/ (bug_report.yml,
                       # feature_request.yml, config.yml)
renovate-config/
  default.json         # Shared Renovate preset extended by all managed repos
.github/workflows/     # ci.yml, infra-plan.yml, infra-apply.yml, infra-drift.yml
```

## Key Patterns

**FileSet vs RepositorySet:** `files-*.yaml` distributes template files to repos (`via: push` = direct commit, no PR). `repos-public.yaml` and `repos-private.yaml` manage repo settings. Per-repo overrides use `vars:` (template variables) or `source:` (different file entirely).
**Public and private RepositorySets** — a repo's visibility is decided by which file its entry lives in. Each file's `defaults.spec` starts with an identical shared block (`archived`, `features`, `release_immutability`, `labels`) followed by its visibility profile (`visibility`, `merge_strategy`, `security`, `actions`, and for public repos `rulesets`, `variables`, `secrets`). `repos-public.yaml` sets `visibility: public` plus the public-only settings (auto-merge, private vulnerability reporting, workflow write permissions, fork PR approval, the `main` ruleset, release-app variables/secret); `repos-private.yaml` sets `visibility: private` and the private equivalents. YAML anchors can't cross files, so a change to a shared setting, including the fields the two profiles' `merge_strategy`/`security`/`actions` blocks have in common, must be made in both files. An entry's `spec` merges over its file's defaults by key (a same-named ruleset replaces the whole ruleset). Consequences:
- **New public repo** — add the entry to `repos-public.yaml`. gh-infra's create path sets visibility, merge strategy (incl. auto-merge), security, and Actions settings (incl. fork PR approval); it skips rulesets, labels, variables, and secrets, which land on the next apply: next push to `main`, `gh workflow run infra-apply.yml`, or a local `gh infra apply github/`.
- **Private → public** — move the entry to `repos-public.yaml`. One apply: gh-infra applies the visibility change before the ruleset, Actions, and secret changes in the same run.
- **Public → private** — move the entry to `repos-private.yaml`. One apply, no GitHub UI step: the desired spec no longer carries `fork_pr_approval`, so validation passes. The plan flips visibility, workflow permissions, and private vulnerability reporting; gh-infra leaves unmanaged settings in place, so the `main` ruleset, `allow_auto_merge`, and the release-app variables/secret stay on the repo (inert on GitHub Free) until removed by hand.
- **Move, never copy** — neither `validate` nor `plan` rejects a repo listed in both files; one entry silently wins and the plan shows only a visibility flip.

**Secrets** — secret values may only reference `${ENV_*}` variables; `apply` errors on non-prefixed refs and on unset/empty `ENV_*` vars (`plan` skips resolution). `infra-apply.yml` exports `ENV_RELEASE_APP_PRIVATE_KEY`. Every local `gh infra apply` needs it exported first — secret refs are resolved across ALL repos, even with `-r <private repo>`. `plan` and `validate` don't need it.

**FileSet names** — every `files-<x>.yaml` sets `metadata.name: <x>`. Without a name, gh-infra identifies a FileSet by owner + sorted repo list, which `files-ci.yaml` and `files-hooks.yaml` would share. That identity labels the FileSet in plan output and names the default `via: pull_request` branch (`gh-infra/sync-<owner>-<name>`); with [gh-infra#203](https://github.com/babarot/gh-infra/pull/203), `validate` rejects two `pull_request` FileSets that would share that branch on a repo. gh-infra builds without #203 commit one colliding FileSet's changes under the other's `commit_message`.

**FileSet commit messages** — `commit_message` is a `<% %>` template with `.Repo` and `.Source.URL` (from `GH_INFRA_SOURCE_URL`, set in `infra-apply.yml` to the triggering commit). Every FileSet appends a `Source:` link; the blank separator line lives inside the `<% if .Source.URL %>` guard so local applies don't leave trailing blank lines. `.Vars` is not available in commit messages.

**Go template syntax — always use `index`, never direct field access:**
```
# ✅ Safe — returns "" for missing key (missingkey=error is active)
<%- if (index .Vars "shell") %>

# ❌ Panics at render time if the key is absent
<%- if .Vars.shell %>
```

**Managed-file header** — every distributed template must start with:
```yaml
# This file is managed by github-config. Do not edit manually.
```
Exception: `pull_request_template.md` has no header. `squash_merge_commit_message: PR_BODY` makes the PR body the squash commit body, so any header or comment would land in every commit on `main`.

**Self-managed CI** — repos with non-standard CI requirements (multi-component stacks, non-Python toolchains) own their `.github/workflows/ci.yml` directly. github-config manages only shared configs (`renovate.json`, `auto-approve.yml`, the PR template) for these repos via `files-all.yaml`. Currently self-managed: github-config (Go-based gh-infra), homelabconfigs (Terraform + Ansible), chartright (pnpm/TypeScript monorepo). **Public self-managed repos need a per-repo `rulesets` override in their `repos-public.yaml` entry** unless their CI emits the contexts the default ruleset requires (`checks`). github-config's CI emits `checks`, so it needs no override.

**E2E testing** — the `e2e` job in `ci-python.yml` is opt-in via `vars: e2e: "true"` in `files-ci.yaml`. Currently `ai-agent-rules`, `SNORE`, and `shell-configs` opt in. The default `main` ruleset requires only the `checks` context; each opted-in repo has a per-repo `rulesets` override in `repos-public.yaml` that additionally requires `e2e` (defined once as `&main_e2e_ruleset` on `ai-agent-rules`, referenced as `*main_e2e_ruleset` elsewhere). The managed `Justfile` keeps e2e out of the fast run (`test: uv run pytest -m "not e2e"`) and runs them separately (`test-e2e`; `test-all` runs everything). Repos without e2e-marked tests pass trivially (the recipe tolerates pytest exit code 5 = no tests collected). To add e2e tests to a repo: create `tests/e2e/`, mark tests with `@pytest.mark.e2e`, register the marker in pyproject, and use `-m 'not e2e'` (not `--ignore`) in `addopts`.

**Web frontends** — repos with a JS/TS frontend opt in via `web` (frontend directory) + `web_pm` (`npm` or `pnpm`) vars on ci.yml in `files-ci.yaml`. This renders a `web` matrix leg that installs from the lockfile (`npm ci` / `pnpm install --frozen-lockfile`) and runs `just web-check` (type-check, lint and format checks, then the unit tests via `web-test`), Node setup pinned by `<web>/.node-version`, and a `pnpm/action-setup` step when `web_pm: "pnpm"`. The same vars on the Justfile FileSet render managed `web-*` recipes (SNORE). syncify self-manages its Justfile but conforms to the `web-*` recipe contract. Managed `web-*` recipes call package.json scripts, which must be named `type-check`, `lint-check`, `format-check`, `lint`, `format`, `build`, `test:unit`. `check-all` appends `web-install web-check` and `pre-commit` appends `web-install web-type-check web-lint web-format` for repos with the `web` var, so the managed pre-commit hook auto-fixes UI issues at commit time. `check` and `ci` still exclude web checks — the CI web leg runs them separately.

**Locked installs in CI** — `ci-python.yml` installs with `uv sync --locked` (and the frozen JS install above) directly rather than through the `sync` / `web-install` recipes, so a lockfile that is stale against its manifest fails CI instead of being silently re-resolved. The Justfile recipes keep re-locking for local use, and self-managed Justfiles (syncify) need no CI-specific recipes.

## Common Gotchas

1. **`ci-python.yml` excluded from actionlint** — `<% %>` directives are not valid YAML; CI explicitly skips it. Don't "fix" the syntax errors — they're intentional template directives.

2. **gh-infra is pinned to a `wpfleger96/gh-infra` `dev` commit** — all 4 CI workflows fetch the SHA in `.gh-infra-ref`; bump it after every `dev` rebuild (a `dev` that lost a merged PR would otherwise silently change behavior, e.g. commit templates landing as literal text). `dev` is `upstream/main` with in-flight PRs [gh-infra#160](https://github.com/babarot/gh-infra/pull/160), [gh-infra#164](https://github.com/babarot/gh-infra/pull/164), [gh-infra#203](https://github.com/babarot/gh-infra/pull/203) merged in, plus [gh-infra#210](https://github.com/babarot/gh-infra/pull/210) (disabling Actions sends only `enabled`) cherry-picked, and is rebuilt from `upstream/main` as upstream merges them. Local builds must use the same SHA. Switch to a pinned release once they all ship.

3. **Private repos ignore `allow_auto_merge`** — GitHub Free plan silently accepts the API call but never applies it without rulesets. `allow_auto_merge` lives only in `repos-public.yaml` `defaults.spec.merge_strategy`; adding it to `repos-private.yaml` causes infinite plan drift.

4. **`via: push` needs the `workflow` OAuth scope** — `gh infra apply` uses `CreateCommitOnBranch` GraphQL to push to `.github/workflows/`. Confirm scope: `gh auth status`.

5. **Secrets can't be diffed** — `gh infra plan` shows new secrets but not existing values. Use `--force-secrets` after rotation.

6. **SC2129** — actionlint/shellcheck flags multiple `>> $GITHUB_OUTPUT` lines. Fix: `{ echo "k=v"; echo "k2=v2"; } >> "$GITHUB_OUTPUT"`.

7. **`release.yml` excluded from actionlint** — stale `@v3` metadata triggers actionlint#648; CI explicitly skips it.

8. **`fork_pr_approval` is public-only** — `gh infra validate` and `plan` both hard-reject it on private repos. It lives in `repos-public.yaml` `defaults.spec.actions` only; private repos have no equivalent setting. Set to `all_external_contributors` so every fork PR from an outside contributor needs maintainer approval before CI runs, not just first-timers.

9. **No `$comment` in `renovate.json`** — Renovate's config validator only whitelists `$schema` as an ignored key. Any other unrecognized field (including `$comment`) is rejected as an invalid config option. Use a YAML/JSON comment-less approach or put provenance in the managed-file header for non-JSON formats only.

10. **Plan output exposes private repo files** — `infra-drift.yml` (`plan --ci --diff`) and the PR plan comment from `infra-plan.yml` print managed-file diffs for every repo into public Actions logs/PR comments. The desired side is the public templates, but drifted content from private repos is printed verbatim. Don't put secrets in managed files.

11. **github-config's own workflows pin actions by full commit SHA** — `sha_pinning_required: true` on this repo makes GitHub reject any `uses:` with a tag or branch ref. Write `owner/action@<40-char SHA> # vX.Y.Z`; Renovate updates both. The release-app token these workflows mint can read and write every managed repo, so a retagged action must not be able to run here.

## Key Files by Task

| Task | File(s) |
|------|---------|
| Add repo to management | `github/repos-public.yaml` or `github/repos-private.yaml` (the file sets visibility) + relevant `files-*.yaml` |
| Distribute a new file | `github/templates/` + new or existing `files-*.yaml` (a new FileSet sets `metadata.name`) |
| Update CI template | `github/templates/ci-python.yml` |
| Update shared Justfile | `github/templates/Justfile` |
| Update Renovate preset | `renovate-config/default.json` |
| Update PR/issue templates | `github/templates/pull_request_template.md` (via `files-all.yaml`), `github/templates/ISSUE_TEMPLATE/` (via `files-issues.yaml`; labels they apply live in `defaults.spec.labels` of both `repos-*.yaml` files) |
| Change branch protection | `github/repos-public.yaml` → `defaults.spec.rulesets` (per-repo: entry `spec.rulesets`) |
| Add repo secret/variable | per-repo `spec` in `github/repos-public.yaml` / `github/repos-private.yaml`, or `defaults.spec` for all repos in one visibility (secret values must be `${ENV_*}` refs) |
| Change infra workflows | `.github/workflows/infra-plan.yml` / `infra-apply.yml` / `infra-drift.yml` |
