<div align="center">

<img src="icon.png" width="64" height="64" alt="SqlGate">

# SqlGate

**A gate between your tools and your SQL Server databases — only what you allowed gets through.**
The host sends a query over HTTPS, the gateway parses it, checks it against the rules you set for each environment and either runs it, asks you first, or refuses.

[![Latest version](https://img.shields.io/github/v/release/vladkharitonov87/sql-gate-releases?label=version&color=0078D4)](https://github.com/vladkharitonov87/sql-gate-releases/releases/latest)
[![Release date](https://img.shields.io/github/release-date/vladkharitonov87/sql-gate-releases?label=released)](https://github.com/vladkharitonov87/sql-gate-releases/releases/latest)
[![Downloads](https://img.shields.io/github/downloads/vladkharitonov87/sql-gate-releases/total?label=downloads)](https://github.com/vladkharitonov87/sql-gate-releases/releases)
![Windows 10 | 11](https://img.shields.io/badge/Windows-10%20%7C%2011-0078D4?logo=windows)

<br>

[![Download for Windows](https://img.shields.io/badge/Download_for_Windows-0078D4?style=for-the-badge&logo=windows&logoColor=white)](https://github.com/vladkharitonov87/sql-gate-releases/releases/latest)

<sub>The <code>SqlGate-Setup-&lt;version&gt;.exe</code> file in the <b>Assets</b> section ·
<a href="https://github.com/vladkharitonov87/sql-gate-releases/releases">Release history</a></sub>

</div>

---

## What it does

- 🔍 **Understands the query, not just its words.** Every request is parsed with the SQL Server parser (ScriptDom). Exactly one statement is accepted; anything that does not parse or contains more than one statement is answered with `400`.
- 🧭 **Rules per environment.** For each environment (`dev`, `staging`, `prod`) every kind of statement is in one of three lists: **Allowed** (runs at once), **Ask me first** (waits for your *Allow* in the window or the tray) or **Blocked**. A new target reads only: `SELECT` is allowed, `INSERT` / `UPDATE` / `DELETE` / `MERGE` ask first, the rest is blocked.
- 🧪 **Try before you save.** The **Try a query** button shows the verdict for any SQL before the rules are saved.
- 🔒 **Secrets stay on the gateway.** Database passwords and the gateway token are entered once in the settings window, encrypted with Windows DPAPI under the service account and never shown again.
- 🛡️ **Encrypted and pinned.** The channel is HTTPS with a token; the client checks the gateway certificate by its SHA-256 thumbprint. The connection to the database is always encrypted.
- 🧱 **Isolated by design.** The service runs under its own virtual account `NT SERVICE\SqlGate`, listens on one address and port only, and the installer opens the firewall only for the host you name. There is no admin page inside the service.
- 🔔 **Asks you in time.** A pending request pops up over the tray with the SQL and **Allow** / **Deny** buttons; a request nobody answered expires as a refusal.
- 📜 **Keeps a history.** Every request and its verdict — allowed, awaiting, denied, rejected, expired — is in the **History** page and in the log.
- 🟢 **Lives in the tray.** The icon shows the state of the service and the number of pending requests; the menu starts, stops and restarts the service and opens the pages of the window.
- 🎨 **Light or dark theme** in the settings window.
- 🔄 **Updates itself:** the window tells you about a new version — one click, the installer is downloaded, its checksum is verified and it is installed.

## How it works

```
 host                        gateway (VM or separate PC)               database
 sqlgate query  ──HTTPS──▶   parse → rules → run / ask / refuse   ──▶  SQL Server / Azure SQL
 {env, sql}                  secrets in DPAPI · history · log
```

SqlGate is one application with two modes: `serve` (the Windows service) and `query` (the client you call from the host).
The settings window is a separate WPF application that edits the configuration and the secrets and talks to the service over a local control channel.

## Installation

1. Click **Download for Windows** above and download `SqlGate-Setup-<version>.exe` and `SqlGate-Setup-<version>.exe.sha256` from the **Assets** section.
2. Check the file (see the tip below) and run the installer **as administrator**.
3. Choose the components:
   - **Gateway service** — the Windows service `SqlGate`. Put it on the machine that reaches the databases.
   - **Settings window** — the WPF window and the tray icon.
   - **Query client** — `sqlgate.exe`, the command-line client. Put it on the **host** that sends the queries, not on the gateway machine.
4. On the service page set the address and port the gateway listens on and the IP of the host the firewall is opened for.
5. If Windows shows "Windows protected your PC", click **More info → Run anyway**. The installer is not digitally signed yet, so SmartScreen does not recognize it.

> [!TIP]
> Check that the file was downloaded intact by its SHA-256 checksum — do not run the installer if the two values differ:
> ```powershell
> (Get-FileHash .\SqlGate-Setup-<version>.exe -Algorithm SHA256).Hash.ToLower()
> Get-Content .\SqlGate-Setup-<version>.exe.sha256
> ```

Silent installation:

```powershell
.\SqlGate-Setup-<version>.exe /VERYSILENT /COMPONENTS=service,window /ListenAddress=<gateway ip> /Port=8443 /HostIp=<host ip>
```

## First setup

1. Open the settings window **as administrator** (the data folder `%ProgramData%\SqlGate` is closed to ordinary users; without the rights the window offers to restart itself elevated).
2. **Targets** — add a target for each environment: server, database, SQL login, the name of the secret with its password, timeout and row limit.
3. **Rules** — check the three lists of each environment and press **Save**; the service applies the rules at once.
4. **Secrets** — enter the gateway token (`gateway-token`) and the passwords of the targets. A value is typed twice and cannot be read back.
5. **Service** — press **Restart service**, then copy the SHA-256 certificate thumbprint.
6. On the host create `%APPDATA%\SqlGate\client.json`:

   ```json
   { "url": "https://<gateway ip>:8443", "token": "<gateway token>", "certificateThumbprint": "<thumbprint>" }
   ```

7. Check from the host:

   ```powershell
   sqlgate query --env dev "SELECT 1"
   ```

Exit codes of the client: `0` success · `1` arguments or config · `2` the gateway answered with an error (the body is in stdout) · `3` the gateway is unreachable · `4` the certificate thumbprint does not match.

## Requirements

| | |
|---|---|
| System | Windows 10 or 11, 64-bit |
| Rights | Administrator to install and to edit the configuration |
| Database | SQL Server or Azure SQL with a SQL login (the gateway has no Entra sign-in mode) |
| Network | The gateway machine reaches the databases; the host reaches the gateway on its address and port |
| Updates | Outgoing access to `api.github.com` and `github.com` (only for the update check and download) |

## Updating

The settings window checks for new versions by itself — at start and every 6 hours, and on **About → Check for updates**. When a new version is out, **Update to X.Y.Z** downloads the installer, verifies its checksum against the `.sha256` file of the release, runs it silently and reopens the window. Windows asks for administrator rights (UAC) for the installer itself.
You can also install the new version over the old one by hand. The service is stopped and started again; the configuration, the secrets and the certificate are kept.

What changed in each version is in the [release history](https://github.com/vladkharitonov87/sql-gate-releases/releases).

## Privacy

- The gateway never connects anywhere but the databases you configured. The only internet access of the app is the update check and the download of the update from GitHub, made by the settings window, not by the service.
- No accounts, no telemetry.
- Passwords are stored encrypted (DPAPI) on the gateway machine only and never leave it.

## Uninstalling

**Windows Settings → Apps → Installed apps → SqlGate → Uninstall.**
The installer stops and removes the service and the firewall rule and takes the client out of PATH. It asks whether to delete the data folder `%ProgramData%\SqlGate` (configuration, secrets, certificate, log); the answer is *no* by default.

## Questions and bugs

Found a bug or have an idea? [Create an issue](https://github.com/vladkharitonov87/sql-gate-releases/issues).
Please attach the version (**About** page) and describe what you did and what the gateway answered.

---

<sub>SqlGate is an independent project. Microsoft, SQL Server and Azure are trademarks of Microsoft Corporation.</sub>
