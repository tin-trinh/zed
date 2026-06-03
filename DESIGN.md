# Auto-Compaction & Handoff — Design

## Summary

When an agent thread approaches its model's context window, Zed will automatically
**compact** the conversation and **hand off** into a fresh model window within the same
thread. The UI remains a flat transcript with a labeled divider, represented directly in the
thread message log as `Message::Compaction`.

On OpenAI models we replicate Codex's approach via the public `/responses/compact` endpoint
to get Codex-quality results; all other providers use a generic local-summarization fallback.

All automatic creation of new compaction markers ships behind a **`handoff` feature flag**
that is **not enabled for staff by default**.

## Goals

- Auto-compact + handoff as context fills, without the user starting a new thread.
- **Codex parity on OpenAI models** via the documented public Responses compaction API
  (verified available on `api.openai.com` with an API key:
  <https://developers.openai.com/api/docs/guides/compaction>).
- A generic fallback path (local summarization) for every other provider.
- Correct checkpoint restore across compaction boundaries.
- A simple flat message-log data model: compaction is a message boundary, not a nested
  conversation structure.

## Non-goals (v1)

- Manual `/compact` action (auto only).
- Subagent / session-source machinery (Codex tags compaction traffic as a subagent for
  server-side billing/telemetry/thread-list hygiene; none of that applies to us — see
  Appendix A).
- Editing a message that lives before an already-compacted boundary in-place. Restore is
  supported; in-place edit across the boundary is not.
- User-facing auto-compaction settings until the end of the rollout. V1 behavior stays behind the
  `handoff` flag with internal defaults while the functional pieces land.

## Feature flag

```rust
// crates/feature_flags/src/flags.rs
pub struct HandoffFeatureFlag;
impl FeatureFlag for HandoffFeatureFlag {
    const NAME: &'static str = "handoff";
    type Value = PresenceFlag;
    fn enabled_for_staff() -> bool { false } // off by default, even for staff
}
register_feature_flag!(HandoffFeatureFlag);
```

The flag gates automatic creation of new compaction markers, threshold checks, compaction
requests, and suppression of the existing token-limit callout. Existing persisted
`Message::Compaction` entries are always loaded, rendered, and honored by request building,
regardless of the flag. Threads without compaction messages behave exactly as they do today.

## Architecture

Two thread types, two responsibilities:

- **`agent::Thread`** (`crates/agent/src/thread.rs`) — the model-facing data model. This is
  where `Message::Compaction` lives and where request building scans for the latest compaction
  boundary.
- **`acp_thread::AcpThread`** (`crates/acp_thread/src/acp_thread.rs`) — the UI-facing
  transcript. Stays a flat `entries` list; gets a divider marker entry when replaying or
  receiving a `Message::Compaction`.

`agent::Thread` emits `ThreadEvent`s, which `NativeAgentConnection::handle_thread_events`
(`crates/agent/src/agent.rs:1832`) translates into `AcpThread` mutations.

## Data model

### `agent::Thread`

```rust
// crates/agent/src/thread.rs
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum Message {
    User(UserMessage),
    Agent(AgentMessage),
    Resume,
    /// A durable transcript boundary. Renders as a divider in the UI and provides
    /// compacted prior context for model requests after this point.
    Compaction(CompactionInfo),
}

pub enum CompactionInfo {
    /// Generic providers: readable summary, rendered as a user-role context item.
    Summary(SharedString),
    /// OpenAI parity: opaque compacted window from `/responses/compact`, replayed
    /// verbatim through a native-capable Responses adapter.
    ProviderNative {
        provider: LanguageModelProviderId,
        items: Vec<serde_json::Value>, // `compacted.output`, passed as-is
    },
}

pub struct Thread {
    messages: Vec<Message>,
    request_token_usage: HashMap<UserMessageId, language_model::TokenUsage>,
    // …everything else unchanged…
}
```

- `Message::Compaction` is a first-class transcript item. It is serialized in
  `DbThread.messages`, replayed into the UI, and included in markdown as a
  `--- Context Compacted ---` separator.
- It is not a user checkpoint target and does not participate in `request_token_usage`.
- `Message::Compaction::to_request()` can render `CompactionInfo::Summary` as a synthetic
  user-role context message, matching Codex's generic compaction shape. `role()` may return
  `Role::User` for `Message::Compaction` because the generic text artifact is sent as a user
  message.
- `CompactionInfo::ProviderNative` is not representable as ordinary
  `LanguageModelRequestMessage`s. Request building carries its `items` through a native-prefix
  path such as `LanguageModelRequest.provider_native_prefix` and only for compatible providers.

### `acp_thread::AcpThread`

Flat `entries` stays; add a marker variant:

```rust
// crates/acp_thread/src/acp_thread.rs (enum at L178)
pub enum AgentThreadEntry {
    UserMessage(UserMessage),
    AssistantMessage(AssistantMessage),
    ToolCall(ToolCall),
    CompletedPlan(Vec<PlanEntry>),
    ContextCompaction,        // renders the divider; rewinds like CompletedPlan
}
```

`Thread::replay` emits `ContextCompaction` entries directly from `Message::Compaction`.
Because the marker is persisted in `DbThread.messages`, UI dividers are not inferred from
hidden replacement-history state.

## Threshold detection

Source of truth after a successful request is the provider-reported running usage
(`UsageUpdate`, `thread.rs:2386`). V1 does not add a new local/tokenizer estimate before provider
usage exists; if no provider usage is available for the latest user message, auto-compaction does
not trigger yet.

Thresholding is deliberately simple in v1: compact when reported active context leaves only a fixed
remaining-token budget in the selected model's context window. We do not subtract carried compacted
context. The default remaining-token budget is `40_000` tokens; a temporary environment override may
exist for local testing while the feature is behind the `handoff` flag.

Checked at Codex's two points in `run_turn_internal` (`thread.rs:2044`): **pre-turn** (before
the first request) and **mid-turn** (the `intent = ToolResults` branch, ~L2248).

