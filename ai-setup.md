# AI agent setup guide: install REA from this checkout

Instructions for an AI coding agent (Claude Code, Codex, and others) asked to
"install REA" on a fresh machine from a clone of this repository. Follow the
steps in order. The general product guide is
[docs/getting-started/installation.md](docs/getting-started/installation.md#run-rea-from-a-checkout).
This file records the steps for running from a checkout and the Windows
problems seen in practice.

## Rules

- Before installing system software (Node.js, VC++ runtime, Git), tell the
  user what will be installed and get approval. REA's own installers never
  install these.
- Read pinned versions from the repository, not from this file:
  - Node.js: `.nvmrc`, which must also satisfy `package.json#engines.node`.
    `.npmrc` sets `engine-strict=true`, so a non-matching Node fails `npm ci`.
  - npm: `package.json#packageManager`.
  - REA version: `package.json#version`. The Windows native addon must match it
    exactly.
- Always run `rea setup ... --dry-run` first, show the planned actions, and
  apply only after approval.

## 1. Prerequisites

Check these first: `node --version`, `npm --version`, `git --version`.

### Windows

| Need                                       | Install                                                                                                                           | Notes                                                                                                                                                                                                                                                             |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Node.js (exact `.nvmrc` version)           | `winget install --id OpenJS.NodeJS.LTS --version <nvmrc> --exact --silent --accept-package-agreements --accept-source-agreements` | Older patch versions may exist only in `OpenJS.NodeJS.LTS`, not in `OpenJS.NodeJS`. List versions with `winget show --id OpenJS.NodeJS.LTS --versions`. Installs to `C:\Program Files\nodejs`.                                                                    |
| npm (pinned)                               | `npm install -g npm@<packageManager version>`                                                                                     |                                                                                                                                                                                                                                                                   |
| Git on PATH                                | Usually already installed with Git for Windows                                                                                    | PowerShell needs `C:\Program Files\Git\cmd` on PATH (`npm ci` runs `git apply`). Add it to the **user** PATH and back up the old value first. Do not add `Git\ucrt64\bin` or `Git\usr\bin`, because their Unix tools would take precedence over Windows commands. |
| Microsoft Visual C++ 2015–2022 x64 runtime | `winget install --id Microsoft.VCRedist.2015+.x64 --exact --silent --accept-package-agreements --accept-source-agreements`        | Required by `turbo.exe`. Check for `C:\Windows\System32\vcruntime140.dll`.                                                                                                                                                                                        |

The current shell keeps its old PATH after a winget install. Prepend the folders
for this session:
`$env:Path = "C:\Program Files\nodejs;C:\Program Files\Git\cmd;$env:Path"`.

### macOS / Linux

Install the `.nvmrc` Node version with your Node manager, then the pinned npm.
Git is usually present. The Windows native addon step does not apply.

## 2. Install dependencies

```sh
npm ci
```

## 3. Restore the Windows native addon (Windows only)

Git does not track `native/windows/build/`, and the build fails without it.
Copy the two files from the published npm tarball whose version **exactly**
matches `package.json#version`. Never use `latest`. Unpack the tarball in a
scratch or temp directory, not in the repository:

```sh
# in a scratch directory; <ver> = package.json version
npm pack rea-agents@<ver>
tar -xzf rea-agents-<ver>.tgz
mkdir -p <repo>/native/windows/build
cp package/native/windows/build/manifest.json package/native/windows/build/rea-windows-x64.node <repo>/native/windows/build/
```

Check that the SHA-256 of `rea-windows-x64.node` matches `artifactSha256` in
`manifest.json`. An agent permission system may block this download because it
brings in a prebuilt binary. If so, explain why the file is needed and ask the
user for approval. Details: [native/windows/README.md](native/windows/README.md).

## 4. Build

```sh
npm run build:cached
```

The first build takes about 3 minutes.

## 5. Register with the agent

Run these from the repository root. Client IDs are listed in
[installation.md#supported-agents](docs/getting-started/installation.md#supported-agents);
Claude Code is `claude_code`.

```sh
node scripts/rea.mjs setup --client claude_code --dry-run --format json   # show the plan to the user
node scripts/rea.mjs setup --client claude_code --yes --format json       # after approval
```

For Claude Code, setup does two things:

- adds an MCP server entry to `~/.claude.json`, with a backup at
  `~/.claude.json.rea.backup`. The entry runs the absolute `node.exe` path with
  this checkout's `scripts/rea.mjs mcp`.
- installs the skill to `~/.claude/skills/reverse-engineer-anything`.

Success looks like `"status": "ready"` and registration `state: aligned`. Check
later with `node scripts/rea.mjs doctor --client claude_code`. Then tell the user
to **restart the agent** (in VS Code, reload the window).

Analysis engines (Ghidra, IDA, Hopper, Kuna) are optional and supplied by the
user. Set them up only if asked.

## Troubleshooting

| Symptom                                                                                                                                                                      | Cause                                                                       | Fix                                                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `npm ci` fails in `scripts/prepare.mjs` with `spawn git ENOENT`                                                                                                              | Git is not on this shell's PATH                                             | Add `C:\Program Files\Git\cmd` to PATH                                           |
| `npm ci` fails with an `EBADENGINE` error                                                                                                                                    | Node version does not satisfy `engines.node`                                | Install the exact `.nvmrc` version                                               |
| `build:cached` fails: `Missing packaged Windows native file native/windows/build/manifest.json`                                                                              | Native addon not restored                                                   | Step 3                                                                           |
| `build:cached` exits with code 53 and **no output**; running `node_modules\@turbo\windows-64\bin\turbo.exe --version` gives exit `-1073741515` (`0xC0000135`, DLL not found) | VC++ runtime missing                                                        | Install `Microsoft.VCRedist.2015+.x64`                                           |
| winget: `No version found matching`                                                                                                                                          | That version is published only under the other package ID                   | Try `OpenJS.NodeJS.LTS` and `OpenJS.NodeJS`                                      |
| Committing from VS Code (or another Git app) fails: `.husky/pre-commit: npx: command not found`                                                                              | The app was started before Node.js was installed and still has the old PATH | Fully quit and reopen the app. Tell the user to do this after installing Node.js |

## After `git pull`

Run `npm ci` and `npm run build:cached` again. If `package.json#version`
changed, repeat step 3 with the new version. If the skill changed, rerun the
setup dry run and apply, then restart the agent.
