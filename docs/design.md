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
So `!talooner` appearing in a comment on an *issue* (GitLab's `noteable_type:
"Issue"`) will still fire a Pipeline Trigger API call, even though Talooner
only reviews merge requests.

This is bounded, not silent: the triggered pipeline's `TALOONER_MR_IID`
variable is templated from `{{event.merge_request.iid}}`, which is absent on
an issue comment and resolves to an empty string. Talooner's own
`internal/event/gitlab` trigger-parsing path
(`opentalon/talooner`, PR #112) already rejects a non-positive/missing MR
IID and skips the run — so the failure mode is a wasted CI pipeline
invocation that immediately no-ops, not a wrong review posted somewhere.

If this turns out to matter (e.g. GitLab CI minutes cost becomes relevant),
the real fix is either extending the core DSL to support AND-of-fields
across a rule list, or teaching Talooner's own event parser to reject before
the pipeline even starts a job (it can't — GitLab decides whether to run the
pipeline before any Talooner code executes). Not attempted here.

## Why system notes are excluded via `dispatch.skip`, not `dispatch.when`

`object_attributes.system: true` marks a GitLab-generated note (e.g. "mentioned
in commit ...", "unresolved a thread") rather than a human-typed comment.
These can embed referenced text — a cross-reference note can, in principle,
quote a commit message — so a system note containing the literal string
`!talooner` is a real (if narrow) way for an attacker who can push a commit
to a fork to get a comment containing arbitrary text attributed to a system
note on the target project. `dispatch.skip` (added alongside `dispatch.when`
support for this exact purpose in `opentalon` PR #370, fixing
[opentalon/opentalon#364](https://github.com/opentalon/opentalon/issues/364))
excludes these regardless of what the note body contains.

## Why `secret_header`, not `validate_jwt`

GitLab's Note Hook authenticates with a static token in the `X-Gitlab-Token`
header (set once, when the webhook is created) — not a JWT. `validate_jwt`
in `opentalon`'s core webhook spec only speaks OIDC JWTs (built for
Microsoft Bot Framework channels). `secret_header`/`secret_value` was added
to the core (`opentalon` PR #370) specifically so this channel — and any
other host with a shared-secret-header webhook auth scheme — has a
first-class, fail-closed option. `inbound.dispatch` on a public
`http_webhook` is rejected at channel-spec load time unless one of
`validate_jwt` or `secret_header` is configured; see that PR for the
enforcement.

## Why the Pipeline Trigger API call uses `variables[KEY]=value` form encoding

GitLab's documented `--form` examples for the trigger-pipeline endpoint use
`token`/`ref` as query parameters and `variables[KEY]=value` as
`application/x-www-form-urlencoded` (or multipart) body fields — not a JSON
object. Talooner's GitLab event parser (`internal/event/gitlab`) reads
`TALOONER_MR_IID`/`TALOONER_NOTE_ID` as pipeline **variables** (the legacy
mechanism), not the newer CI/CD Catalog **inputs** mechanism, which *is*
JSON-only. This repo has no Go module and no test suite of its own to verify
the exact wire format against a live GitLab instance — treat this as
unverified until exercised for real, per the caveat in the README.

## Why `ref` is the project's default branch

The triggered pipeline needs some branch's `.gitlab-ci.yml` to run against.
It isn't the MR's source or target branch — Talooner's CI job identifies
which MR to review from `TALOONER_MR_IID` and fetches whatever it needs via
the GitLab API itself, it doesn't rely on `CI_MERGE_REQUEST_*` predefined
variables (those are only populated for `merge_request_event`-sourced
pipelines, not API-triggered ones). `ref` here only selects which
`.gitlab-ci.yml` revision defines the trigger-source pipeline job — the
default branch is the standard stable choice.
