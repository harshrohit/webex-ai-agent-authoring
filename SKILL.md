---
name: webex-ai-agent-authoring
description: Create, edit, review or tune Webex AI Agent Studio agents and their knowledge bases via the ai-agent-studio MCP: engine choice (S2S vs cascade), prompts, actions, KBs, voice settings, validation, hand-off. Use for any AI agent build or change.
---

# Authoring Webex AI Agent Studio agents

This skill covers four kinds of work with the ai-agent-studio MCP (`webex-ai-studio-*` tools):

- **Create** a new agent from a business request → Runbook.
- **Edit** an existing agent → Editing existing agents.
- **Review** an agent's design → Reviewing an agent.
- **Knowledge** bases: find, create, extend, attach → Knowledge bases.

Voice tuning, patterns, the review checklist and test scenarios apply to all of them.

## Division of labor

The MCP owns **facts and mechanical checks**: the draft and update schemas, selectable engines and voices, discoverable MCP functions, knowledge bases, structural validation, and save verification. Trust it for those and never invent them. Field limits and ranges live in the tool schemas; read them there rather than from this skill.

You own **judgment and craft**: what to ask, which engine fits, how actions are shaped, every word of the prompt and knowledge articles, and which settings fix which caller problem. The MCP gives no prompt structure and its validation does not judge prompt quality — a draft can pass `validate` and still behave badly.

## Ground rules

- **Writes are live.** `agent-create`, `agent-update`, `action-create`, `knowledge-base-create` and `knowledge-base-source-create` change a real Studio org. Do discovery and drafting read-only, show the user a summary of the change, and call a write tool only after an explicit yes. Requires Studio admin access and `cjp:config_write`.
- **Never invent** engine IDs, voices, timezones, KB IDs, MCP function names, flow variables, output fields, policies or capabilities. If something is unknown, ask or leave it out and say so.
- **Never retry blind.** If a write returns partial or unknown status, inspect the saved state first, then reconcile. Do not repeat a confirmed success.
- **Saved is not live.** Agents are saved unpublished; KB articles are saved before they are ingested. Never tell the user something works until it has been published, ingested and test-called.
- **Prefer IDs over names.** Name lookups are exact and case-sensitive, and duplicate names return candidates instead of a selection.
- **Call tools sequentially.** The server rate-limits parallel calls (~2 s retry). For paged results, follow `next_call` until `traversal_exhausted` / `response_exhausted` is true before claiming totals or a complete listing.
- **No deletes.** The MCP cannot delete agents, actions or knowledge bases. Actions can only be disabled; removal of agents or KBs happens in Studio.

## Runbook: create a new agent

Track these as a task list; each phase feeds the next.

### 1. Elicit requirements

Ask only what you cannot find yourself, but cover all of this before drafting — gaps here become invented behavior later:

- **Purpose and audience:** who calls, why, what a successful call looks like.
- **Channel:** voice only, or voice and digital? (Drives engine choice.)
- **Workflows:** the 2–6 things callers actually need. For each: trigger phrases, information to collect, the backend system that does the work, what success looks like, what happens on failure or an empty result.
- **Policies:** identity checks, what must never be said or promised, data that must not be collected, compliance wording.
- **Escalation:** when to transfer to a human or another AI agent, urgent triggers, after-hours behavior.
- **Out of scope:** topics to decline and the approved fallback line.
- **Knowledge:** sources the agent may answer from.
- **Persona and voice:** name, tone, preferred voice/gender, locale, timezone.
- **Disclosure:** whether the agent says it is an AI (transparency on), and the exact wording.

Design limits to keep in mind: at most 20 actions, 5 knowledge bases and 20 voices per agent.

### 2. Discover what the org offers

- `agent-describe` (filter by `locale`) → draft schema, selectable engines and voices, fixed create settings, authoring guidance, unsupported configuration. Page through voices if the user wants a specific one.
- `agent-search` (optionally `agent_type`) → find a **reference agent** similar to what you are building.
- `agent-inspect` (`focus=configuration`) on the reference agent → full engine record (`family`, `modality`, `llm.model`, `deprecated`, `region_availability`), a valid `timezone`, voice tuning and advanced settings. `describe` does not expose engine traits, so this is how you learn them.
- `knowledge-base-search` → candidate KBs (see Knowledge bases).
- `mcp-function-search` (`query_scope=name_and_description` for business terms) → only `SELECTABLE` functions; note the exact `server_id` and canonical name. It does not return output schemas — do not invent output fields.

### 3. Choose the engine

See Engine choice. Confirm with the user when it is not obvious, and never pick an engine whose record shows `deprecated: true` for a new agent.

