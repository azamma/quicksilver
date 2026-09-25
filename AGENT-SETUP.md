# Set up Quicksilver (Jev) in Claude Code: agent runbook

Steps for an agent configuring Quicksilver on a user's machine. Every step ends on a
check. Move to the next step only after its check passes. Paths assume macOS/Linux; on
Windows use `%USERPROFILE%` for `~`.

`QS` below means `node ~/.claude/skills/quicksilver/scripts/qs.mjs`.

## 1. Install the skill

From a clone of this repo: `./install.sh` (Windows: `.\install.ps1`). Without a clone:
`npx github:UditAkhourii/quicksilver`. Both copy `skills/quicksilver` into
`~/.claude/skills/quicksilver`.

**Done when** `~/.claude/skills/quicksilver/scripts/qs.mjs` and `guard.mjs` both exist.

## 2. Pick a provider and store the key

Jev is served by two providers. The answers are the same.

| provider | key from | env var |
| --- | --- | --- |
| `typesafe` (default) | https://console.typesafe.ai | `JEV_API_KEY` or `TYPESAFE_API_KEY` |
| `openrouter` | https://openrouter.ai/settings/keys | `OPENROUTER_API_KEY` |

Ask the user which provider they want, unless a key already exists. Check without reading
values: `env | grep -c "TYPESAFE_API_KEY\|JEV_API_KEY\|OPENROUTER_API_KEY"`, and
`grep -c "^\(TYPESAFE\|OPENROUTER\)_API_KEY=" ~/.env`.

Keep the key out of the chat. Pick the first route that applies:

- The key is in `~/.env`: pipe it into setup, so the value never reaches your context:
  `grep "^TYPESAFE_API_KEY=" ~/.env | cut -d= -f2- | QS setup --provider typesafe`
  (swap in `OPENROUTER_API_KEY` and `--provider openrouter`).
- There is no key yet: ask the user to run `! node ~/.claude/skills/quicksilver/scripts/qs.mjs setup --provider <p>`
  in the prompt. It asks for the key with hidden input.
- The user pastes the key in chat anyway: run `QS setup <KEY> --provider <p>`.

`setup` checks the key against the provider, saves it to `~/.quicksilver/config.json`
(mode 600), and makes that provider the default. Precedence and the other env vars are
listed in `QS help` and `skills/quicksilver/SKILL.md`.

**Done when** `QS status` prints `ready · provider <p> · …` and exits 0. Exit 3 means the
key is missing or was rejected; repeat this step.

## 3. Enforce and show Jev with hooks

`guard.mjs` handles two hook events:

- **PreToolUse on `Read`**: denies a whole-file read of a text file over 600 lines or
  60 KB. It points Claude to `qs find` / `qs filter`, then to `Read` with `offset`/`limit`.
- **PostToolUse on `Bash`**: after a `qs` run, shows the Jev footer in the UI
  (`☿ quicksilver → Jev … tok ($…)`).

Merge both hooks into `~/.claude/settings.json`, keeping the hooks already there. Back the
file up first:

```bash
cp ~/.claude/settings.json ~/.claude/settings.json.bak-quicksilver 2>/dev/null
node -e '
const fs=require("fs"),f=process.env.HOME+"/.claude/settings.json";
const s=fs.existsSync(f)?JSON.parse(fs.readFileSync(f,"utf8")):{};
const cmd="node \""+process.env.HOME+"/.claude/skills/quicksilver/scripts/guard.mjs\"";
s.hooks??={};
const add=(ev,matcher,extra)=>{s.hooks[ev]??=[];
  if(!JSON.stringify(s.hooks[ev]).includes("guard.mjs"))
    s.hooks[ev].push({matcher,hooks:[{type:"command",command:cmd,timeout:5,...extra}]});};
add("PreToolUse","^Read$",{statusMessage:"☿ quicksilver guard…"});
add("PostToolUse","^Bash$",{});
fs.writeFileSync(f,JSON.stringify(s,null,2)+"\n");'
```

**Done when** both checks pass:

1. `jq '[.hooks.PreToolUse[],.hooks.PostToolUse[]]|map(select(tostring|test("guard.mjs")))|length' ~/.claude/settings.json`
   prints `2`.
2. A pipe test of the guard on a large file prints JSON with `"permissionDecision":"deny"`:
   ```bash
   node -e 'console.log(JSON.stringify({hook_event_name:"PreToolUse",tool_name:"Read",tool_input:{file_path:process.argv[1]}}))' \
     ~/.claude/skills/quicksilver/scripts/qs.mjs | node ~/.claude/skills/quicksilver/scripts/guard.mjs
   ```

Tuning: `QUICKSILVER_GUARD_LINES`, `QUICKSILVER_GUARD_BYTES`. Set `QUICKSILVER_GUARD=off`
to switch the guard off without removing the hooks.

## 4. Add the standing rule

Append this to `~/.claude/CLAUDE.md` unless a `# quicksilver` section is already there:

```markdown
# quicksilver
- Before reading many files, long logs or big lists just to decide what matters, use the `quicksilver` skill (`qs filter|classify|rank|find`) and read only the survivors. A PreToolUse hook (`~/.claude/skills/quicksilver/scripts/guard.mjs`) blocks whole-file Reads of large files; follow its message instead of working around it.
```

**Done when** `grep -c "^# quicksilver" ~/.claude/CLAUDE.md` prints `1`.

## 5. Hand off

Tell the user:

- Restart Claude Code, or open `/hooks` once, so the new hooks load.
- The two UI lines they will see: `☿ quicksilver: blocked whole Read of …` and
  `☿ quicksilver → Jev …`.
- Content sent to Jev leaves the machine (to `api.typesafe.ai` or `openrouter.ai`).
  `.env*`, keys and credential files are never sent.
- To undo: restore `~/.claude/settings.json.bak-quicksilver`, delete the `# quicksilver`
  section from `CLAUDE.md`, and run `QS setup --remove --provider <p>`.

**Done when** the user has the restart instruction and the undo path.