```rust
fn should_auto_compact(&self, cx: &App) -> bool {
    if !cx.has_flag::<HandoffFeatureFlag>() { return false; }
    let (Some(usage), Some(model)) = (self.latest_request_token_usage(), self.model.as_ref())
        else { return false };

    let active = total_input_tokens(usage).saturating_add(usage.output_tokens);
    let limit = model
        .max_token_count()
        .saturating_sub(AUTO_COMPACT_REMAINING_TOKEN_BUDGET);
    active >= limit
}
```

## Per-provider compaction strategy

```rust
// crates/language_model
enum CompactionStrategyKind { Native, GenericSummary }
fn LanguageModel::compaction_strategy(&self, cx: &App) -> CompactionStrategyKind;
```

### OpenAI — `Native` (Codex parity)

`POST {api_url}/responses/compact` (base API URL + route, not a full endpoint setting) with
the current model-visible window. Do not pre-prune retained user messages before the compact
call; only perform conservative request-fit trimming if the compact request itself would exceed
the window. Response `compacted.output` = retained user messages + one opaque compaction item.
Per the docs: pass it to the next `/responses` call **as-is**, **do not prune**. Store it
verbatim as `CompactionInfo::ProviderNative` with the provider that produced it.

Requires Zed's Responses API path (`crates/open_ai/src/responses.rs`); the compaction item is
not representable in Chat Completions. Native compaction is available only for first-party
OpenAI/Azure or providers with an explicit native-compaction capability; do not infer support
from "OpenAI-compatible" alone. Every subsequent `/responses` request that replays native
compaction items must use `store=false` where the endpoint supports it.

### Everyone else — `GenericSummary`

A normal completion with `CompletionIntent::ThreadContextSummarization` +
`SUMMARIZE_THREAD_DETAILED_PROMPT`, collected to text (reuses the machinery in
`Thread::summary`, `thread.rs:2748`) → `CompactionInfo::Summary`.

The summarization input is the current model-visible window, not the full historical transcript
and not only messages after the latest compaction marker. That means request construction for
summarization uses the same helper as normal model requests:

- latest compatible native compaction info or generic summary compaction info,
- dynamically retained prior user messages for generic summaries,
- messages after the latest compaction marker,
- summarization prompt.

## Request building & layout

`build_request_messages` (`thread.rs:3154`) becomes boundary-aware:

