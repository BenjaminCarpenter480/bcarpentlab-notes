---
title: How to use Gnome Remote Desktop for headless login
date: 2025-10-03 19:00:00 +0000
#categories: [TOP_CATEGORY, SUB_CATEGORY]
tags: [gnome, gnome-remote-desktop, rdp, xrdp, remote-access, headless]     # TAG names should always be lowercase
---

If you just want to plug a machine into power and network and then create a Remote Desktop Protocol connection (RDP) to it without a monitor attached, GNOME's built‑in Remote Desktop can do that. GNOME Remote Desktop (GRD) is emerging as a native solution, moving beyond external tools like XRDP. My exploration into GRD below focused on its potential as a built-in RDP solution. This guide walks you through setting it up from the command line for remote login, focusing on the user experience and key considerations.

## What you'll get

- RDP access that works even with no physical monitor attached (headless).
- A service that starts at boot without anyone needing to log in locally
- Native GNOME desktop experience over RDP
- Support for multiple users without extra RDP configuration

## Prerequisites

- GNOME with gnome-remote-desktop (GNOME 44+ required for remote login).
- Sudo on the target machine.
- Open TCP 3389 between client and host.

Note: exact commands and flags can vary slightly by distro/version. If a command isn't found, check your package manager and grdctl --help.

>[!Important] Important version note: For headless login using only GRD, you need GNOME 46 or higher. This means:
> Ubuntu 24.04+ ✅
> RHEL/Rocky 10+ ✅
> Ubuntu 22.04 and below ❌
> RHEL/Rocky 9 ❌

If you're on an older GNOME version, XRDP is a good alternative for headless RDP  access to your machine.

## The two-step login

GRD with remote login requires that we authenticate twice. The double authentication might seem weird at first, but here's why it works this way:

- **System credentials:** Are used to login to the classic XRDP portal and the GNOME login screen. Think of these as building access. Everyone uses the same "key" to get into the building
- **User credentials:** These will be used to access your GNOME desktop from the Gnome login screen. These are your personal office key once you're inside the building

This approach lets GNOME maintain its security model while allowing headless operation. The GNOME developers are discussing ways to streamline this in the future.

## Step 1: Install the packages

```shell
# Debian family/Ubuntu
sudo apt update
sudo apt install gnome-desktop gnome-remote-desktop

# RedHat family/Rocky
sudo dnf install gnome-desktop gnome-remote-desktop

```

Verify everything's there:

```shell
gnome-shell --version
systemctl status gnome-remote-desktop.service
```

This should show some information about the service, even if it's inactive for now.

## Step 2: Enable system-level RDP

This is where GNOME Remote Desktop differs from per-user setups. We're configuring it at the system level so multiple users can connect:

Enable RDP at system level

```shell
sudo grdctl --system rdp enable
```

And then check the status of system-level RDP

```shell
sudo grdctl --system status
```

If the above doesn't work then verify that **gnome-remote-desktop** is installed, not just gnome-desktop. Another thing to try is using the gsettings:

```shell
gsettings set org.gnome.desktop.remote-desktop.rdp enable true
```

## Step 3: Set System RDP credentials

Here's the bit that provides some confusion and to me is a bit user-unfriendly. You need to set system-level RDP credentials that **all** users will use to see the login screen:

Set system-level credentials (it will prompt for password)

```shell
sudo grdctl --system rdp set-credentials <new-rdp-username>
```

You can also specify the password inline

```shell
sudo grdctl --system rdp set-credentials <new-rdp-username> <new-rdp-password>
```

**Important:** These aren't the same as user account passwords. Think of these as "door credentials" to get to the actual login screen.

If you are using a GNOME version before 46 then you will need to login locally and set the credentials once in your desktop settings GNOME Settings > Sharing > Remote Desktop. You can then continue headless but you will need to login on any machine restarts before the machine is remotely accessible.

## Step 4: Generate TLS certificates

To work properly a TLS certificate is needed for RDP. You can generate a self-signed one as below, if you are doing somethng more fancy than exposing locally or an experimental setup then consider using a proper CA-signed certificate:

Install certificate tools

```shell

sudo apt install winpr-utils  # Ubuntu/Debian
sudo dnf install winpr-tools  # Fedora/RHEL
```

Generate certificates for the gnome-remote-desktop user and configure the service to use them

```shell
sudo -u gnome-remote-desktop winpr-makecert -silent -rdp -path ~gnome-remote-desktop rdp-tls

sudo grdctl --system rdp set-tls-key ~gnome-remote-desktop/rdp-tls.key
sudo grdctl --system rdp set-tls-cert ~gnome-remote-desktop/rdp-tls.crt
```

## Step 5: Enable and start the service

```shell
sudo systemctl enable --now gnome-remote-desktop.service
```

Then make sure it's running with:

```shell
sudo systemctl status gnome-remote-desktop.service
```

## Step 6: Configure your firewall

On Debian family/Ubuntu using UFW

```shell
sudo ufw allow 3389/tcp
```

or firewalld for Red Hat family/Rocky

