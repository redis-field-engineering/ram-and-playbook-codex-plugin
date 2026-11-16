# Console clicks you can replace with the `iris` CLI

Most README setup steps are console clicks. Many of them already have an
`iris` command. The CLI ships with the pinned `redis-iris` (1.1.0) in this
repository's `.venv`, so `.venv/bin/iris` is there after the first setup, with
nothing extra to install. Set `IRIS_CONFIG=iris.toml` (this repository's
`kind = "cloud"` file). Control-plane commands then print the console steps
instead of failing.

For a step-by-step run of every **Today** row, see
[demo_script_cloud.md](demo_script_cloud.md).

This is a running list. The status shows what works against Redis Cloud
**today**:

- **Today**: works now with the keys this demo already uses (data plane).
- **When Cloud sign-in ships**: the command exists, but on Redis Cloud it needs
  the CLI's control-plane sign-in, which isn't released yet. It works today
  against self-managed Iris (`--local` or a service token).
- **No command yet**: still console only. Listed for the Iris team.

The `set-up-demo` agent skill (`.agents/skills/set-up-demo/`) already uses
every **Today** row.

## Setup (README Parts 2 and 3)

| README step / console click | `iris` command | Status |
| --- | --- | --- |
| Step 3: **Create Agent Memory service**, name it, **Create**, copy endpoint and store ID from **Connect** | `iris agent-memory service create --name real-reaction-demo --write-env editor.env` creates the store, mints the key and writes `IRIS_AGENT_MEMORY_URL`, `_STORE_ID`, `_API_KEY` | When Cloud sign-in ships |
| Step 4: **Configuration → Edit → Add type**, then **Add field** ×9 with type and description | Same `service create`, with `--from service.yaml` and a `memory_types:` block (below). Creates the store and the type together. | When Cloud sign-in ships (field descriptions: see gaps) |
| Step 5: **MemoryETL → New extractor**, paste prompt, copy `ex_…` from the REST example | None. The control-plane API has `POST /stores/{id}/extractors`, but the CLI has no command. | No command yet |
| Step 6: **Create API key**, read/write | `iris agent-memory service key <store-id> --write editor.env` | When Cloud sign-in ships |
| Step 7.1: `package_skill.py`, then keep its output open to paste | `iris agent-playbooks skills validate dist/skills` validates and packages offline. Intents and attributes live in `.iris-playbook.json`, so nothing is pasted. | **Today** |
| Steps 7.2–7.4: **New Playbook**, name, **Create**, copy endpoint and playbook ID | `iris agent-playbooks service create --name real-reaction-agenda --write-env editor.env` | When Cloud sign-in ships |
| Steps 7.5–7.6: **Registry → Skills → Upload Skill**, choose ZIP, **Optional settings**: alias, 5 intents, filter JSON | `iris agent-playbooks skills push dist/skills/prepare-management-review-agenda` (needs a content key) | **Today** |
| Step 7.7: **Publish version → Publish** | Add `--publish all` to the `push` above, or run `iris agent-playbooks skills publish prepare-management-review-agenda --version latest` | **Today** |
| Step 8: **Connect → Create an API key → Customize in the full page**, untick **full**, **Create key** | `iris agent-playbooks service key <pb-id> --actions read --write editor.env` mints a read-only key without the full-page workaround. `--role content` mints the publish key. | When Cloud sign-in ships |
| Steps 6 and 8: two separate key dialogs | Console, but fewer clicks: **Iris → Access & Identity → New API key** takes one grant per resource. One key (store read/write, Playbook read) covers the plugin, with no full-page workaround. The CLI's Cloud refusal prints these steps. | **Today** (console) |
| Step 9: copy 7 values from 3 console pages into the setup prompts | `iris agent-memory service env <store-id> --write editor.env` and `iris agent-playbooks service env <pb-id> --write editor.env` write the exact variable names, leaving the key empty. Then `setup_demo.py --yes` reads the file. | **Today** on a `kind = "cloud"` profile once the URL is set (tried offline on 1.1.0). `--yes`: **Today** |

Step 4 as a file, once Cloud sign-in ships. One command replaces the type
dialog and its nine **Add field** rows:

```yaml
name: real-reaction-demo
memory_types:
  - name: organizational_state
    description: Organizational-state observations and current-state projections for management review.
    # No prompt, so the CLI sends no extraction strategy. Check in the
    # console that the type reads "Not used for session extraction".
    fields:
      schema_version: str
      state_kind: str
      lifecycle_status: str
      project_id: str
      meeting_id: str
      accountable_entity_id: str
      conflict_status: str
      record_kind: str
      details: object
```

```sh
.venv/bin/iris agent-memory service create --from service.yaml --write-env ~/.config/redis-shared-memory/editor.env
```

## Checking and troubleshooting (instead of console tabs)

| Console click | `iris` command | Status |
| --- | --- | --- |
| Open the service to see whether the endpoint, store ID and key work | `iris agent-memory health` makes one authenticated read and names what failed | **Today** |
| Playbook **Configuration** page, to check it is reachable | `iris agent-playbooks health` | **Today** |
| **Registry** tab, to see whether the Skill is published and which version | `iris agent-playbooks skills versions prepare-management-review-agenda` | **Today** |
| The tour's "Look for: the Skill version shown in your Registry" | `iris --json agent-playbooks discover "prepare management review agenda" --pin-key prepare-management-review-agenda` returns the same discovery the plugin runs | **Today** |
| **Memory Inspector** tab, to check that the documents loaded | `iris agent-memory memories search "LCD decision" --owner andrew --shared real-reaction` and `iris agent-memory memories list --owner andrew` | **Today** |
| **Memory Inspector**, for a participant's private sessions | `iris agent-memory sessions list --owner andrew`, then `sessions show <id>` | **Today** |
| Click through the service, Identity and key to find which hop is broken | `iris agent-memory service check <store-id> --env-file editor.env` and `iris agent-playbooks service check <pb-id> --env-file editor.env` | When Cloud sign-in ships |
| **API Keys & Team**, to list, rotate or revoke the demo keys | `iris identity keys list`, `iris identity keys rotate …`, `iris identity keys revoke …` | When Cloud sign-in ships |

