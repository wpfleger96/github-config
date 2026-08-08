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

> **Local gh-infra binary:** Build from the fork's dev branch (CI does too).
> `cd ~/Development/Personal/gh-infra && git pull && go build -o gh-infra ./cmd/gh-infra/`

## Managed Repos

| Repo | Visibility | Tier | FileSet overrides |
|------|-----------|------|------------------|
| ai-agent-rules | public | Full (CI+Deps+Release+Publish) | — |
| claude-code-status-line | public | Full (CI+Deps+Release+Publish) | — |
| pagerduty-mcp-server | public | Full (CI+Deps+Release+Publish) | — |
| JamBot | public | CI (CI+Deps+Justfile) | — |
| github-config | public | Self (CI self-managed; source only) | — |
| SNORE | public | CI+Deps+Justfile+Hooks+Release | `vars: web: "ui", web_pm: "pnpm"` on ci.yml + Justfile; `e2e: "true"` on ci.yml |
| shell-configs | private | Release (CI+Deps+Justfile+Release) | `vars: shell: "true"` on ci.yml + Justfile |
| recall | private | CI (CI+Deps+Justfile) | `vars: git_identity: "true"` on ci.yml |
| homelabconfigs | private | Deps (CI + Justfile self-managed) | — |
| meowdb | public | CI+Deps+Justfile+Hooks+Release | — |
| syncify | private | CI+Deps+Hooks (Justfile self-managed) | `vars: web: "ui/web", web_pm: "npm"` on ci.yml |
| envsync | private | CI (CI+Deps+Justfile+Hooks) | `vars: system_packages: "libsqlcipher-dev"` on ci.yml |
| BOOTLEG | private | CI (CI+Deps+Justfile+Hooks) | — |

## Project Structure

```
github/
  files-all.yaml       # renovate.json + auto-approve.yml → 12 repos
  files-ci.yaml        # ci.yml → 11 repos (github-config, homelabconfigs self-manage CI)
  files-full.yaml      # publish.yml → 3 PyPI repos
  files-hooks.yaml     # .hooks/pre-commit → 11 repos
  files-justfile.yaml  # Justfile → 10 repos (homelabconfigs, syncify excluded)
  files-release.yaml   # release.yml → 6 release-tier repos
  repos-public.yaml    # RepositorySet: 7 public repos (with rulesets)
  repos-private.yaml   # RepositorySet: 6 private repos (no rulesets — Free plan)
  templates/           # ci-python.yml, Justfile, auto-approve.yml, publish.yml,
                       # release.yml, renovate.json
renovate-config/
  default.json         # Shared Renovate preset extended by all managed repos
.github/workflows/     # ci.yml, infra-plan.yml, infra-apply.yml, infra-drift.yml
```

## Key Patterns

**FileSet vs RepositorySet:** `files-*.yaml` distributes template files to repos (`via: push` = direct commit, no PR). `repos-*.yaml` manages repo settings. Per-repo overrides use `vars:` (template variables) or `source:` (different file entirely).

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

**Self-managed CI** — repos with non-standard CI requirements (multi-component stacks, non-Python toolchains) own their `.github/workflows/ci.yml` directly. github-config manages only shared configs (`renovate.json`, `auto-approve.yml`) for these repos via `files-all.yaml`. Currently self-managed: github-config (Go-based gh-infra), homelabconfigs (Terraform + Ansible). **Public self-managed repos need a per-repo `rulesets` override in `repos-public.yaml`** to match the contexts their CI actually emits — the default ruleset requires `checks`, which only `ci-python.yml` repos satisfy.

**E2E testing** — the `e2e` job in `ci-python.yml` is opt-in via `vars: e2e: "true"` in `files-ci.yaml`. Currently only `ai-agent-rules` and `shell-configs` opt in. The default `main` ruleset requires only the `checks` context; `ai-agent-rules` has a per-repo ruleset override in `repos-public.yaml` that additionally requires `e2e`. The managed `Justfile` keeps e2e out of the fast run (`test: uv run pytest -m "not e2e"`) and runs them separately (`test-e2e`; `test-all` runs everything). Repos without e2e-marked tests pass trivially (the recipe tolerates pytest exit code 5 = no tests collected). To add e2e tests to a repo: create `tests/e2e/`, mark tests with `@pytest.mark.e2e`, register the marker in pyproject, and use `-m 'not e2e'` (not `--ignore`) in `addopts`.

