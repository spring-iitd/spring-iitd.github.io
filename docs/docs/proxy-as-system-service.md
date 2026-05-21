# IITD Proxy Setup Guide (Ubuntu Server)

## Overview

IIT Delhi uses a **web-based captive portal proxy** (`proxy61.iitd.ac.in`) that requires an active authenticated session to route traffic. A login script is provided by the institute that handles session authentication and keeps it alive with periodic refreshes.

This guide covers:
- Running the login script as a persistent background service via `systemd`
- Configuring system-wide proxy environment variables
- Configuring `apt` to use the proxy

---

## Prerequisites

- Ubuntu server (tested on 22.04 LTS)
- The `proxy.sh` login script provided by the IITD CC
- Your IITD credentials

---

## Step 1: Install the Login Script

Copy the provided `proxy.sh` to a system-wide location and make it executable:

```bash
sudo cp proxy.sh /usr/local/bin/iitd-proxy.sh
sudo chmod +x /usr/local/bin/iitd-proxy.sh
```

Verify the shebang line is clean (no Windows line endings):

```bash
cat -A /usr/local/bin/iitd-proxy.sh | head -1
# Expected output: #!/bin/bash$
# If you see #!/bin/bash^M$ run: sudo sed -i 's/\r//' /usr/local/bin/iitd-proxy.sh
```

---

## Step 2: Create a systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/iitd-proxy.service
```

Paste the following:

```ini
[Unit]
Description=IITD Proxy Login
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/bin/bash /usr/local/bin/iitd-proxy.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> **Note:** Use `ExecStart=/bin/bash /usr/local/bin/iitd-proxy.sh` rather than calling the script directly — this avoids `Exec format error` issues on some systems.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable iitd-proxy
sudo systemctl start iitd-proxy
```

Verify it is running:

```bash
sudo systemctl status iitd-proxy
```

Expected output includes `Active: active (running)` and `proxy login` in the log.

To view detailed logs:

```bash
journalctl -u iitd-proxy.service -n 50
# Or follow in real time:
journalctl -u iitd-proxy.service -f
```

---

## Step 3: Set Proxy Environment Variables

Add the following to `~/.bashrc` (or `~/.profile` for login shells):

```bash
export http_proxy="http://proxy61.iitd.ac.in:3128"
export https_proxy="http://proxy61.iitd.ac.in:3128"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export no_proxy="localhost,127.0.0.1,*.iitd.ac.in"
export NO_PROXY="$no_proxy"
```

Reload:

```bash
source ~/.bashrc
```

> **Note:** When running commands with `sudo`, use `sudo -E` to preserve environment variables, e.g. `sudo -E apt update`.

---

## Step 4: Configure apt

`apt` does not read shell environment variables. Create a dedicated config file:

```bash
sudo nano /etc/apt/apt.conf.d/95proxy
```

Add:

```
Acquire::http::Proxy "http://proxy61.iitd.ac.in:3128";
Acquire::https::Proxy "http://proxy61.iitd.ac.in:3128";
```

Test:

```bash
sudo apt update
```

---

## Known Limitations

### External PPAs blocked by IITD proxy

Some external repositories (e.g. `ppa.launchpadcontent.net`) are blocked at the proxy level with a `503 ERR_CONNECT_FAIL` error. This is a firewall restriction and cannot be resolved through proxy configuration.

**Workaround — disable the affected PPA:**

```bash
sudo add-apt-repository --remove ppa:example/ppa
```

Re-enable it when connected off-campus or via VPN.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `status=203/EXEC` | Script not executable or bad shebang | `sudo chmod +x` and check for `\r` with `cat -A` |
| `Exec format error` in journal | systemd can't exec script directly | Use `ExecStart=/bin/bash /path/to/script.sh` |
| `error in login` in logs | Wrong credentials or proxy unreachable | Check credentials in script; verify network |
| `apt` 503 on external PPA | IITD proxy blocks the domain | Disable the PPA or use off-campus network |
| Proxy works interactively, fails under `sudo` | `sudo` strips env vars | Use `sudo -E` or set proxy in `/etc/environment` |

---

## Security Note

The `proxy.sh` script stores credentials in plaintext. Restrict its permissions:

```bash
sudo chmod 700 /usr/local/bin/iitd-proxy.sh
sudo chown root:root /usr/local/bin/iitd-proxy.sh
```

Avoid sharing the file or committing it to version control.
