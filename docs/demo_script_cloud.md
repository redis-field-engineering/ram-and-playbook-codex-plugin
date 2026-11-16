# Real Reaction demo on Redis Cloud: the fast path with the `iris` CLI

This script covers the same setup as README Parts 2 and 3, but most clicks
are replaced with `iris` commands. Only five things stay in the console:
creating the store, the memory type, the extractor, the Playbook, and two keys.
The CLI does everything else: packaging, upload, publish, checks, loading the
documents, installing the plugin, and inspecting the result.

**Status (2026-11-16):** every command was checked on redis-iris 1.1.0. The
offline steps ran: install, pipeline check, Skill validate, the Cloud refusals
and `service env`. The steps that need keys (3 to 6) have not yet run against
a live Redis Cloud service, and the script has not been timed. The target is
about 15 minutes, against the README's 45.

| Step | README (console) | This script |
|---|---|---|
| Install | Python check, clone | step 0, plus the `iris` CLI from the repo's own `.venv` |
| Store, memory type, extractor, Playbook | console | console, step 1 (control plane: no Cloud CLI sign-in yet) |
| Keys | three key dialogs, one needing "Customize in the full page" | **two** keys in Access & Identity, step 1e |
| Package, upload, intents, filter JSON, publish | Step 7, about a dozen clicks | step 4: `iris agent-playbooks skills push --publish all` |
| Type seven values into setup prompts | Step 9 | pasted once into `editor.env`; `setup_demo.py --yes` reads it |
| Check it worked | open the service, Registry and Memory Inspector tabs | steps 3 and 6: `iris ... health`, `skills versions`, `discover`, `memories search` |

**One terminal, T1**, in `~/ram-and-playbook-codex-plugin`, plus the console in
your browser. Each block is one command. Run it, read the output, then run the
next.

[console-to-cli.md](console-to-cli.md) has the full list of console clicks and
their commands.

## 0. Start from nothing

**T1.** Clone the project:

```sh
cd ~ && git clone https://github.com/redis-field-engineering/ram-and-playbook-codex-plugin.git
```

```sh
cd ~/ram-and-playbook-codex-plugin
```

Clear any `IRIS_*` variable this shell may already have, because a leftover
value silently overrides the files below:

```sh
unset -m 'IRIS_*'
```

```sh
env | grep -c '^IRIS_'        # 0
```

Create the project's private Python environment. It includes the `iris` CLI,
pinned in the plugin's requirements:

```sh
python3 -m venv .venv
```

```sh
.venv/bin/python -m pip install -q -r plugins/redis-shared-memory-editor/server/requirements.txt
```

```sh
export PATH="$PWD/.venv/bin:$PATH"
```

```sh
iris --version                # iris, version 1.1.0
```

Point the CLI at this repo's `iris.toml`. It holds a single line,
`kind = "cloud"`:

```sh
export IRIS_CONFIG="$PWD/iris.toml"
```

Prove the checked-in context model before touching any service:

```sh
python scripts/validate_context_pipeline.py
```

## 1. Console: five things only the console can create

Create the two settings files first. You paste each console value into them
as you go. Both files are private (`0600`):

```sh
mkdir -p -m 700 ~/.config/redis-shared-memory
```

```sh
(umask 077; touch ~/.config/redis-shared-memory/editor.env ~/.config/redis-shared-memory/publisher.env)
```

```sh
open -e ~/.config/redis-shared-memory/editor.env
```

Paste this into `editor.env`. You fill in the empty values during 1a to 1e:

```text
# MemoryETL extractor used by scripts/setup_demo.py: ex_
IRIS_AGENT_MEMORY_URL=
IRIS_AGENT_MEMORY_STORE_ID=
IRIS_AGENT_MEMORY_API_KEY=
IRIS_AGENT_PLAYBOOKS_URL=
IRIS_AGENT_PLAYBOOKS_PLAYBOOK_ID=
IRIS_AGENT_PLAYBOOKS_API_KEY=
RAM_PARTICIPANT=andrew
```

Pick `andrew`, `craig`, `david` or `laura`; each has demo work assigned. Keep
the extractor line as a comment: the plugin rejects settings it doesn't use.