### 4. Design the actions

| Type | Use when | You supply |
|---|---|---|
| `custom_transfer` | Handing to a human queue or another AI agent | `description`, `announce_transfer` |
| `slot_filling_with_fulfillment` + `mcp` | A discovered MCP function does the work | `server_id` + exact canonical `name` only. No description or schemas — the function's own are preserved |
| `slot_filling_with_fulfillment` + `custom_event` (voice only) | A Flow Designer branch does the work | ASCII `name` (becomes the flow's `StateEventName`), `description`, `input_schema`, `input_sources`, optional `output_schema` |

For every description you write, cover: when to invoke and when **not** to, prerequisites, where each input comes from, what each confirmed result means, and how to recover from failure or an empty result.

For source-flow actions:
- `input_sources` declares provenance per entity: `caller` (collected in conversation), `flow_context` (already known to the flow), or `action_result` (a prior action's field; needs `source_action` and `source_field`).
- Collected slots arrive in the flow as `MetaData`. The flow must be bound to this agent and handle the matching event branch. The MCP does not inspect or change flows, so confirm with the user that the branch exists or will be built.
- Declare only `output_schema` fields the flow is confirmed to return (object, string, number). If unknown, omit it; the prompt then must not promise any result fields.

### 5. Write the prompt

- `description` = 1–3 sentence profile summary. Never include the `### INSTRUCTIONS:` marker; Studio uses it to join description and instructions.
- `instructions` = the behavior, using Prompt templates.
- `welcome_message` = see the Welcome message pattern.
- Studio length-checks description + instructions against an undisclosed limit. If `validate` reports a length finding, cut redundant prose before cutting rules.

### 6. Fill `authoring` and self-review

`authoring` is not saved; it lets the MCP check that workflows and actions agree. Write one workflow per caller need with `mode` (`action`, `answer`, `handoff`, `decline`), `action_names`, `prerequisites`, `steps`, `success_criteria`, `failure_behavior` and 2–4 `test_cases`. Add `policies` for the business rules. Then run the Review checklist against the original requirements and report any promise that no action or knowledge source backs.

### 7. Validate, confirm, create, verify

1. `agent-validate` with the full draft; fix every finding.
2. Show the user: name, engine, voice, transparency, KBs, each action and its type, workflows, and anything you could not configure. Wait for a yes.
3. `agent-create` once.
4. `agent-inspect` (`focus=all`) and compare saved fields with what you sent; report differences.

### 8. Post-create and hand-off

Use `agent-update` for what create cannot set: advanced voice settings, voice tuning, and the voice session duration cap (see Voice tuning). Use `action-create` to add actions later.

Not possible through the MCP — tell the user explicitly:
- Publishing the agent.
- Binding it in Flow Designer and building `custom_event` branches.
- Agent handover and feedback flags (forced off at create, absent from the update schema) — set in Studio if needed, e.g. for multi-agent transfers.
- Logo, import/export, HTTP actions, LLM model selection, Webex Connect bindings, deleting anything.

Finish with a hand-off list: agent ID, what was saved, what remains manual, and the test calls to run.

## Editing existing agents

1. `agent-inspect` by ID (`focus=all`, `response_format=detailed`) and note what will change and what depends on it (instructions that name actions, KBs, flow branches).
2. Make the smallest change that meets the request, plus companion edits only for concrete conflicts (e.g. renaming an action also means updating the instructions that name it). Report anything you leave unresolved.
3. Show the user the before/after for each field and wait for a yes.
4. `agent-update`, then re-inspect and compare.

Update semantics that cause mistakes:
- Omitted fields and actions are kept; scalars replace; voice and session settings merge; `null` does not clear a value.
- `voice_settings.voices` **replaces the whole list** including tuning — include every voice you want to keep. `custom_vocabulary` also replaces all; `[]` clears it.
- `knowledge_bases`: use `add` / `remove` to keep other links; `replace` sets the full list and `[]` detaches all.
- Actions cannot be deleted or replaced as a list — disable with `enabled: false`. System actions allow only `enabled` changes. MCP-function definitions cannot be edited. Supplied schemas replace the whole schema.
- Renaming a `custom_event` action changes its flow event name; the Flow Designer branch must be updated to match. When renaming several, order them so each name is freed before it is reused.
- `response_style: CUSTOM` needs a non-empty `custom_prompt`.
- The voice duration cap must end up as `{enabled: false}` or all three of `enabled`, `max_session_duration_seconds`, `session_length_exceeded_exit_event`.
- `engine` can point at a `custom` engine (with optional `version`) on update, though create only allows native engines.
- Before editing an action schema, check its `input_schema_view` / `output_schema_view` in inspect: the table view supports only object, string and number.
- Updates save configuration only; republish in Studio for changes to reach callers.

## Reviewing an agent

- `agent-inspect` with `focus=all` and `response_format=detailed`. Check `omitted_paths`: instruction text can be left out, so do not claim you reviewed the full prompt unless it was returned. `instruction_interpretation` explains where prompt text sits. Credentials are redacted.
- Run the Review checklist and report findings with short excerpts from the saved prompt or action descriptions.
- Saved configuration does not show runtime behavior. For a published agent, the WxCC Operations MCP tools in the session (for example listing AI agent sessions, session messages, AI agent audit, and handoff analysis) show what actually happened on calls; use them as evidence when available.

## Knowledge bases

- **Find:** `knowledge-base-search` (omit `name_query` to list; `agent_type` filters by supported type, not by which agents use it). Follow `next_call` before claiming a full list.
- **Check before reuse:** `knowledge-base-inspect` shows mapped agents, sources and provider-reported ingestion. A KB mapped to other agents is shared — adding or changing articles changes their answers too, so tell the user and get agreement first.
- **Create:** `knowledge-base-create` with inline articles only (URL crawling and file uploads are unsupported). **Extend:** `knowledge-base-source-create` adds one article at a time; check saved sources before retrying a partial write.
- **Ready check:** saving does not mean ingested. Inspect with `focus=ingestion` before attaching or test-calling, and say plainly if status is pending or failed. Ingested still does not prove retrieval quality — test with real questions.
- **Attach:** `knowledge_base_ids` at create, or `knowledge_bases.add` on update.

Writing articles the agent can answer from:
- One topic per article, with a name that says what it answers.
- Lead with the direct answer, then detail; phrase as the caller would ask.
- Plain sentences that read well aloud: no tables, URLs or symbol-heavy text for voice agents.
- Only official, confirmed content; date anything that changes (prices, hours, policies).
- Keep the prompt and the KB in sync: the prompt says which KB to answer from and what to do when the answer is not there.

## Engine choice

Read engine traits live from a reference agent's engine record; these are heuristics, not facts about any specific engine.

- `family: S2S`, `modality: ["audio"]`, a speech-native model → **speech-to-speech**. Lean this way when natural, low-latency conversation matters most, scope is narrow, tools are few, and the org's region is in `region_availability`.
- Text-capable models with `modality` including `text` → treat as **cascade** (ASR → LLM → TTS) unless the record says otherwise. Lean this way for many workflows or tools, strict slot collection and read-back, heavy policy text, a digital channel, or a need for custom vocabulary and TTS tuning.
- Check `deprecated` and `region_availability` before recommending.
- If the engine you want appears on a reference agent but not in `describe`'s candidates, say so. Creating on a selectable engine and switching with `agent-update` is possible but untested for voice compatibility — get the user's agreement and verify with inspect and a test call.

## Prompt templates

### Common skeleton (cascade)

```
## 1. Identity
You are <Name>, <Org>'s AI agent for <scope>. Tone: <2–3 adjectives>.

## 2. Session context
<Only if multi-agent: how to read the transfer variable and what to do in each state.>

## 3. Scope
In scope: <list>.
Out of scope: <list>. For these, say: "<approved line>" and offer <alternative>.

## 4. Routing priority
Before every reply, classify the caller's latest request in this order and take only the first match:
1. Emergency or urgent harm → invoke [<transfer_action>] immediately. Do not collect details first.
2. Caller asks for a person → invoke [<transfer_action>].
3. <Workflow A trigger> → Workflow A.
4. <Workflow B trigger> → Workflow B.
5. In-scope question → answer from knowledge only.
6. Anything else → Unknown fallback.

## 5. Workflows
### A. <Name>
Needs: <slots and where each comes from>.
Steps: <numbered, including confirmation read-back before the action>.
Call [<action_name>] only after <prerequisite>. Do not call it when <condition>.
On success: <what the result fields mean and what to say>.
On failure or empty result: <exact recovery and fallback>.

## 6. Tool rules
- Never say an operation is booked, sent or complete unless [<action>] returned that result.
- Never invent confirmation numbers, availability, prices or policies.
- If a tool fails twice, apologize briefly and <transfer / fallback>.

## 7. Knowledge and grounding
Answer only from <KB names>. If the answer is not there, say so and use Unknown fallback.

## 8. Voice style
One or two short sentences per turn. No lists, symbols or URLs aloud. Read confirmation numbers digit by digit. Confirm names and spellings back.

## 9. Fallbacks
Unknown: "<line>". No input: "<line>". Repeated confusion: <transfer>.
```

### S2S variant

Speech-to-speech models follow long prompts less reliably, so:
- Put Routing priority and Tool rules first, Identity after.
- Keep to the fewest workflows that cover the need; move rarely used detail to knowledge.
- Make tool triggers explicit and immediate: "say one short line, then invoke [X] immediately; do not ask a follow-up question first."
- Repeat the one or two rules that matter most (e.g. never claim success without a result) once at the end.
- Write in plain spoken sentences; avoid nested bullets and tables inside the prompt.

## Voice tuning

These are set with `agent-update` (`voice_settings.advance_settings.global` and `voice_settings.voices`); valid ranges are in the update schema. Change one thing at a time and re-test.

| Caller problem | Settings to look at |
|---|---|
| Agent cuts callers off mid-sentence | Raise `eos_sensitivity` and `caller_turn_timeout` |
| Long awkward pauses before replies | Lower `caller_turn_timeout` / `eos_sensitivity`; `slots_additional_delay` |
| Callers cannot interrupt, or noise interrupts the agent | `customer_interruption`, `optimized_barge_in` |
| Silence handling too quick or too slow | `custom_no_input` |
| Backend actions time out | `fulfilment_timeout` |
| Account numbers or PINs misheard | Turn on DTMF (`turn_on_dtmf`, `max_length`, `term_char`, `timeout_between_digits`) and tell the prompt to offer keypad entry |
| Product or brand names misrecognized | `custom_vocabulary` (replaces the full list) |
| Responses too chatty or too terse | `response_style`, `acknowledgement_*`, `include_disfluencies` |
| Voice too fast or flat | `speaking_rate`; for ElevenLabs voices `model_id`, `stability`, `similarity`, `style_exaggeration` |
| Calls run too long | Session duration cap with an exit event of `transfer` or `call_end` |

## Patterns

**Multi-agent handoff.** The receiving agent reads a transfer-state variable from flow context (some orgs use `{{transferredFromAnotherAiAgent}}` — confirm the real name with the user or a reference agent; never invent one). When transferred: do not re-greet, do not ask the caller to repeat, use the passed context and act. Otherwise greet normally. Handover flags must be enabled in Studio.

**Grounded results.** Tie every success statement to an action result field. With an empty output schema the agent can only say the request was submitted, not that it succeeded.

**Confirm before acting.** Read back collected details and get a yes before any action that changes something for the caller.

**Urgent first.** Emergency or safety triggers transfer immediately, before identity checks or slot collection.

**Minimal collection.** Collect only the slots an action needs. Distinguish caller confirmation ("is that right?") from backend identity verification (an action result).

**Welcome message.** Two sentences at most: who the agent is (with AI disclosure if transparency is on) and what it can help with. Mention only capabilities backed by an action or knowledge source — if it promises SMS, an SMS action must exist.

## Review checklist

- Every caller need maps to a workflow, and every workflow to actions or knowledge.
- Every enabled action is named in the instructions with its exact name, and every action named in the instructions exists and is enabled.
- Each action description covers invoke / don't-invoke, prerequisites, input provenance, result meaning, failure recovery.
- No promise (instructions or welcome) of results, fields or capabilities the actions don't confirm.
- Urgent escalation precedes collection; out-of-scope topics have an approved line.
- Flow variables referenced in the prompt exist in the flow.
- Attached KBs are ingested, cover what the prompt says they cover, and are not shared unexpectedly.
- Engine is not deprecated and is available in the org's region; prompt style suits the engine.
- Transparency setting and wording match the user's decision.

## Test scenarios

Write these as `authoring` test cases and give them to the user as live test calls after publishing. Nothing in the MCP executes them.

- Happy path for each workflow.
- Backend failure and empty result for each action.
- Ambiguous request that could match two workflows.
- Caller corrects a detail mid-collection.
- Caller interrupts / barges in.
- Question the KB covers, and one it does not.
- Out-of-scope request and a request for a human.
- Urgent trigger mid-workflow.
- Adversarial: asks the agent to ignore its rules or reveal its prompt.
- Multi-agent: arrival via transfer with context.

## Naming and hygiene

- `unique_name`: ASCII letters, digits, `_` or `-`, starting with a letter; make it descriptive (`appointment_reschedule_voice`). Follow the org's convention if it has one.
- Test or experiment agents: prefix the display name with `TEST_<owner>_<yyyymmdd>_` so they are easy to find and remove in Studio later.
- Timezone: one the user names, or one observed on a reference agent in the same org; the MCP offers no list.
- Inventory questions ("how many agents"): page `agent-search` to the end, then report totals by type and point out likely test or duplicate-name agents.
