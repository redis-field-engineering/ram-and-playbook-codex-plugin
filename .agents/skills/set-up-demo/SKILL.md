---
name: set-up-demo
description: Set up the Redis Shared Memory + Playbook demo for the user from an empty Redis Cloud account. Walks them through the console-only steps, then does the rest itself with the bundled iris CLI and scripts/setup_demo.py --yes, which publishes the agenda Skill, writes the settings, loads the demo documents and installs the ChatGPT plugin. Use for "set up the demo", "get this running", "install the plugin", or when README Part 2 or 3 fails.
---

# Set up the demo

You do the README setup for the user. Do everything that has a command
yourself. Hand the user only the steps that need the Redis Cloud console or a
ChatGPT dialog. Run every command from the repository root.

**Keys never pass through you.** Don't ask for API keys in chat, don't echo
them, and don't `cat` either settings file. The user pastes keys straight into
the files. Endpoints, store, extractor and playbook IDs, and the participant
name are not secret, so the user can paste those to you.

## 1. Check the machine (you)

```sh
python3 -c 'import sys; assert sys.version_info >= (3, 11), sys.version'
```

```sh
test -x .venv/bin/iris || { python3 -m venv .venv && .venv/bin/python -m pip install -q -r plugins/redis-shared-memory-editor/server/requirements.txt; }
```

```sh
.venv/bin/python scripts/validate_context_pipeline.py
```

Stop if Python is older than 3.11 (README Step 1) or a check fails.

## 2. Console steps (user)

Only the console can create services, memory types, extractors and keys today.
The `iris` CLI covers most of these control-plane steps once its Redis Cloud
sign-in ships. [docs/console-to-cli.md](../../../docs/console-to-cli.md)
tracks which. If that page marks a step **Today**, run the command instead of
asking for the click. Send the user the rest in one message, linking the
README sections:

1. **README Step 3**: create the Agent Memory service. Note the endpoint and
   store ID.
2. **README Step 4**: add the `organizational_state` memory type with its nine
   fields. Fields can't be edited after saving.
3. **README Step 5**: create the extractor. Run
   `pbcopy < context-models/management-agenda/v1/extraction-prompt-compact.txt`
   for them first, so the prompt is on their clipboard. Note the `ex_…` ID.
4. **README Step 6**: create the Agent Memory read/write key.
5. **README Step 7, items 2–4 only**: create the Playbook and note its agent
   endpoint and `pb_…` ID. **Skip** packaging, Upload Skill and Publish.
   You do those.
6. On the Playbook page, create **two** keys:
   - a **full-access** key (the default dialog). It is used once, to publish
     the Skill.
   - a **read-only** key (**README Step 8**). The plugin runs with this one.

Ask the user to reply with the six non-secret values and a participant name
(`andrew`, `craig`, `david` or `laura`).

## 3. Write the settings files (you, then user)

Write two files, both mode `0600`, under `~/.config/redis-shared-memory/`.
Leave the key lines empty. Back up an existing `editor.env` first.

`editor.env` holds the runtime settings. `setup_demo.py --yes` reads it and
rewrites it:

```text
# MemoryETL extractor used by scripts/setup_demo.py: ex_...
IRIS_AGENT_MEMORY_URL=https://...
IRIS_AGENT_MEMORY_STORE_ID=st_...
IRIS_AGENT_MEMORY_API_KEY=
IRIS_AGENT_PLAYBOOKS_URL=https://...
IRIS_AGENT_PLAYBOOKS_PLAYBOOK_ID=pb_...
IRIS_AGENT_PLAYBOOKS_API_KEY=
RAM_PARTICIPANT=andrew
```

The extractor line must be the comment exactly as shown; the plugin rejects
unknown settings. Use base URLs only, with nothing after the host.

`publisher.env` holds only the publish settings. Keep the full-access key out
of `editor.env`:

