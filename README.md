# OpenTalon GitLab Channel

YAML-driven GitLab webhook relay for [OpenTalon](https://github.com/opentalon/opentalon). No compiled binary — runs in-process using the core's generic YAML channel runtime.

This channel does exactly one job: receive GitLab's **Note Hook** webhook, and when a comment invokes [Talooner](https://github.com/opentalon/talooner) (`!talooner ...`), call GitLab's **Pipeline Trigger API** to run a Talooner CI job against that merge request. It never talks to the LLM orchestrator — see [docs/design.md](docs/design.md) for why.

GitLab CI has no native "comment triggers a job" event (unlike GitHub Actions' `issue_comment`). Talooner's own auto-review on push/reopen doesn't need this channel — that runs natively via GitLab's `merge_request_event` pipeline source. This channel exists only to bridge the comment-command path.

## Prerequisites

1. A GitLab project (or group) with a **pipeline trigger token** — Project → Settings → CI/CD → Pipeline trigger tokens
2. A webhook **signing token** — GitLab 19.0+ (GitLab.com today). GitLab signs every delivery with HMAC-SHA256 ([Standard Webhooks](https://www.standardwebhooks.com/) format, `webhook-signature` header), so the secret itself never travels on the wire, unlike the legacy "Secret token" mode GitLab labels "not recommended"
3. An OpenTalon core that includes [opentalon#375](https://github.com/opentalon/opentalon/pull/375) (`signature_scheme` webhook auth). Older cores reject this spec at load time — `inbound.dispatch` without any auth they recognize — rather than run unauthenticated
4. Your GitLab instance's API base URL (`https://gitlab.com/api/v4` for GitLab.com, or your self-hosted instance's equivalent)
5. A `.talooner`-configured project running Talooner's GitLab CI job (see [talooner#105](https://github.com/opentalon/talooner/issues/105)) — this channel only triggers the pipeline, it doesn't run Talooner itself
6. The GitLab username of whatever account `GITLAB_TOKEN` (Talooner's own write credential, not this channel's) authenticates as — **required**, not optional. Without it, Talooner's own replies (which often contain the literal string `!talooner`) re-trigger a pipeline against themselves. A [project or group access token](https://docs.gitlab.com/user/project/settings/project_access_tokens/) gives you a stable, dedicated bot username for this; see docs/design.md

### Creating the GitLab Webhook

1. Go to your project → **Settings → Webhooks → Add new webhook**
2. **URL**: `https://<your-opentalon-host>/webhook/gitlab`
3. **Signing token**: generate one (GitLab's UI offers a generate button), save it as `GITLAB_WEBHOOK_SIGNING_TOKEN` below. It has the form `whsec_<base64>`. Leave **Secret token** empty
4. **Trigger**: check **Comments** only (Note events) — nothing else is consumed
5. Save

### Creating the Pipeline Trigger Token

1. Go to your project → **Settings → CI/CD → Pipeline trigger tokens**
2. Add a trigger token, save it as `GITLAB_TRIGGER_TOKEN` below

## Setup

### 1. Clone this repo into your OpenTalon channels directory

```bash
cd your-opentalon-project
git clone https://github.com/opentalon/gitlab-channel channels/gitlab
```

Or let OpenTalon fetch it automatically via `github`/`ref` in config (see below).

### 2. Set environment variables

```bash
export GITLAB_TRIGGER_TOKEN="glptt-..."
export GITLAB_WEBHOOK_SIGNING_TOKEN="whsec_..."
export GITLAB_API_URL="https://gitlab.com/api/v4"
export GITLAB_BOT_USERNAME="project_12345_bot_abcdef"  # the username GITLAB_TOKEN authenticates as
```

Or add them to your `.env` file.

### 3. Add to your OpenTalon config.yaml

```yaml
channels:
  gitlab:
    enabled: true
    plugin: "./channels/gitlab/channel.yaml"
```

Or use auto-fetch from GitHub:

```yaml
channels:
  gitlab:
    enabled: true
    plugin: "./channels/gitlab/channel.yaml"
    github: "opentalon/gitlab-channel"
    ref: "master"
```

### 4. Expose the webhook publicly

`inbound.http_webhook` binds to the core's shared webhook server (default port 3978, path `/webhook/gitlab`). GitLab needs to reach it over the public internet — see `opentalon`'s [deployment guide](https://github.com/opentalon/opentalon/blob/master/docs/deployment-guide-k8s.md) for the Ingress example. Do not expose this endpoint without `GITLAB_WEBHOOK_SIGNING_TOKEN` set — see [docs/design.md](docs/design.md) for why that's enforced, not just recommended.

### 5. Run OpenTalon

```bash
source .env && go run ./cmd/opentalon -config config.yaml
```

You should see:

```
yaml-channel: gitlab started
channel-manager: loaded gitlab via yaml
```

## Usage

Comment on a merge request:

```
!talooner /review
```

The relay matches the comment, calls GitLab's Pipeline Trigger API with the MR IID and note ID as pipeline variables, and GitLab runs the Talooner CI job. Talooner itself parses the actual verb (`/review`, `/stop`, `/why`, `/plan`) from the note body once the job re-fetches it — this channel doesn't parse verbs, it only decides whether to trigger a pipeline at all.

## Known limitations (v1)

See [docs/design.md](docs/design.md) for the full rationale. Short version:

- Only comments on merge requests are meaningfully handled — comments on issues/epics containing `!talooner` will still trigger a pipeline call (the underlying YAML match DSL can't express "AND noteable_type == MergeRequest" alongside the trigger-phrase match), but Talooner's own trigger-parsing side rejects a missing/invalid MR IID, so this fails closed on the CI side, just wastefully.
- MR-close cleanup (unsubscribing Talooner's standing review) is not handled — no native GitLab pipeline source fires on MR close, and this channel doesn't listen to the Merge Request Hook, only the Note Hook. Deferred deliberately: supporting it would mean listening to a second webhook type for a resource-cleanup case, not core review behavior.
- Fork merge request behavior is unverified — every live test so far has been same-project branches. Whether the trigger call can even resolve a fork's branch ref against the base project, and what GitLab's fork-MR CI/CD variable restrictions do to this flow, has not been checked.

**Exercised against a live GitLab.com project** as of 2026-10 (real MRs, real pipelines, real webhook deliveries) — the Pipeline Trigger API wire format (`variables[KEY]=value` form-urlencoded body, `token`/`ref` query params) is confirmed correct, and the `ref`/dispatch-skip fixes above came directly from that testing. See [talooner's deployment-and-setup.md](https://github.com/opentalon/talooner/blob/master/docs/deployment-and-setup.md) for the full GitLab troubleshooting table this surfaced.

## Files

| File | Purpose |
|------|---------|
| `channel.yaml` | Channel spec — webhook auth, dispatch matching, Pipeline Trigger API call |
| `docs/design.md` | Non-obvious design decisions and why |
| `LICENSE` | Apache 2.0 License |
