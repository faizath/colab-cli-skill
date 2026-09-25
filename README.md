# colab-cli skill

An [Agent Skill](https://agentskills.io/home) that teaches your coding agent to run
GPU/TPU jobs on Google Colab from the terminal with the `colab` CLI — provisioning
VMs, launching long training runs, syncing files, and recovering orphaned sessions.

The `colab` CLI ships its own docs (`colab skill`, `colab readme`). This skill covers
what those docs *don't* say: the failure modes that silently kill multi-hour runs.
Each one was learned by losing a run to it.

## What it covers

- **Keep-alive and VM reclaim** — why a session prune at the 1-hour token expiry kills
  the keep-alive daemon and gets your VM reclaimed ~40 minutes later, and how to drive
  keep-alive yourself so a long run survives it.
- **Detaching long jobs** — launching with `start_new_session=True` so the job outlives
  the kernel, the websocket, the token, and the session record.
- **Orphan recovery** — why `colab new` is the wrong way to recover an orphaned session
  (it provisions a second billable GPU) and what to use instead.
- **Misleading errors** — the cases where the message names the wrong problem, such as
  an `HFValidationError` about repo ids that actually means the weights never uploaded.
- **Verified transfers** — chunking uploads above ~24 MB and checking size and md5 from
  the VM side, because a large upload can be accepted and then silently dropped.
- **Cost discipline** — stopping idle VMs, and smoke-testing a config before spending
  hours of GPU time on it.

## Install

### Claude Code (`/plugin`)

Register this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add faizath/colab-cli-skill
/plugin install colab-cli@colab-cli-skill
```

From a local clone instead, point the same commands at the directory:

```bash
git clone https://github.com/faizath/colab-cli-skill
claude plugin marketplace add ./colab-cli-skill
claude plugin install colab-cli@colab-cli-skill
```

To try it for a single session without installing anything:

```bash
claude --plugin-dir ./colab-cli-skill
```

Installed as a plugin, the skill is namespaced under the plugin name and invokes as
`/colab-cli:colab-cli`.

### `npx skills`

```bash
npx skills add faizath/colab-cli-skill
```

Add `-g` to install into your user directory instead of the current project, or pass a
local path for development:

```bash
npx skills add ./colab-cli-skill
```

## Usage

Once installed, the skill activates on its own when a task involves Colab, remote GPU
runs, or the `colab` CLI. You can also invoke it explicitly:

```
/colab-cli:colab-cli          # installed as a Claude Code plugin
```

Or just mention it:

> "Use the colab-cli skill to launch this training run on an A100 and poll it until it
> finishes."

## Requirements

The skill drives the `colab` CLI, which you install separately. Pin the kernel-client
dependency — version 1.0 renamed a class that `colab-cli` ≤ 0.6.0 still expects:

```bash
uv tool install --force google-colab-cli --with "jupyter-kernel-client<1.0"
```

You will also need Google OAuth credentials for Colab configured on your machine. The
skill documents the auth paths it expects and which Python interpreter to run its
snippets with.

## Repository layout

`SKILL.md` sits at the repository root, which both installers understand: Claude Code
loads a plugin whose root holds `SKILL.md` (and no `skills/`) as a single skill, and
`npx skills` searches the root before `skills/`.

```
colab-cli-skill/
├── .claude-plugin/
│   ├── plugin.json         # plugin metadata
│   └── marketplace.json    # one-plugin marketplace, source "./"
├── SKILL.md                # the skill itself
├── LICENSE
└── README.md
```

Validate both manifests after any edit:

```bash
claude plugin validate --strict .
```

## License

[MIT](LICENSE)