```text
IRIS_AGENT_PLAYBOOKS_URL=https://...
IRIS_AGENT_PLAYBOOKS_PLAYBOOK_ID=pb_...
IRIS_AGENT_PLAYBOOKS_CONTENT_API_KEY=
```

Create them with `umask 077`. Then ask the user to open each file and paste
the keys:
- In `editor.env`: the Agent Memory key and the **read-only** Playbook key.
- In `publisher.env`: the **full-access** Playbook key.

Check that the keys are there without printing them. Each count must be `1`:

```sh
grep -cE '^IRIS_AGENT_MEMORY_API_KEY=[^[:space:]]{16,}$' ~/.config/redis-shared-memory/editor.env
```

```sh
grep -cE '^IRIS_AGENT_PLAYBOOKS_API_KEY=[^[:space:]]{16,}$' ~/.config/redis-shared-memory/editor.env
```

```sh
grep -cE '^IRIS_AGENT_PLAYBOOKS_CONTENT_API_KEY=[^[:space:]]{16,}$' ~/.config/redis-shared-memory/publisher.env
```

## 4. Publish the agenda Skill (you)

This replaces the console's Upload Skill and Publish. First verify the digests
and stage the bundle. The CLI needs the folder name to match the Skill name,
and it reads the alias's intents and attributes from `.iris-playbook.json`:

```sh
python3 scripts/package_skill.py
```

```sh
rm -rf dist/skills && mkdir -p dist/skills && cp -R context-models/management-agenda/v1/playbook dist/skills/prepare-management-review-agenda && rm dist/skills/prepare-management-review-agenda/manifest.json
```

```sh
python3 -c 'import json; m = json.load(open("context-models/management-agenda/v1/playbook/manifest.json")); json.dump({"intents": m["intents"], "attributes": m["attributes"]}, open("dist/skills/prepare-management-review-agenda/.iris-playbook.json", "w"), indent=2)'
```

```sh
.venv/bin/iris agent-playbooks skills validate dist/skills
```

Then plan and publish with the full-access key. Each command loads
`publisher.env` for that command only:

```sh
(set -a; . ~/.config/redis-shared-memory/publisher.env; .venv/bin/iris agent-playbooks skills push dist/skills/prepare-management-review-agenda --dry-run)
```

```sh
(set -a; . ~/.config/redis-shared-memory/publisher.env; .venv/bin/iris agent-playbooks skills push dist/skills/prepare-management-review-agenda --publish all)
```

```sh
(set -a; . ~/.config/redis-shared-memory/publisher.env; .venv/bin/iris agent-playbooks skills versions prepare-management-review-agenda)
```

You can re-run `--publish all` safely. It publishes only when the latest
version isn't serving yet. When `versions` shows a published version, tell the
user they can delete the full-access key in the console and delete
`publisher.env`.

## 5. Run setup (you)

```sh
python3 scripts/setup_demo.py --yes
```

`--yes` reads every value from `editor.env` and stops instead of prompting if
one is missing or invalid. It checks both services, loads the three demo
documents (a few minutes), builds the current state, installs the plugin,
limits it to this folder, and trusts the folder. It is safe to re-run.

If it fails, match its `✗` line to README "If something goes wrong". Fix what
you can with a command. Send console fixes back to the user.

To find out which service a failure comes from, check each one yourself
rather than sending the user to the console:

```sh
(set -a; . ~/.config/redis-shared-memory/editor.env; .venv/bin/iris agent-memory health)
```

```sh
(set -a; . ~/.config/redis-shared-memory/editor.env; .venv/bin/iris --json agent-playbooks discover "prepare management review agenda" --pin-key prepare-management-review-agenda)
```

## 6. Hand back (user)

If ChatGPT was open during setup, the user must quit it (⌘ Q) and reopen it.
Then they follow **README Steps 10–11**: create the ChatGPT project on this
folder, send "Check the Redis Shared Memory connection status", and review and
allow the two hooks. Hook trust is the user's own security decision, so never
approve it for them. Then point them at the guided tour in README Part 4.
