# hasib-rns

Cursor plugin marketplace with three installable review skills:

| Plugin | What it does |
| --- | --- |
| **GDPR Check** | Data-protection review when a change touches personal data |
| **Backend Security Check** | Security review of APIs, auth, queries, uploads, and server code |
| **Frontend Security Check** | Security review of UI, XSS, tokens, storage, and route guards |

Each plugin is a standard Cursor plugin (`SKILL.md` + `.cursor-plugin/plugin.json`). The repo root has `.cursor-plugin/marketplace.json` so Cursor can import this GitHub repo and list those plugins.

## Install from Cursor

### Option A — add this repo as a Team Marketplace

Needs a Cursor Teams or Enterprise plan.

1. Open [cursor.com/dashboard](https://cursor.com/dashboard) → **Plugins**
2. Under **Team Marketplaces**, click **Add Marketplace**
3. Choose **Import from Repo**
4. Paste: `https://github.com/hasibul-selise/hasib-rns`
5. Review the three plugins, set install mode (Default Off / Default On / Required), then save

Teammates then open **Customize** in the Cursor sidebar and install the plugins from this marketplace.

### Option B — install from Agent chat

This only works if the GitHub repo is **public**, or Cursor already has GitHub access to this private repo.

In the chat box, pick the `/add-plugin` slash command (do not send it as a normal message):

```text
/add-plugin hasibul-selise/hasib-rns
```

Or paste: `https://github.com/hasibul-selise/hasib-rns`

### Option C — test locally (before or without publishing)

Copy or symlink each plugin folder into Cursor's local plugin directory, then reload the window (**Developer: Reload Window**).

PowerShell:

```powershell
$local = "$env:USERPROFILE\.cursor\plugins\local"
New-Item -ItemType Directory -Force -Path $local | Out-Null

$repo = (Get-Location).Path
foreach ($plugin in @("gdpr-check", "security-check-be", "security-check-fe")) {
  $link = Join-Path $local $plugin
  $target = Join-Path $repo "skills\$plugin"
  if (Test-Path $link) { Remove-Item $link }
  New-Item -ItemType SymbolicLink -Path $link -Target $target | Out-Null
}
```

On Teams/Enterprise, local plugin imports may be disabled by an admin (**Dashboard → Settings → Security & Identity → Marketplace and Plugins**).

After a reload, open **Customize** and confirm the skills appear. Invoke one in chat with `/gdpr-check`, `/security-check-be`, or `/security-check-fe`.

## After you push

Cursor indexes plugins from GitHub. After the first import:

- Turn on **Enable Auto Refresh** on the marketplace (requires the Cursor GitHub App on this repo), or
- Click **Refresh** on the marketplace after you push changes

Bump the `version` in that plugin's `.cursor-plugin/plugin.json` (and the matching entry in `.cursor-plugin/marketplace.json`) when you ship an update.

## Add another plugin later

1. Create `skills/my-skill/SKILL.md` (YAML `name` + `description` in the frontmatter)
2. Add `skills/my-skill/.cursor-plugin/plugin.json` with a unique kebab-case `name`
3. Add a matching entry to `.cursor-plugin/marketplace.json` with `"source": "skills/my-skill"`
4. Push, then refresh the marketplace in the Cursor dashboard