1. Scan backward in `Thread.messages` for the latest `Message::Compaction`.
2. If none exists, build requests from all normal messages as today.
3. If a compaction marker exists, build the model request from that marker plus the suffix after
   it. Messages before the marker are retained for UI/checkpoint restore, but are not sent as
   normal history.

For generic summary compaction:

- Dynamically derive retained user messages by scanning backward before the latest
  `Message::Compaction`, newest-first, until the fixed retention budget is reached. Then render
  the selected messages in chronological order.
- Render `Message::Compaction::to_request()` as the user-role summary context message.
- Append normal messages after the marker.

Layout:

```text
system
retained prior user messages, derived from the flat log
user-role compaction summary message
normal messages after the compaction marker
pending/live input
```

This mirrors Codex's generic replacement-history role/order: retained user messages followed by
a user-role summary.

For native OpenAI compaction:

- If the active provider can consume the native compaction info, splice `items` verbatim as the
  native request preamble, then append normal messages after the marker.
- If the active provider cannot consume the native compaction info, do not replay the opaque blob.

## Handoff flow (turn-loop integration)

When `should_auto_compact` trips:

1. Emit `ThreadEvent::CompactionStarted` → UI shows the live "Compacting Context…" spinner
   divider.
2. Run the model's `CompactionStrategy` (native endpoint or local summary).
3. Append a new `Message::Compaction(info)` to `Thread.messages`:
   - Native: `info = ProviderNative { provider, items: compacted.output }`.
   - Generic: `info = Summary(text)`.
4. Emit `ThreadEvent::CompactionSucceeded` → push `AgentThreadEntry::ContextCompaction`,
   clear the spinner. On failure/cancellation, emit a terminal compaction event that clears the
   spinner without adding the marker.
5. The turn loop continues; the next `build_completion_request` scans back to this marker and
   sends only the new model-visible window.

Pre-turn and mid-turn timing:

- **Pre-turn**: compact the existing transcript before recording the newly submitted user
  message. After compaction commits, append/send the new user message as fresh live input after
  the compaction marker.
- **Mid-turn (`ToolResults`)**: record/flush completed tool results first, then compact before
  the next model request. The compacted summary must account for those tool results. Do not send
  orphan `ToolResult` request content after compaction; if a provider request still needs
  concrete tool-result objects, pin the minimal assistant-tool-call/tool-result suffix verbatim
  until the provider protocol is satisfied.

## Checkpoint restore across compaction boundaries

Today `AcpThread::restore_checkpoint(id)` (`acp_thread.rs:2557`):

1. `rewind(id)` → `connection.truncate` → `agent::Thread::truncate(message_id)` (drops the
   message + everything after, rejects action-log edits, emits `EntriesRemoved`), and
2. restores the git worktree to the `GitStoreCheckpoint` stored on that user message.

With a flat `Message::Compaction` log, this remains straightforward:

- `Thread::truncate(message_id)` searches `Thread.messages` for the target user message and
  drains from that index onward.
- Any later `Message::Compaction` entries are removed naturally by the drain.
- If the target is after a compaction marker, that marker remains and continues to provide
  compacted context for future requests.
- If the target is before a compaction marker, the marker is removed and future requests fall
  back to the earlier model-visible window.
- The `EntriesRemoved` range already covers visible divider entries. Git restore is unchanged
  because checkpoints remain per user message.

## Provider strategy + retention budgets

### Native compaction capability

Do not add a separate full `compaction_url` setting in v1. Native compaction uses the provider's
normal Responses API base URL plus the fixed `responses/compact` route, matching Codex's base
URL + route construction. Values ending in `/responses` or `/responses/compact` should not be
interpreted as base URLs.

Native compaction is capability-gated:

- First-party OpenAI Responses API: `Native`.
- Azure Responses-compatible OpenAI: `Native` if the adapter supports the same item contract.
- Providers with an explicit future `supports_native_compaction` capability: `Native`.
- Other OpenAI-compatible or Responses-compatible providers: `GenericSummary`.

A `ProviderNative` compaction stores the provider that produced it. If the active provider is
not compatible, the request builder must not replay the opaque native blob.

### Retention budgets — path-specific constants

Codex uses path-specific retention rather than a per-known-model table. For v1, mirror that:

- **Generic summary**: retain recent user messages as model-only context up to a fixed `20_000`
  token budget, newest-first; truncate the oldest included text to fit and preserve images when
  possible. This budget is not persisted on each compaction marker.
- **Native OpenAI `/responses/compact`**: do not apply a client-side retained-message budget
  before the compact call; trust `compacted.output` from the endpoint. If Zed later implements a
  local native-v2-style reconstruction path, use Codex's `64_000` retained-message budget there.

Do not expose `compaction_retained_tokens` in v1. If tuning is needed later, make it
model-scoped metadata or a global/session override, not provider transport configuration.

## Auto-compaction settings

Do not expose user-facing auto-compaction settings until the end of the rollout, after the generic
handoff, UI, and native OpenAI paths have landed. Until then, automatic creation of new compaction
markers is controlled only by the `handoff` feature flag and internal constants.

If settings are still needed after the core behavior stabilizes, add them as a final PR. Prefer a
minimal `enabled` setting first; avoid exposing threshold tuning unless there is a demonstrated need.
Thresholds should not be expressed as a user-facing percentage of the context window.

No manual `/compact` action in v1.

## UI

- **Static divider**: `render_entry` arm (`crates/agent_ui/src/conversation_view/thread_view.rs`,
  ~L4863) → horizontal `Divider` (`crates/ui/src/components/divider.rs`) with a centered
  `Label("Context Compacted")`, modeled on the existing subagent-output separator (~L5158).
- **Live "Compacting Context…"**: a transient `compacting` flag drives a trailing row via the
  existing `sync_generating_indicator` mechanism (~L5729), rendering
  `GeneratingSpinnerElement::new(SpinnerVariant::Sand)` + `LoadingLabel::new("Compacting Context")`,
  replaced by the static entry on completion.
- **Events**: new compaction lifecycle events (`thread.rs:687`) handled in
  `handle_thread_events` (`agent.rs:1832`). Start sets the flag; success pushes the explicit
  `ContextCompaction` marker and clears the spinner; failure/cancellation clears the spinner and
  surfaces the turn error without adding a divider.
- **Gate the existing callout (do not delete)**: `render_token_limit_callout` /
  `NewNativeAgentThreadFromSummary` (`thread_view.rs:9288`) stays when auto-compaction creation is
  not enabled. When `HandoffFeatureFlag` is enabled, suppress it because auto-compaction supersedes
  it. It must not be removed outright.
- Existing `Message::Compaction` entries render regardless of the feature flag.
- Clear the live compaction indicator on any terminal turn state (success, failure, or
  cancellation), not only on a matching success event.

## Persistence & migration

Keep the existing blob-based persistence model. There is no SQLite join table and no new
conversation table.

`DbThread.messages: Vec<DbMessage>` (`crates/agent/src/db.rs:56`) remains the thread payload and
now serializes the new `Message::Compaction` variant directly. Threads that never create a
compaction marker remain byte-shape-compatible with current behavior. Threads that do contain
`Message::Compaction` require a Zed version that knows that enum variant.

`SharedThread` export/import can continue using the flat `messages` payload. Existing
`Message::Compaction` entries render and are honored on import; provider-native blobs are replayed
only if the active provider can consume them.

## Edge cases & chosen behavior

- **Model switch across a boundary**: `ProviderNative` compaction info is provider-scoped. If the user
  switches the current thread to a provider that can't consume it, do not replay the opaque native
  blob. Continue with post-handoff visible messages only and surface a warning if this loses
  compacted context; the next threshold hit re-compacts with the new provider's strategy. We never
  replay an OpenAI blob through a non-native provider.
- **Restore into a prior boundary**: supported (see Checkpoint section). In-place editing of a
  message before a compaction marker is not supported in v1.
- **Compaction failure**: on error, emit the error, leave the thread uncompacted, clear the live
  compaction indicator, and do not add a divider. With the flag enabled, do not fall back to the
  old token-limit callout; show a distinct compaction failure / turn error instead.
- **Cancellation**: compaction install is an atomic commit. If the user cancels before commit,
  discard late compaction results and leave history unchanged. If commit already happened before
  cancellation was accepted, keep the committed handoff.