Load the settings for one command with `(set -a; . ~/.config/redis-shared-memory/editor.env; .venv/bin/iris agent-memory health)`.
Use `publisher.env` for commands that need the content key.

## Resetting the demo

| Today's path | `iris` command | Status |
| --- | --- | --- |
| `scripts/reset_demo_ram.py` (all memories and sessions) | `iris agent-memory memories forget --owner <participant> --yes` resets one participant | **Today** |
| Delete and re-upload the Skill in **Registry** | `iris agent-playbooks skills withdraw …` / `skills delete … --yes` | **Today** |
| Delete the Playbook's content but keep the Playbook | `iris agent-playbooks clear --scope entries --yes` | When Cloud sign-in ships |
| Delete the service | `iris agent-memory service delete <store-id>` / `iris agent-playbooks service delete <pb-id>` | When Cloud sign-in ships |

## SDK alternatives

The Python SDKs that the plugin already installs (`redis-agent-memory`,
`redis-agent-playbook`, `redis-iris`, all 1.1.0) cover everything the CLI does.
They also cover the two control-plane steps the CLI can't do: the memory type
**with its field descriptions**, and the extractor.

| README step / console click | SDK call | Status |
| --- | --- | --- |
| Step 3: create the store | `redis_agent_memory.controlplane.AgentMemoryService.create_store(database=..., name=..., custom_memory_types=[...])` | When Cloud sign-in ships |
| Step 4: the memory type, nine fields **with descriptions** | `AgentMemoryService.create_memory_type(store_id=..., name=..., description=..., fields=[...])`, or `update_memory_type` later. Also works on a store created in the console. | When Cloud sign-in ships |
| Step 5: the extractor | `AgentMemoryService.create_extractor(store_id=..., memory_type=..., name=..., prompt=..., source="DOCUMENTS", strategy="CUSTOM")` returns `extractor_id` | When Cloud sign-in ships |
| Step 7: create the Playbook | `redis_agent_playbook.controlplane.AgentPlaybooksService.create_playbook(...)` | When Cloud sign-in ships |
| Step 7: upload and publish the Skill | `redis_agent_playbook.Playbook.create_skill(...)` / `append_skill_version(...)`, then `publish_skill(...)`. The CLI's `skills push --publish all` wraps these with digest checks and retries, so prefer the CLI. | **Today** |
| Step 9: load the documents | `redis_agent_memory` document ingestion, already what `setup_demo.py` calls | **Today** |
| Checks: health, discovery, recall | `redis_iris.AgentPlaybooks.from_env().discover(...)`, `redis_iris.AgentMemory` recall, `Playbook.get_playbook_health(...)` | **Today** |

The control-plane SDK takes a `server_url` and a bearer token: the same
operator credential the CLI is waiting for on Cloud. It works today on
self-managed Iris. On Cloud, it works once control-plane sign-in ships.

Steps 4 and 5 in one script, reading the checked-in contract so nothing is
retyped. This was checked against a mock control plane on 1.1.0: it sends
the nine fields with their descriptions, then creates the extractor.

```python
import json
import os
from pathlib import Path

from redis_agent_memory.controlplane import AgentMemoryService

V1 = Path("context-models/management-agenda/v1")
TYPES = {"str": "STRING", "object": "OBJECT"}
spec = json.loads((V1 / "memory-types/organizational-state-compact.json").read_text())["customMemoryType"]
store_id = os.environ["IRIS_AGENT_MEMORY_STORE_ID"]

service = AgentMemoryService(
    server_url=os.environ["IRIS_AGENT_MEMORY_SERVICE_URL"],
    token=os.environ["IRIS_AGENT_MEMORY_SERVICE_TOKEN"],
)
service.create_memory_type(
    store_id=store_id,
    name=spec["name"],
    description=spec["description"],
    fields=[{"name": f["name"], "type": TYPES[f["type"]], "description": f["description"]} for f in spec["fields"]],
)
extractor = service.create_extractor(
    store_id=store_id,
    name="organizational-state-documents",
    memory_type=spec["name"],
    prompt=(V1 / "extraction-prompt-compact.txt").read_text(),
    source="DOCUMENTS",
    strategy="CUSTOM",
)
print(extractor.extractor_id)   # the ex_... value for editor.env
```

## Gaps for the Iris team

The SDK already covers all three of these (see above). Until the CLI covers
them too, Cloud sign-in alone still leaves console steps for anyone using only
the CLI:

1. **No extractor command.** Step 5 stays a console form even with sign-in.
   The control-plane route exists (`POST /iris/v1/agent-memory/stores/{storeId}/extractors`).
2. **The CLI can't set memory-type field descriptions.** `memory_types.fields`
   takes `name: type` only and derives each description from the field name.
   This demo's descriptions carry extraction guidance (for example
   `state_kind`'s allowed values), so the console is still needed to enter
   them.
3. **No memory-type command for an existing store.** Types can only be set
   at `service create`. Adding one to a store that already exists stays in
   the console.

Status as of redis-iris 1.1.0 (pinned here, the latest on 2026-11-16). I
checked that each command exists and ran the offline ones: `skills validate`,
the Cloud refusals and `service env`. I haven't run the key-based **Today**
commands against a live Redis Cloud service.
