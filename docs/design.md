# Design notes

## Why `process_when` duplicates `dispatch.when`

`inbound.dispatch` bypasses the orchestrator entirely and returns as soon as it
matches — `process_when` never runs for a matched event. Any event that
*does* reach `process_when` is therefore, by construction, one that already
failed the identical `dispatch.when` condition. Repeating the same rule there
turns `process_when` into a deny-by-default net: nothing this channel
receives is ever mapped into an `InboundMessage` and handed to the LLM
orchestrator. There is no legitimate use case in v1 for a raw GitLab webhook
payload reaching an LLM — this channel's only job is the deterministic
comment-to-pipeline-trigger relay.

`mapping` is configured anyway (`content`/`sender_id`/etc.) even though it's
currently dead code under that net, so that loosening `process_when` later —
e.g. to support some non-command chat use case — degrades into a sane
`InboundMessage` instead of an empty/unmapped one.

## Why the trigger match can't also require `noteable_type == MergeRequest`

The core's `dispatch.when`/`process_when`/`skip` rule lists are OR'd across
entries — there's no way to express "field A matches AND field B matches" in
the current DSL (see `opentalon/internal/channel/yaml_ws.go`'s `matchRule`).
So the trigger phrase appearing in a comment on an *issue* (GitLab's
`noteable_type: "Issue"`) will still fire a Pipeline Trigger API call, even
though the triggered job only makes sense for merge requests.

This is bounded, not silent, as long as the CI job validates its input: the
triggered pipeline's `REVIEW_MR_IID` variable is templated from
`{{event.merge_request.iid}}`, which is absent on an issue comment and
resolves to an empty string. A job that rejects a non-positive/missing MR
IID and skips the run turns the failure mode into a wasted CI pipeline
invocation that immediately no-ops, not a wrong comment posted somewhere.

If this turns out to matter (e.g. GitLab CI minutes cost becomes relevant),
the real fix is extending the core DSL to support AND-of-fields across a
rule list. Rejecting inside the job can't avoid the cost — GitLab decides
whether to run the pipeline before any job code executes. Not attempted
here.

## Why system notes are excluded via `dispatch.skip`, not `dispatch.when`

`object_attributes.system: true` marks a GitLab-generated note (e.g. "mentioned
in commit ...", "unresolved a thread") rather than a human-typed comment.
These can embed referenced text — a cross-reference note can, in principle,
quote a commit message — so a system note containing the trigger phrase
is a real (if narrow) way for an attacker who can push a commit
to a fork to get a comment containing arbitrary text attributed to a system
note on the target project. `dispatch.skip` (added alongside `dispatch.when`
support for this exact purpose in `opentalon` PR #370, fixing
[opentalon/opentalon#364](https://github.com/opentalon/opentalon/issues/364))
excludes these regardless of what the note body contains.

## Why `signature_scheme: standard_webhooks`, not `secret_header`

GitLab offers two webhook auth modes. The legacy "Secret token" sends a
static value in the `X-Gitlab-Token` header on every delivery, so anything
that logs headers between GitLab and OpenTalon leaks a reusable credential;
GitLab's own UI labels it "not recommended". The "Signing token" mode
(GitLab 19.0+) follows the Standard Webhooks spec: GitLab sends
`webhook-id`, `webhook-timestamp` and `webhook-signature: v1,<base64>`,
where the signature is HMAC-SHA256 over `{id}.{timestamp}.{raw body}` keyed
with the base64-decoded `whsec_` token. Only the signature travels; a
captured request can't be replayed after the core's 5-minute timestamp
window, and can't be altered without the key.

The core gained this as `signature_scheme`/`signature_secret` in
`opentalon` PR #375 (issue #374). This channel used `secret_header`
(`opentalon` PR #370) until then only because it was the sole shared-secret
option. `validate_jwt` was never an option: it only speaks OIDC JWTs (built
for Microsoft Bot Framework channels). `inbound.dispatch` on a public
`http_webhook` is rejected at channel-spec load time unless `validate_jwt`,
`secret_header` or `signature_scheme` is configured.

## Why the Pipeline Trigger API call uses `variables[KEY]=value` form encoding

GitLab's documented `--form` examples for the trigger-pipeline endpoint use
`token`/`ref` as query parameters and `variables[KEY]=value` as
`application/x-www-form-urlencoded` (or multipart) body fields — not a JSON
object. `REVIEW_MR_IID`/`REVIEW_NOTE_ID` are sent as pipeline **variables**
(the legacy mechanism), not the newer CI/CD Catalog **inputs** mechanism,
which *is* JSON-only. **Verified against a live GitLab.com project** — this is the
correct wire format; the README's old "unverified, has no test suite of its
own" caveat about this specific call no longer applies.

## Why `ref` is the MR's source branch, not the default branch

Was the project's default branch originally, on the reasoning that the
triggered pipeline only needs *some* branch's `.gitlab-ci.yml` to run
against, and the CI job identifies which MR to act on from
`REVIEW_MR_IID` rather than any `CI_MERGE_REQUEST_*` predefined variable
(those only populate for `merge_request_event`-sourced pipelines, not
API-triggered ones).

That reasoning missed a real consequence, found against a live project: a
project's own `.gitlab-ci.yml` commonly has a top-level `workflow: rules:`
gating on `$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH` (among others) — ref'ing
the default branch makes a comment-triggered pipeline match that rule, which
runs the *entire* pipeline (build/test/deploy jobs included), not just the
comment-triggered job. A PR comment could fire a production deploy. Fixed
to `{{event.merge_request.source_branch}}`. A project adding the triggered
job to existing CI also needs a `$CI_PIPELINE_SOURCE == "trigger"` workflow
rule and per-job guards on anything that shouldn't run on a trigger-sourced
pipeline.

## Why `dispatch.skip` also excludes the relay's own GitLab username

GitLab's Note Hook fires for *every* note on a merge request, including ones
the triggered job itself posts (sticky summary/usage comments, command
confirmations) — and those bodies legitimately contain the trigger phrase
(usage text lists every command; a confirmation may say "until `!review` is
run"). `process_when`/`dispatch.when` match on that substring with no author
check, so without this skip rule, the bot's own replies re-trigger a pipeline
against themselves — confirmed live: two of the bot's own posted comments on
a real test MR each fired a real (wasted) pipeline run, one of them twice.

GitHub has no equivalent gap — events caused by `GITHUB_TOKEN` don't
retrigger GitHub Actions workflows at all, so this class of bug can't exist
there. GitLab has no analogous protection; `gitlab-channel` has to implement
its own, the same way `dispatch.skip`'s `object_attributes.system` exclusion
already does for GitLab's own system-generated notes.

`GITLAB_BOT_USERNAME` is therefore a **required** env var, not optional —
without it, the skip rule's `equals` compares against an empty template
result, which never matches a real username, so the self-trigger stays live
by default rather than failing closed. Set it to the username of whatever
account the CI job posts its replies as; a [project or group access
token](https://docs.gitlab.com/user/project/settings/project_access_tokens/)
auto-provisions a dedicated bot user (`project_{id}_bot_{random}`, or the
token's own name if you rename it at creation) specifically so this value is
stable and distinct from any human's username.

## Why the trigger phrase is an env var

`GITLAB_TRIGGER_PHRASE` keeps the channel independent of any particular bot:
the phrase belongs to whatever CI job consumes the trigger, not to the relay.
It is templated into both `process_when` and `dispatch.when` as a `contains`
needle. On cores before `opentalon` PR #378 a template resolving to empty
made `contains` match every string, so an unset phrase meant every comment
on every merge request fired a pipeline. #378 makes a rule whose template
resolves to empty never match, so an unset phrase fails closed: nothing is
dispatched.