```sh
open -e ~/.config/redis-shared-memory/publisher.env
```

Paste this into `publisher.env`. Only the Skill publish in step 4 uses it:

```text
IRIS_AGENT_PLAYBOOKS_URL=
IRIS_AGENT_PLAYBOOKS_PLAYBOOK_ID=
IRIS_AGENT_PLAYBOOKS_CONTENT_API_KEY=
```

To see what the console must do, ask the CLI:

```sh
iris agent-memory service create --name real-reaction-demo
```

It refuses, and prints the console steps. On Cloud the console owns stores,
Playbooks and keys until the CLI's Cloud sign-in ships, and the CLI says so
instead of failing halfway.

Open https://cloud.redis.io, then **Iris** in the left-hand menu.

### 1a. Agent Memory store

**Agent Memory → Create Agent Memory service.** Name:

```text
real-reaction-demo
```

Keep **Free Redis database** and select **Create**. In the **Connect** dialog,
copy the **Endpoint** (base URL only, nothing after `.redis.io`) into
`IRIS_AGENT_MEMORY_URL` and the **Store ID** into
`IRIS_AGENT_MEMORY_STORE_ID`. Then select **Skip for now**.

### 1b. The memory type

**Configuration → Edit → Add type.** Enter the name `organizational_state` and
switch **Use for session extraction** off. Then add the nine fields from README
Step 4's table and **Save**. Fields can't be edited after saving.

This is the slowest step. The CLI can create the type
(`service create --from service.yaml`), but only when it creates the store,
only once Cloud sign-in ships, and without the field descriptions this demo
relies on. See the gaps in [console-to-cli.md](console-to-cli.md).

### 1c. The extractor

**T1.** Put the extraction prompt on the clipboard:

```sh
pbcopy < context-models/management-agenda/v1/extraction-prompt-compact.txt
```

**MemoryETL → New extractor.** Name:

```text
organizational-state-documents
```

Keep memory type `organizational_state` and leave **Destination namespace**
empty. Switch on **Customize extraction prompt for documents**, paste with
⌘ V, and select **Save**. Copy the `ex_…` value from the REST example onto the
extractor comment line in `editor.env`.

### 1d. The Playbook

**Agent Playbooks → New Playbook.** Name:

```text
real-reaction-agenda
```

Keep the defaults and select **Create Playbook**. Copy the **Endpoint** into
`IRIS_AGENT_PLAYBOOKS_URL` and the **Playbook ID** into
`IRIS_AGENT_PLAYBOOKS_PLAYBOOK_ID`, **in both files**. Select **Skip for now**.

**Skip everything else on this page:** no ZIP, no Upload Skill, no Publish.
Step 4 does those.

### 1e. Two keys in Access & Identity

One key can carry a grant for each resource, so the plugin needs only one.
That also replaces the "Customize in the full page" workaround.

1. **Iris → Access & Identity → New API key.** Name:
   ```text
   real-reaction-runtime
   ```
   Under **Access**, tick the store `real-reaction-demo` with **read and
   write**, and the Playbook `real-reaction-agenda` with **read**.
   **Create key**, then paste it into **both** `IRIS_AGENT_MEMORY_API_KEY` and
   `IRIS_AGENT_PLAYBOOKS_API_KEY` in `editor.env`.
2. **New API key** again. Name:
   ```text
   real-reaction-publisher
   ```
   Tick the Playbook `real-reaction-agenda` with **full**. **Create key**, then
   paste it into `IRIS_AGENT_PLAYBOOKS_CONTENT_API_KEY` in `publisher.env`.

Each key is shown once. Save both files.

## 2. Load the runtime settings

**T1.** Load `editor.env` into this shell for the CLI:

```sh
set -a; source ~/.config/redis-shared-memory/editor.env; set +a
```

```sh
env | grep -c '^IRIS_AGENT_'  # 6
```

## 3. Check the keys

These commands replace opening each service page in the console.

```sh
iris agent-memory health
```

It exits 1 and names the problem if the URL, store ID or key is wrong.

```sh
iris agent-playbooks health
```

```sh
iris agent-playbooks skills list        # empty: the key reads the Playbook
```

