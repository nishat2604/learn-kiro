# Kiro CLI Setup on AVD (WSL) with Jira & Confluence MCP

Step-by-step guide to install Kiro CLI inside WSL on an organization-managed Azure Virtual Desktop (AVD), sign in through AWS IAM Identity Center, and connect Jira and Confluence through an MCP server.

> **Placeholders:** values in `<angle brackets>` are specific to your environment. Replace them with your own and **do not type the angle brackets**. Never commit real tokens, emails or internal IPs.

---

## Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Fix DNS in WSL](#2-fix-dns-in-wsl)
3. [Install Kiro CLI](#3-install-kiro-cli)
4. [Log in with IAM Identity Center](#4-log-in-with-iam-identity-center)
5. [Connect Jira and Confluence (MCP)](#5-connect-jira-and-confluence-mcp)
6. [Verify the connection](#6-verify-the-connection)
7. [Everyday Kiro commands](#7-everyday-kiro-commands)
8. [Troubleshooting](#8-troubleshooting)
9. [Security notes](#9-security-notes)

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| AVD with WSL (Ubuntu) | The commands below run inside WSL |
| Approved Kiro access | Your Kiro request must be approved and processed before login works |
| Organization account | Sign in with your regular work account, **not** a privileged or admin account, and **not** a personal Builder ID |
| Atlassian API token | Create at <https://id.atlassian.com> → Security → API tokens |
| `uv` / `uvx` | Used to run the MCP server. Check with `uvx --version` |

---

## 2. Fix DNS in WSL

### Symptom

```text
curl: (6) Could not resolve host: cli.kiro.dev
```

Also `cat /etc/resolv.conf` reports *No such file or directory*, and `getent hosts github.com` prints nothing. If **no** hostname resolves, DNS is the problem, not the proxy.

### Step 1: get your organization's DNS servers (PowerShell on Windows)

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object ServerAddresses
```

Note the addresses under `ServerAddresses`.

### Step 2: check whether a proxy is used (PowerShell on Windows)

```powershell
netsh winhttp show proxy
```

```powershell
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' | Select-Object ProxyEnable, ProxyServer, AutoConfigURL
```

If `ProxyEnable` is `0` and `ProxyServer` / `AutoConfigURL` are empty, there is **no proxy**. Do not set `http_proxy` / `https_proxy`.

### Step 3: create `resolv.conf` in WSL

```bash
printf 'nameserver <dns-server-1>\nnameserver <dns-server-2>\n' > /etc/resolv.conf
```

### Step 4: test name resolution

```bash
getent hosts github.com cli.kiro.dev
```

Both names should print an IP address.

### Step 5: keep the fix after WSL restarts

```bash
printf '[network]\ngenerateResolvConf = false\n' >> /etc/wsl.conf
```

Then in PowerShell:

```powershell
wsl --shutdown
```

Reopen WSL and confirm `/etc/resolv.conf` still contains your nameservers.

> If a name resolves in Step 4 but `cli.kiro.dev` does not, your DNS is blocking it by policy. Ask IT to allowlist `cli.kiro.dev`.

---

## 3. Install Kiro CLI

```bash
curl -fsSL https://cli.kiro.dev/install | bash
```

Verify:

```bash
kiro-cli --version
```

---

## 4. Log in with IAM Identity Center

Without logging in, commands fail with:

```text
error: You are not logged in, please log in with kiro-cli login
```

### Steps

```bash
kiro-cli login
```

1. **Select login method:** use the arrow keys to choose **Use with Your Organization**. Do **not** pick Builder ID, Google or GitHub.
2. **Start URL:** your Identity Center *start* URL, for example:
   ```text
   https://<your-org>.awsapps.com/start
   ```
   > Use the `/start` URL, **not** `/login`. A wrong URL ends in `error: service error`.
3. **Region:** the AWS Region where your Identity Center is set up (ask IT or the Kiro admin), for example `<region>`.
4. WSL cannot open a browser. Copy the printed URL into a browser **on the AVD**, sign in with your regular work account, enter the code if asked, and click **Allow**.
5. Return to the terminal and confirm:

```bash
kiro-cli whoami
```

### Common login problems

| Problem | Fix |
|---|---|
| Access denied / wrong account | You are signed in with a privileged account. Switch to your regular profile in the AWS access portal (top-right), then retry |
| Signed in with Builder ID | Sign out and log in again via **Your Organization** |
| No Kiro access | Kiro request not yet approved or processed |
| `service error` | Wrong start URL or Region, or a blocked network. See [Troubleshooting](#8-troubleshooting) |

---

## 5. Connect Jira and Confluence (MCP)

This uses the open-source [`mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) server, launched with `uvx`.

### 5.1 Check `uvx`

```bash
uvx --version
```

If missing:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Open a new terminal (or `source ~/.bashrc`).

### 5.2 Create the credentials file

```bash
nano /root/.atlassian-mcp.env
```

Contents (replace the placeholders):

```env
JIRA_URL=https://<your-site>.atlassian.net
JIRA_USERNAME=<your-email>
JIRA_API_TOKEN=<your-api-token>
CONFLUENCE_URL=https://<your-site>.atlassian.net/wiki
CONFLUENCE_USERNAME=<your-email>
CONFLUENCE_API_TOKEN=<your-api-token>
READ_ONLY_MODE=true
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`. Restrict permissions:

```bash
chmod 600 /root/.atlassian-mcp.env
```

Check the file without printing the token:

```bash
sed -E 's/(TOKEN=).*/\1***/' /root/.atlassian-mcp.env
```

> `READ_ONLY_MODE=true` blocks create/update/delete actions. Remove it only if you want Kiro to modify Jira and Confluence.

### 5.3 Create the MCP config file

Copy the whole block, including the `cat` line and the final `EOF`:

```bash
cat > /root/atlassian-mcp.json <<'EOF'
{
  "mcpServers": {
    "atlassian": {
      "command": "uvx",
      "args": ["mcp-atlassian", "--env-file", "/root/.atlassian-mcp.env"],
      "disabled": false
    }
  }
}
EOF
```

> Do **not** paste the JSON directly into the terminal. It must be written to a file first. Use an absolute path (`/root/...`), not `~`.

### 5.4 Import into Kiro

Use `global` for all projects, or `workspace` for the current repo only:

```bash
kiro-cli mcp import --file /root/atlassian-mcp.json global
```

---

## 6. Verify the connection

```bash
kiro-cli mcp list
```

```bash
kiro-cli mcp status --name atlassian
```

Start Kiro and check inside the chat:

```bash
kiro-cli
```

```text
/mcp
/tools
```

You should see Jira and Confluence tools. Then try:

```text
Search Jira for my open issues
Show details of issue <ISSUE-KEY>
Search Confluence for pages about <topic>
```

> `(empty)` under every agent in `kiro-cli mcp list` means no MCP server is configured yet. It is not an error.

### Custom agents

Workspace agents (for example in `.kiro/agents`) can control which servers they see:

```json
{
  "name": "myagent",
  "mcpServers": {},
  "includeMcpJson": true
}
```

- `includeMcpJson: true`: the agent also gets the global and workspace MCP servers.
- `includeMcpJson: false`: the agent only uses servers in its own `mcpServers`.

---

## 7. Everyday Kiro commands

| Command | Purpose |
|---|---|
| `kiro-cli` | Start an interactive chat |
| `kiro-cli chat --no-interactive "<prompt>"` | Run one prompt and exit |
| `kiro-cli login` / `kiro-cli whoami` | Sign in / show the current user |
| `kiro-cli mcp list` | List configured MCP servers |
| `kiro-cli mcp status --name <server>` | Check one MCP server |
| `kiro-cli mcp add --help` | Show the add-server options |
| `kiro-cli mcp import --file <file> <global\|workspace>` | Import servers from a JSON file |

Inside a chat session (slash commands only work in interactive mode):

| Command | Purpose |
|---|---|
| `/help` | List commands |
| `/mcp` | MCP server status |
| `/tools` | Available tools, including MCP tools |
| `/context` | Context files and usage |
| `/model` | Choose a model |
| `/quit` | Exit |

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Could not resolve host` | No DNS in WSL | [Fix DNS](#2-fix-dns-in-wsl) |
| `not logged in` | Not signed in | `kiro-cli login` |
| `service error` at login | Wrong start URL or Region, or blocked network | Use `/start` URL; confirm Region; run the check below |
| `mcp list` is empty | No MCP server configured | [Section 5](#5-connect-jira-and-confluence-mcp) |
| Server listed but agent shows empty | Agent has `includeMcpJson: false` | Set it to `true` |
| Status shows a download error | PyPI blocked | Ask IT to allowlist PyPI |
| Tool calls return 401 | Wrong/expired token or username | Regenerate the Atlassian API token |
| Tool calls time out | Atlassian blocked | Test connectivity (below) |
| `syntax error near unexpected token` | JSON pasted into the shell | Write it to a file with the `cat <<'EOF'` command |

Check that AWS sign-in endpoints are reachable (any HTTP status means reachable):

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://oidc.<region>.amazonaws.com/
```

Check Atlassian reachability:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://<your-site>.atlassian.net
```

If these fail, ask IT to allowlist the domains: `cli.kiro.dev`, your `<your-org>.awsapps.com` portal, `oidc.<region>.amazonaws.com`, `<your-site>.atlassian.net`, and the PyPI hosts.

---

## 9. Security notes

- **Never commit** `.atlassian-mcp.env`, API tokens, or internal DNS server addresses. Add the env file to `.gitignore` if it ever lives in a repository.
- Keep the credentials file at `chmod 600`.
- Prefer `READ_ONLY_MODE=true` unless writes are explicitly needed.
- Rotate the Atlassian API token if it is ever pasted into a chat, ticket or log.
- Check your organization's policy before running `curl ... | bash` installers on managed machines.

---

## References

- Kiro CLI docs: <https://kiro.dev/docs/cli/>
- Kiro CLI commands: <https://kiro.dev/docs/cli/reference/cli-commands>
- Kiro slash commands: <https://kiro.dev/docs/cli/reference/slash-commands>
- Kiro MCP docs: <https://kiro.dev/docs/mcp/>
- mcp-atlassian: <https://github.com/sooperset/mcp-atlassian>
