# Research: community-scripts contribution mechanics & companion-app precedent

- **Ticket:** cachemoney/cliautoproxy-ws#2 (part of #1)
- **Date:** 2026-10-13
- **Question:** What does contributing a new LXC script to community-scripts require, and does CPA-Manager-Plus (CPAMP) qualify? What is the companion-app precedent (an app that connects to a separate service in another container)? What CI must a PR pass?
- **Resolution highlight:** **Recommend CPAMP-only architecture** (see bottom).

## The 2026 re-architecture (this changes the ticket's premise)

community-scripts is no longer a single repo. Three repos now split engine, incubation, and stable apps ([core README](https://github.com/community-scripts/core#readme), verified 2026-10-13):

| Repo | Role |
| --- | --- |
| [`community-scripts/core`](https://github.com/community-scripts/core) | The shared shell **engine**: whiptail wizard, container/VM creation, in-container installer, error handling, helpers (`build.func`, `tools.func`, …) |
| [`community-scripts/DevScripts`](https://github.com/community-scripts/DevScripts) | Where **all new scripts start** and are tested (formerly ProxmoxVED) |
| [`community-scripts/ProxmoxVE`](https://github.com/community-scripts/ProxmoxVE) | Stable application scripts; reached only by bot promotion from DevScripts |

Consequences:

1. **A new-app PR must target DevScripts, not ProxmoxVE.** ProxmoxVE's [`close-new-script-prs.yml`](https://github.com/community-scripts/ProxmoxVE/blob/main/.github/workflows/close-new-script-prs.yml) automatically closes any PR that adds files under `ct/`, `install/`, `turnkey/`, or `vm/` unless the author is OWNER/MEMBER/COLLABORATOR or an allowed bot. The close comment says: *"New scripts must be submitted to DevScripts for testing. PRs without prior testing will be closed."* The ProxmoxVE PR template carries the same warning at the top.
2. **App scripts load Core, they don't carry it.** `ct/<app>.sh` bootstraps via `source ... core/build.func` — local checkout first (`COMMUNITY_SCRIPTS_CORE_DIR` or a sibling `../core`), falling back to `raw.githubusercontent.com/community-scripts/core/main` ([core docs/core.md](https://github.com/community-scripts/core/blob/main/docs/core.md); see [`ct/cliproxyapi.sh:1-3`](https://github.com/community-scripts/ProxmoxVE/blob/main/ct/cliproxyapi.sh), ProxmoxVE main @ `1e4b435`).
3. **The website JSON no longer lives in ProxmoxVE.** ProxmoxVE main has 625 `ct/*.sh` and **zero** `ct/*.json`. Metadata is authored as `json/<app>.json` **in DevScripts**; on merge, [`push_json_to_pocketbase.yml`](https://github.com/community-scripts/DevScripts/blob/main/.github/workflows/push_json_to_pocketbase.yml) pushes each record (files with a `.slug`) into the site's PocketBase backend. The site's [script-metadata page](https://community-scripts.org/docs/contribution/templates_json/appname) puts it plainly: *"Website metadata is stored as a record and is edited through the site tooling; it is not a second source of shell behaviour."*

Promotion path (all verified in DevScripts' [`move-to-main-repo.yaml`](https://github.com/community-scripts/DevScripts/blob/main/.github/workflows/move-to-main-repo.yaml)):

1. Author opens a **🛠️ New Script issue** in DevScripts ([template](https://github.com/community-scripts/DevScripts/blob/main/.github/ISSUE_TEMPLATE/new-script.yaml): issue name = APP name, e.g. "SnipeIT" ↔ `snipeit.sh`) — this becomes the testing/feedback thread.
2. The ct script points `var_testurl` at that issue so test containers nag for feedback ([DevScripts AGENTS.md](https://github.com/community-scripts/DevScripts/blob/main/AGENTS.md), *Important Rules*).
3. After testing/approval, the "Migration To ProxmoxVE" label triggers the bot: it verifies the file trio exists, clones ProxmoxVE, copies `ct/<app>.sh` + `install/<app>-install.sh`, **deletes the `var_testurl` and `_CS_DEFAULT_URL` lines**, rewrites `community-scripts/DevScripts` → `community-scripts/ProxmoxVE` in URLs, and opens a bot PR to ProxmoxVE main. `json/` is not copied — it lives in PocketBase.

## 1. File layout & mechanics for a new app

**The file set is exactly three files, moving together** (a Greptile review rule in DevScripts' [`greptile.json`](https://github.com/community-scripts/DevScripts/blob/main/greptile.json): *"Adding or changing an app script means all three files move together: ct/<app>.sh, install/<app>-install.sh and json/<app>.json. Flag a PR that touches one and not the others."*):

### `ct/<app>.sh` — host-side entry script

Canonical shape (DevScripts [AGENTS.md](https://github.com/community-scripts/DevScripts/blob/main/AGENTS.md) *CT Script* template; real examples: [`ct/cliproxyapi.sh`](https://github.com/community-scripts/ProxmoxVE/blob/main/ct/cliproxyapi.sh), [`ct/journiv.sh`](https://github.com/community-scripts/ProxmoxVE/blob/main/ct/journiv.sh) — the docs' designated "good example"):

- `#!/usr/bin/env bash`
- `_CS_DEFAULT_URL="https://raw.githubusercontent.com/community-scripts/DevScripts/main"` (DevScripts only; the promotion bot strips this line)
- `_cs_boot="${COMMUNITY_SCRIPTS_CORE_DIR:-$(dirname "${BASH_SOURCE[0]}")/../../core}/core/build.func"` + `source "$_cs_boot" 2>/dev/null || source <(curl -fsSL ...core/main...)` — **no comment block above this** (anti-pattern #32)
- Copyright header: `Author:`, `License: MIT | ...`, `Source: <upstream URL>`
- `APP="PrettyName"` and `var_*` defaults, each with `${var_x:-…}` override syntax: `var_tags` (semicolon-separated), `var_cpu`, `var_ram`, `var_disk`, `var_os="debian"`, `var_version="13"`, `var_unprivileged="1"`; optional `var_arm64` (three-state: `yes` = verified, `no` = known broken + stated reason, unset = ask), `var_gpu`, `var_testurl`
- Exported app inputs: `export var_admin_user="${var_admin_user:-}"` — *without the export the value never reaches the container* (`lxc-attach` only carries exported vars)
- `header_info "$APP"` / `variables` / `color` / `catch_errors`
- **Exactly one function**: `update_script()` — `header_info`, `check_container_storage`, `check_container_resources`, install-exists check, `check_for_gh_release` guard, stop service, `create_backup` config files, `CLEAN_INSTALL=1 CLEAN_INSTALL_KEEP="…" fetch_and_deploy_gh_release …`, `restore_backup`, rebuild if needed, start service, always ends with `exit`
- Footer: `start` / `build_container` / `description` + `msg_ok` and the access-URL echo

### `install/<app>-install.sh` — runs inside the new container

- Sources `source /dev/stdin <<<"$FUNCTIONS_FILE_PATH"` (Core's prepared install runtime), then `color`, `verb_ip6`, `catch_errors`, `setting_up_container`, `network_check`, `update_os`
- Dependencies: `$STD apt install -y <only real deps>` (never `curl`/`sudo`/`wget`/`jq`/`mc` — preinstalled)
- Deploy: `fetch_and_deploy_gh_release "<app>" "owner/repo" "<mode>" ["latest"] ["/opt/<app>"] ["<asset-glob>"]` — modes `tarball` (source), `binary` (.deb), `prebuild` (prebuilt archive), `singlefile` (single binary). Multi-arch assets use `$(arch_resolve "amd64" "aarch64")` **inline** in the glob — exactly what `install/cliproxyapi-install.sh:20` does: `fetch_and_deploy_gh_release "cliproxyapi" "router-for-me/CLIProxyAPI" "prebuild" "latest" "/opt/cliproxyapi" "CLIProxyAPI_*_linux_$(arch_resolve "amd64" "aarch64").tar.gz"`
- Runtimes/DBs via `setup_*` helpers (`setup_nodejs`, `setup_uv`, `setup_postgresql`, …); config written with a single heredoc; secrets via `random_password` (hex tokens stay `openssl rand -hex`)
- systemd unit heredoc → `systemctl enable -q --now <app>` (no `daemon-reload` for new units; no service users — LXC runs as root and *is* the isolation boundary)
- Footer, always: `motd_ssh` / `customize` / `cleanup_lxc`
- **Zero functions of its own**; bare-metal only

### `json/<app>.json` — website metadata record (DevScripts `json/`)

Full field model per [AGENTS.md JSON section](https://github.com/community-scripts/DevScripts/blob/main/AGENTS.md) and a live example ([`json/4gaboards.json`](https://github.com/community-scripts/DevScripts/blob/main/json/4gaboards.json)):

- Required: `name`, `slug` (lowercase), `categories` (numeric IDs; e.g. 20 = *AI / Coding & Dev-Tools*, 9 = *Monitoring & Analytics*), `date_created`, `type: "ct"`, `updateable`, `privileged`, `interface_port`, `documentation`, `website`, `repository` (full URL — GitLab/Gitea/Forgejo/Codeberg supported), `architectures` (must agree with `var_arm64`), `logo` (prefer selfhst icons CDN), `description`, `install_methods` `[{type:"default", script:"ct/<app>.sh", config_path, resources{cpu,ram,hdd,os,version}}]`, `default_credentials`, `notes[]`
- Optional: `platforms` (`["pve"]`/`["incus"]`), **`app_vars`** — declares each `var_*` input (name/label/type `text|password|number|boolean|select`/default/required/help) so the website can render unattended-deployment fields
- Path rules ([site metadata page](https://community-scripts.org/docs/contribution/templates_json/appname)): `install_methods[].script` wins; if empty the site derives the path from type+slug; Alpine variants resolve to `ct/alpine-<slug>.sh`; scripts under test stay pointed at DevScripts

**About "the JSON generator":** the old model (a web generator emitting `ct/<app>.json` committed into ProxmoxVE) is retired. Today the record is authored by hand (or via the repo's `.github/agents/pve-script-creator.agent.md` scaffolding agent) as `json/<app>.json` in DevScripts, then synced to PocketBase automatically. Local pre-flight validation is light: the push workflow only requires the file to parse and contain `.slug` (`jq -e '.slug'`).

### Naming rules

- Slug: **lowercase, hyphenated** (`cpa-manager-plus`); files `ct/<slug>.sh`, `install/<slug>-install.sh`, `json/<slug>.json`; `APP="CPA-Manager-Plus"`-style display name; the site's [contribution rules](https://community-scripts.org/docs/contribution/contributing): *"Keep a script focused on one application and use lowercase, hyphenated filenames."*
- Alpine variants: Shape A (`alpine-<slug>` prefixed pair) or Shape B (one multi-OS script branching only resource defaults) — never a duplicated script

### Version-pinning rules

- **App versions are never pinned.** Anti-pattern #18 (*hardcoded versions for external tools*): always `fetch_and_deploy_gh_release … "latest"`; the resolved version is recorded in `~/.<app>` (read with `cat ~/.<app>`). Update checks go through `check_for_gh_release`, never hand-rolled `curl api.github.com` diffs. Pre-release-only projects set `GH_INCLUDE_PRERELEASE=1`.
- What **is** pinned: the container OS (`var_os`, `var_version`), runtime majors for `setup_*` helpers (`NODE_VERSION="22"`), and (implicitly) asset glob patterns.
- CI/review flags "a pinned upstream URL that embeds a version" as rot ([greptile.json](https://github.com/community-scripts/DevScripts/blob/main/greptile.json) rules).

## 2. Acceptance criteria — does CPAMP pass?

The **hard rules** live in DevScripts' [PR template](https://github.com/community-scripts/DevScripts/blob/main/.github/pull_request_template.md), "📦 Application Requirements (for new scripts)", enforced by [`unmet-pr-close.yml`](https://github.com/community-scripts/DevScripts/blob/main/.github/workflows/unmet-pr-close.yml) (missing/unchecked → `requirements missing` label + contributor-team ping; [`stale_pr_close.yml`](https://github.com/community-scripts/DevScripts/blob/main/.github/workflows/stale_pr_close.yml) closes after 7 days):

> - The application is **at least 6 months old**
> - The application is **actively maintained**
> - The application has **600+ GitHub stars**
> - Official **release tarballs** are published
> - "I understand that not all scripts will be accepted due to various reasons and criteria by the community-scripts ORG"

Plus the structural policy: **bare-metal only.** *"If the application can only be installed via Docker, stop… A Docker-based ct/ script is rejected on sight"* (AGENTS.md, *Bare-Metal Installation*; Docker is tolerated only in `tools/addon/` compose managers). The template's review section also forbids `git pull` updates and hardcoded credentials. No explicit OSI-license allowlist for apps appears in the requirements; the scripts themselves are MIT.

**CPAMP scorecard** (GitHub API, 2026-10-13, [`seakee/CPA-Manager-Plus`](https://github.com/seakee/CPA-Manager-Plus)):

| Criterion | Value | Verdict |
| --- | --- | --- |
| 600+ stars | **3,832** | ✅ (6.4× the bar) |
| Actively maintained | pushed & released **v1.14.6 on 2026-10-13** (today), not archived | ✅ |
| Official release tarballs | `cpa-manager-plus_v1.14.6_linux_{amd64,arm64}.tar.gz` + checksums + `management.html` | ✅ |
| ≥ 6 months old | created **2026-05-17** → 4 months 26 days | ⚠️ crosses the line **2026-11-17** |
| License | MIT | ✅ |
| Installable bare-metal | Go binaries in release assets → `prebuild` mode + `arch_resolve`, native systemd — mirrors `cliproxyapi.sh` exactly | ✅ |

The only miss is the age rule, by ~5 weeks. It doesn't block building a contribution-ready branch now (our destination is a pushed fork branch, no PR), but an actual DevScripts PR before mid-November would technically violate the letter of the requirement — worth acknowledging in any future PR description. For calibration: CLIProxyAPI itself (55,065 stars, MIT, admitted via `ct/cliproxyapi.sh`) sails through.

## 3. Companion-app precedent (app talking to a service in another container)

**"One application per script" in practice:** the script installs exactly one app's artifacts into one container. Anything the app needs from elsewhere is supplied as a **prompted input** — a `var_<name>` that is read env-first (unattended-friendly), exported from `ct/<app>.sh`, and declared in the JSON `app_vars` — never installed alongside. No script in either repo ships two apps.

Three concrete precedents:

1. **forgejo-runner** (ProxmoxVE [`ct/forgejo-runner.sh`](https://github.com/community-scripts/ProxmoxVE/blob/main/ct/forgejo-runner.sh) + [`install/forgejo-runner-install.sh`](https://github.com/community-scripts/ProxmoxVE/blob/main/install/forgejo-runner-install.sh)) — a CI runner whose control plane (a Forgejo instance) lives elsewhere. The installer prompts `var_forgejo_instance` *"Forgejo Instance URL (e.g. https://codeberg.org)"* (with default), plus runner UUID + token, each guarded by `if [[ -z "${var_…:-}" ]]` so pre-supplied values skip the prompt. This is precisely the CPAMP→CLIProxyAPI shape: one container, pointing at an existing service.
2. **gitea-runner** (DevScripts [`json/gitea-runner.json`](https://github.com/community-scripts/DevScripts/blob/main/json/gitea-runner.json)) — remote endpoint as a first-class website field: `app_vars` includes `var_gitea_url` (*"URL of your Gitea instance (e.g. http://192.168.1.100:3000 or https://gitea.example.com)"*) and `var_gitea_token` (`type: password`, `secret: true`).
3. **woodpecker-ci** (DevScripts [`json/woodpecker-ci.json`](https://github.com/community-scripts/DevScripts/blob/main/json/woodpecker-ci.json)) — the app has *no local accounts at all*; sign-in goes through an external forge. `app_vars`: `var_forge` (`select`: forgejo/gitea/github/gitlab/bitbucket) + `var_forge_url`, explicitly allowed to stay empty ("configure it after the install"), with `notes` explaining the external dependency and webhook reachability.

The pattern's required mechanics (AGENTS.md *Prompting Without an Escape Hatch* + `prompt_input_required`): read the variable before prompting, `var_`-prefix, export from the CT script, declare in JSON `app_vars`, fail early in the CT script when a required var is missing for unattended runs. A **CPAMP-only LXC that prompts for the existing CLIProxyAPI instance URL (and management secret/API key) is squarely the accepted pattern**; a CPA+CPAMP bundle has zero precedent and would violate the one-app rule.

## 4. CI / checks a PR must pass

| Check | Where | What it does |
| --- | --- | --- |
| `bash -n` on every `ct/`, `install/`, `vm/`, `tools/` `.sh` | DevScripts [`bash-syntax.yml`](https://github.com/community-scripts/DevScripts/blob/main/.github/workflows/bash-syntax.yml) | **Syntax check only** — the engine's own `.func` files are checked in core's CI. No shellcheck/shfmt CI exists in either repo. |
| Template + requirements validation | DevScripts [`unmet-pr-close.yml`](https://github.com/community-scripts/DevScripts/blob/main/.github/workflows/unmet-pr-close.yml) | PR must use the template; new-script PRs must have the 📦 Application Requirements checkboxes checked, else `requirements missing` label; closed by `stale_pr_close` after 7 days (bypass: `keep-open` label) |
| Greptile AI review | DevScripts [`greptile.json`](https://github.com/community-scripts/DevScripts/blob/main/greptile.json) | AGENTS.md rules are *binding*: flags hand-rolled download/version-check/runtime logic, Docker installs, rotting pinned URLs, and PRs not moving all three files together |
| Auto-labeling | DevScripts + ProxmoxVE `autolabeler.yml` (+ `autolabeler-config.json`) | Labels `new script` / `update script` / `delete script` by file status & path |
| New-script gate | ProxmoxVE [`close-new-script-prs.yml`](https://github.com/community-scripts/ProxmoxVE/blob/main/.github/workflows/close-new-script-prs.yml) | Auto-closes new-script PRs from non-trusted authors |
| Template gate | ProxmoxVE [`close-invalid-pr-template.yml`](https://github.com/community-scripts/ProxmoxVE/blob/main/.github/workflows/close-invalid-pr-template.yml) | Closes PRs that don't use the PR template |
| Changelog | ProxmoxVE `changelog-pr.yml` | Automated bot PR; not a manual step |

**Local pre-flight recipe** (mirrors everything above):

1. `bash -n ct/<app>.sh install/<app>-install.sh` (all scripts, like CI does)
2. `jq -e '.slug' json/<app>.json` — parses, has slug
3. Walk the AGENTS.md **"Checklist Before PR Creation"** (≈40 items: no Docker, explicit `fetch_and_deploy_*` mode, no msg-wrapping helpers, `$STD`, `apt` not `apt-get`, `random_password`, balanced msg blocks, footer trio, `var_arm64` decided, …)
4. Test install **and** `update_script()` on a real PVE host (the PR template demands a "Tested on" statement; the docs require testing the actual install/update flow)
5. Optionally run core's dev-mode flags against a local core checkout for faster iteration

## Recommendation for the architecture ticket: **CPAMP-only**

**Build the CPAMP-only (Full Mode Manager) script; do not bundle CPA.** Justification:

1. **Contribution mechanics demand it.** One script = one application; the three-file trio maps 1:1 to one app. A bundle would double review surface, need a merged JSON record for two apps, and has no precedent in either repo.
2. **The companion pattern is established and well-tooled.** forgejo-runner / gitea-runner / woodpecker-ci show exactly how a "connects to a remote instance" app is done: prompt `var_cliproxyapi_url` + secret via the env-first pattern, export from the ct script, declare in `app_vars`, explain the dependency in `notes`. CPAMP-only slots into this mold identically.
3. **CPA already exists as `ct/cliproxyapi.sh`.** A bundle would maintain a second, divergent CLIProxyAPI install path and fight the existing script for update ownership; the community would direct any CPA changes to the existing script anyway.
4. **CPAMP qualifies on its own merits** — 3.8k stars, MIT, released today, prebuilt linux amd64+arm64 tarballs (only the 6-month age rule lags until 2026-11-17).

One consequence for the map's destination: since real contribution now flows through **DevScripts**, the `feat/cpa-manager-plus` branch should carry the **DevScripts shape** of the trio — including `_CS_DEFAULT_URL` pinned to DevScripts, `var_testurl` pointed at a (future) testing issue, and `json/cpa-manager-plus.json` — so that the branch is a drop-in DevScripts PR. The promotion bot then strips/rewrites those DevScripts-specific bits when moving to ProxmoxVE. (If the branch is kept on the ProxmoxVE fork as planned, omit `_CS_DEFAULT_URL`/`var_testurl` and keep the json/ file anyway for the eventual DevScripts PR.)

## Sources

- community-scripts.org: [Contribution rules](https://community-scripts.org/docs/contribution/contributing), [Script metadata](https://community-scripts.org/docs/contribution/templates_json/appname), plus /docs/contribution/ siblings (guide, code-audit, helper_functions, templates_ct, templates_install) — crawled 2026-10-13
- community-scripts/ProxmoxVE @ main `1e4b435` (clone): `ct/cliproxyapi.sh`, `install/cliproxyapi-install.sh`, `ct/forgejo-runner.sh`, `install/forgejo-runner-install.sh`, `.github/workflows/*`, `.github/autolabeler-config.json`, `.github/pull_request_template.md`
- community-scripts/DevScripts @ `46511d27`: `AGENTS.md`, `.github/pull_request_template.md`, `.github/ISSUE_TEMPLATE/new-script.yaml`, `.github/workflows/{bash-syntax,unmet-pr-close,stale_pr_close,move-to-main-repo,push_json_to_pocketbase}.yml`, `greptile.json`, `json/{gitea-runner,woodpecker-ci,4gaboards}.json`, `README.md`
- community-scripts/core @ `3d66bb9`: `README.md`, `docs/core.md`
- GitHub API (2026-10-13): `repos/seakee/CPA-Manager-Plus` (+ `releases/latest`), `repos/router-for-me/CLIProxyAPI`