## 4. Publish the agenda Skill

This replaces README Step 7's ZIP, Upload Skill, Optional settings and
Publish. First verify every file against the manifest digests:

```sh
python scripts/package_skill.py
```

Stage the bundle. The CLI needs the folder to be named after the Skill:

```sh
rm -rf dist/skills && mkdir -p dist/skills
```

```sh
cp -R context-models/management-agenda/v1/playbook dist/skills/prepare-management-review-agenda
```

```sh
rm dist/skills/prepare-management-review-agenda/manifest.json
```

The alias, intents and filter attributes go in `.iris-playbook.json`, not in
form fields:

```sh
python -c 'import json; m = json.load(open("context-models/management-agenda/v1/playbook/manifest.json")); json.dump({"intents": m["intents"], "attributes": m["attributes"]}, open("dist/skills/prepare-management-review-agenda/.iris-playbook.json", "w"), indent=2)'
```

```sh
iris agent-playbooks skills validate dist/skills     # VALID ... resources=6
```

Plan, then publish with the publisher key. `publisher.env` is loaded for that
one command only:

```sh
(set -a; source ~/.config/redis-shared-memory/publisher.env; iris agent-playbooks skills push dist/skills/prepare-management-review-agenda --dry-run)
```

```sh
(set -a; source ~/.config/redis-shared-memory/publisher.env; iris agent-playbooks skills push dist/skills/prepare-management-review-agenda --publish all)
```

Back on the runtime key, check what the plugin will see:

```sh
iris agent-playbooks skills versions prepare-management-review-agenda
```

```sh
iris --json agent-playbooks discover "prepare management review agenda" --pin-key prepare-management-review-agenda
```

The first result is `prepare-management-review-agenda`, at the published
version. You can delete the `real-reaction-publisher` key in the console now.
Publish again only when the Skill changes.

## 5. Load the documents and install the plugin

Quit ChatGPT (⌘ Q). Setup installs the plugin into it.

```sh
python scripts/setup_demo.py --yes
```

`--yes` reads `editor.env` and stops, rather than prompting, if a value is
missing. It checks both services, extracts the three demo documents (a few
minutes), builds the current state, installs the plugin for this folder only,
and trusts the folder. It is safe to re-run.

## 6. See what loaded

This replaces the console's Memory Inspector:

```sh
iris agent-memory memories search "LCD decision" --owner "$RAM_PARTICIPANT" --shared real-reaction --limit 5
```

The LCD and talk-back decisions from the AMI documents, with their sources.

```sh
iris agent-memory memories search "open action items" --owner "$RAM_PARTICIPANT" --shared real-reaction --limit 5
```

Your participant's open action, which the guided tour completes.

## 7. ChatGPT

Open ChatGPT and follow **README Steps 10 and 11**: create the project on this
folder, send "Check the Redis Shared Memory connection status", then review
and allow the two hooks. Hook trust is each user's own decision, so no script
does it. Then take the guided tour in **README Part 4**.

After the tour's publish step (prompt 4), check the update from the shell:

```sh
iris agent-memory memories search "trend-watching research completed" --owner "$RAM_PARTICIPANT" --shared real-reaction --limit 3
```

## Clean up

Reset one participant's memories and sessions:

```sh
iris agent-memory memories forget --owner "$RAM_PARTICIPANT" --yes
```

Reset the whole demo store, keeping the memory type and extractor (see
DEMO_ENVIRONMENT_SETUP.md):

```sh
python scripts/reset_demo_ram.py
```

Remove the plugin:

```sh
python scripts/setup_demo.py --remove
```

```sh
rm ~/.config/redis-shared-memory/publisher.env
```

Delete the keys, the store and the Playbook in the console. On Cloud,
`service delete` refuses, like `service create`.

## Known

- **The memory type is still nine fields by hand.** The CLI can't set field
  descriptions, and can only create types along with the store.
- **No extractor command.** Step 1c stays in the console, even once Cloud
  sign-in ships.
- **Not yet run live end to end.** Steps 3 to 6 need a real Cloud service. If
  a command behaves differently, record it in console-to-cli.md.