- **Telemetry**: dedicated compaction telemetry can land after the first functional implementation.
  When added, mirror Codex's terminal event fields (`trigger`, `reason`, `phase`, `implementation`,
  `strategy`, `status`, before/after active-context tokens, duration). No session/provenance
  system.

## Implementation sequence (PR-sized)

Each PR is independently reviewable. The feature flag has already landed. The generic path becomes
functional in PR 3, and OpenAI parity lands later; all automatic creation of new compactions remains
behind the flag.

1. **`Message::Compaction` data model + replay (no auto behavior)** — add the enum variant,
   `CompactionInfo`, summary rendering, markdown rendering, DB serde, and ACP replay/rendering for
   existing markers. Existing threads without markers behave unchanged.
2. **Request-window helpers + checkpoint restore** — build current model requests by scanning to
   the latest compaction marker; derive generic retained user messages on the fly; make truncate
   naturally remove later markers and verify checkpoint restore across a synthetic marker.
3. **Threshold detection + generic compaction strategy + handoff (end-to-end generic functional)** —
   total-context `should_auto_compact` based on provider-reported usage and the fixed
   remaining-token budget; pre-turn/mid-turn hook points; `GenericSummary` path; append
   `Message::Compaction(CompactionInfo::Summary(...))`; Codex-aligned retained-users-then-summary
   layout; failure/cancellation handling. Auto-compaction works for all providers via local
   summarization, behind the flag. No user-facing settings in this PR.
4. **UI polish** — live "Compacting Context…" spinner; gate (not delete) the token-limit callout
   under the flag; clear the spinner on success/failure/cancellation. (May land with PR 3 if we
   want it visible on first functional ship.)
5. **OpenAI native deep path (parity payoff)** — `/responses/compact` integration via the
   Responses API base URL + fixed route; `ProviderNative` compaction info with provider metadata;
   `LanguageModelRequest.provider_native_prefix` + verbatim replay only in native-capable
   adapters; `store=false` on subsequent `/responses` replay requests; no pre-pruning before
   compact; provider/capability-scoped handling.
6. **Deferred user-facing settings, if still needed** — add minimal settings only after the core
   generic, UI, and native paths have landed. Keep settings out of earlier PRs to avoid exposing
   knobs before behavior is stable.

Suggested validation: unit tests for threshold math and retention truncation; a fake-model handoff
test asserting the current-window request shape; a checkpoint-restore test across a synthetic
`Message::Compaction`; cancellation/failure tests asserting no marker is added and late compaction
results are discarded; an `insta` snapshot for the divider + spinner; a mock `/responses/compact`
round-trip asserting verbatim replay only for native-capable providers and `store=false` on
subsequent `/responses` requests.

## Appendix A — Why no subagent / session-source concept

Codex tags compaction traffic as a subagent (`SubAgentSource::Compact`, `x-openai-subagent`) only
when compaction runs as its _own session_; the inline path reuses the parent client and is
distinguished server-side purely by hitting the `/responses/compact` endpoint. That tagging exists
for Codex-backend concerns — thread-list hygiene (hiding spawned sessions from the user's history),
telemetry, and billing/rate-limit attribution. None of that applies to us: we run compaction inline
on the existing thread, talk to an arbitrary provider endpoint, and render the result as an inline
divider. So we carry no session-source machinery; compaction is tagged only by a completion intent
for local telemetry.

## Appendix B — Reference: OpenAI public compaction API

<https://developers.openai.com/api/docs/guides/compaction> documents two public mechanisms on the
standard Responses API (usable with an API key; subsequent replay requests must preserve
ZDR-friendly `store=false` behavior where supported):

1. **Server-side**: set `context_management` with `compact_threshold` on `POST /responses`; the
   server compacts at the threshold and emits the encrypted compaction item in-stream.
2. **Standalone**: `POST /responses/compact` returns a compacted window (retained user messages +
   one opaque compaction item) to pass into the next `/responses` call as-is.

We use the **standalone endpoint** (#2) because it matches our "detect threshold → compact → append
`Message::Compaction`" architecture. Corroborated by the official `openai-python` SDK types
`ResponseCompactParams` / `CompactedResponse` (`object: "response.compaction"`, `output` = "all
user messages, followed by a single compaction item").