**Web frontends** — repos with a JS/TS frontend opt in via `web` (frontend directory) + `web_pm` (`npm` or `pnpm`) vars on ci.yml in `files-ci.yaml`. This renders a `web` matrix leg running `just web-install web-check`, Node setup pinned by `<web>/.node-version`, and a `pnpm/action-setup` step when `web_pm: "pnpm"`. The same vars on the Justfile FileSet render managed `web-*` recipes (SNORE). syncify self-manages its Justfile but conforms to the `web-*` recipe contract. Managed `web-*` recipes call package.json scripts, which must be named `type-check`, `lint-check`, `format-check`, `lint`, `format`, `build`. `check-all` appends `web-install web-check` and `pre-commit` appends `web-install web-type-check web-lint web-format` for repos with the `web` var, so the managed pre-commit hook auto-fixes UI issues at commit time. `check` and `ci` still exclude web checks — the CI web leg runs them separately.

## Common Gotchas

1. **`ci-python.yml` excluded from actionlint** — `<% %>` directives are not valid YAML; CI explicitly skips it. Don't "fix" the syntax errors — they're intentional template directives.

2. **gh-infra must come from `wpfleger96/gh-infra -b dev`** — fork's dev branch merges 3 in-flight upstream contributions (#159, #160, #161). All 4 CI workflows build from this branch; local dev must too.

3. **Private repos ignore `allow_auto_merge`** — GitHub Free plan silently accepts the API call but never applies it without rulesets. Intentionally absent from `repos-private.yaml`; adding it back causes infinite plan drift.

4. **`via: push` needs the `workflow` OAuth scope** — `gh infra apply` uses `CreateCommitOnBranch` GraphQL to push to `.github/workflows/`. Confirm scope: `gh auth status`.

5. **Secrets can't be diffed** — `gh infra plan` shows new secrets but not existing values. Use `--force-secrets` after rotation.

6. **SC2129** — actionlint/shellcheck flags multiple `>> $GITHUB_OUTPUT` lines. Fix: `{ echo "k=v"; echo "k2=v2"; } >> "$GITHUB_OUTPUT"`.

7. **`release.yml` excluded from actionlint** — stale `@v3` metadata triggers actionlint#648; CI explicitly skips it.

8. **`fork_pr_approval` is public-only** — `gh infra validate` and `plan` both hard-reject it on private repos. It lives in `repos-public.yaml` `defaults.spec.actions` only; private repos have no equivalent setting. Set to `all_external_contributors` so every fork PR from an outside contributor needs maintainer approval before CI runs, not just first-timers.

9. **No `$comment` in `renovate.json`** — Renovate's config validator only whitelists `$schema` as an ignored key. Any other unrecognized field (including `$comment`) is rejected as an invalid config option. Use a YAML/JSON comment-less approach or put provenance in the managed-file header for non-JSON formats only.

## Key Files by Task

| Task | File(s) |
|------|---------|
| Add repo to management | `github/repos-public.yaml` or `repos-private.yaml` + relevant `files-*.yaml` |
| Distribute a new file | `github/templates/` + new or existing `files-*.yaml` |
| Update CI template | `github/templates/ci-python.yml` |
| Update shared Justfile | `github/templates/Justfile` |
| Update Renovate preset | `renovate-config/default.json` |
| Change branch protection | `github/repos-public.yaml` → rulesets section |
| Add repo secret/variable | `github/repos-public.yaml` or `repos-private.yaml` per-repo `spec` |
| Change infra workflows | `.github/workflows/infra-plan.yml` / `infra-apply.yml` / `infra-drift.yml` |
