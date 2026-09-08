# hasib-rns

One Cursor plugin that installs all review skills:

| Skill | What it does |
| --- | --- |
| **gdpr-check** | Data-protection review when a change touches personal data |
| **security-check-be** | Security review of APIs, auth, queries, uploads, and server code |
| **security-check-fe** | Security review of UI, XSS, tokens, storage, and route guards |

```
hasib-rns/
├── .cursor-plugin/plugin.json
└── skills/
    ├── gdpr-check/SKILL.md
    ├── security-check-be/SKILL.md
    └── security-check-fe/SKILL.md
```

## Install

### Local (this machine)

Copy the plugin into Cursor's local plugin folder, then reload (**Developer: Reload Window**). Cursor rejects symlinks that point outside that folder.

```powershell
$dst = "$env:USERPROFILE\.cursor\plugins\local\hasib-rns"
$src = (Get-Location).Path
if (Test-Path $dst) { Remove-Item $dst -Recurse -Force }
New-Item -ItemType Directory -Force -Path "$dst\.cursor-plugin", "$dst\skills" | Out-Null
Copy-Item "$src\.cursor-plugin\plugin.json" "$dst\.cursor-plugin\plugin.json"
Copy-Item "$src\skills\*" "$dst\skills" -Recurse -Force
```

On Teams/Enterprise, local plugin imports may be disabled by an admin.

### From GitHub

Works if the repo is **public**, or Cursor has GitHub access to this private repo.

- Agent chat slash command: `/add-plugin hasibul-selise/hasib-rns`
- Or **Customize** → install `hasib-rns`

After install, open **Customize** and confirm one plugin: **Hasib RNS**, with all three skills. Invoke them with `/gdpr-check`, `/security-check-be`, or `/security-check-fe`.

## Add another skill

1. Create `skills/my-skill/SKILL.md` with YAML `name` and `description`
2. Bump `version` in `.cursor-plugin/plugin.json`
3. Copy the updated plugin into `~\.cursor\plugins\local\hasib-rns` (or push and refresh)