```shell
sudo firewall-cmd --add-service=rdp --permanent || sudo firewall-cmd --add-port=3389/tcp --permanent
sudo firewall-cmd --reload
```

You can verify the port is listening with

```shell
sudo ss -ntlp | grep 3389
```

which should show something like:

```shell
LISTEN   0        128              *:3389             *:*      users:(("gnome-remote-desktop",pid=1234,fd=20))
```

## Step 7: Set up users

(Create your users and) add users to the required groups:

Create a user and set password

```shell
sudo useradd -m testuser

sudo passwd testuser
```

Add users to the gnome-remote-desktop group to allow them to use RDP

```shell
sudo usermod -aG gnome-remote-desktop
```

>[!tip] Pro tip
> We are creating, starting, stopping and changing a lot of system daemons during this setup. If users don't appear in the login screen or other issues occur it might be worth restarting GDM or rebooting the machine to put things in the correct state.
>
> ```shell
> sudo systemctl restart gdm
> ```

## Step 8: Connect

Now comes the slightly unusual bit. When you connect with your RDP client:

1. **First authentication:** Use the system RDP credentials you set in step 3
2. **Second authentication:** You'll see the familiar GNOME login screen where you enter your actual user credentials

From your RDP client of choice (Remmina, Microsoft Remote Desktop, FreeRDP, etc.) connect to:

- Host: `<machine ip or hostname>`
- Protocol: RDP
- Username/password: what you set above in step 3

## Troubleshooting common issues

### Check the status of system-level RDP

```shell
sudo grdctl --system status
```

### Can't connect at all

- Check the service is running: `sudo systemctl status gnome-remote-desktop.service`
- Verify port 3389 is open: `sudo ss -ntlp | grep 3389`
- Make sure your firewall rules are correct

### Certificate errors

```shell
# Regenerate certificates
sudo -u gnome-remote-desktop winpr-makecert -silent -rdp -path ~gnome-remote-desktop rdp-tls
sudo grdctl --system rdp set-tls-key ~gnome-remote-desktop/rdp-tls.key
sudo grdctl --system rdp set-tls-cert ~gnome-remote-desktop/rdp-tls.crt
sudo systemctl restart gnome-remote-desktop.service
```

### Connection immediately cancelled

- Verify system credentials are set: `sudo grdctl --system status`
- Look for "Username: (hidden)" and "Password: (hidden)" in the output

If you have only just added or modified users then you may need to restart GDM to refresh the user list:

```shell
sudo systemctl restart gdm
```

## Important security notes

- Use strong passwords for both system and user credentials
- Don't expose port 3389 directly to the internet
- Consider using a VPN or restricting access to trusted networks
- The system RDP password is shared among all users, so choose it carefully

## When to use something else

GNOME Remote Desktop works great for:

- Small teams needing native GNOME experience
- Environments where you control the OS versions
- Situations where Wayland compatibility matters

Consider XRDP instead if:

- You're stuck with older GNOME versions (pre-46)
- The two-step login is more complex than an existing XRDP setup
- You require extensive RDP customization
- You want a more traditional RDP experience that is more mature than the very recent GRD implementation

Whilst GRD is still maturing and might not be suitable just yet for many scenarios, the good news is that the underlying technology keeps improving, and the user experience should get smoother over time.

<!-- Inline callouts CSS/JS; move to your layout for site‑wide use -->
<style>
.callout {
  margin: 1rem 0;
  padding: 0.75rem 1rem;
  border-left: 0.3rem solid #6b7280; /* default slate */
  border-radius: 6px;
  background: rgba(0,0,0,.03);
}
@media (prefers-color-scheme: dark) {
  .callout { background: rgba(255,255,255,.05); }
}
.callout__title {
  font-weight: 600;
  margin: 0 0 .25rem 0;
  text-transform: none;
}
.callout--important { border-left-color: #ef4444; } /* red */
.callout--tip       { border-left-color: #22c55e; } /* green */
.callout--note      { border-left-color: #0ea5e9; } /* sky */
.callout--warning   { border-left-color: #eab308; } /* amber */
</style>
<script>
(() => {
  const titleMap = { important: 'Important', tip: 'Tip', note: 'Note', warning: 'Warning' };
  document.querySelectorAll('blockquote').forEach(bq => {
    const firstP = bq.querySelector(':scope > p');
    if (!firstP) return;
    const m = firstP.textContent.trim().match(/^\[!(\w+)\]\s*(.*)$/i);
    if (!m) return;
    const kind = m[1].toLowerCase();
    const titleText = m[2] || titleMap[kind] || kind;
    firstP.innerHTML = firstP.innerHTML.replace(/^\s*\[!\w+\]\s*/i, '');
    const wrap = document.createElement('div');
    wrap.className = `callout callout--${kind}`;
    const titleEl = document.createElement('div');
    titleEl.className = 'callout__title';
    titleEl.textContent = titleText;
    wrap.appendChild(titleEl);
    while (bq.firstChild) wrap.appendChild(bq.firstChild);
    bq.replaceWith(wrap);
  });
})();
</script>
