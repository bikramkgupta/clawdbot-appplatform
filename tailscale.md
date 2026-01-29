# Tailscale Setup Guide

Tailscale creates a secure, private network (called a "tailnet") between your devices using WireGuard encryption. Clawdbot uses Tailscale so your AI assistant is only accessible to you — never exposed to the public internet.

## Quick Start

### 1. Create a Tailscale Account

1. Go to [tailscale.com](https://tailscale.com)
2. Click **Get Started** and sign up (free tier works great)
3. You'll land in the **Admin Console** at `login.tailscale.com/admin`

### 2. Generate an Auth Key

The auth key allows Clawdbot to automatically join your tailnet during deployment.

1. In the Admin Console, go to **Settings** (top navigation)
2. Under **Personal Settings**, click **Keys**
3. Click **Generate auth key...**

Configure the key with these recommended settings:

| Setting | Value | Why |
|---------|-------|-----|
| **Description** | `Clawdbot` or `AppPlatform` | Identify what the key is for |
| **Reusable** | ✅ On | Allows redeployments without new keys |
| **Expiration** | 90 days | Maximum allowed; regenerate before expiry |
| **Ephemeral** | ❌ Off | Keep the device registered after restarts |
| **Tags** | `tag:clawdbot` | Enables ACL-based access control |

4. Click **Generate key**
5. **Copy the key immediately** — it won't be shown again
6. Use this as your `TS_AUTHKEY` when deploying

> **Key format:** `tskey-auth-xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

## Using Tags for Access Control

Tags make managing access much easier, especially as you add more devices to your tailnet.

### Why Use Tags?

Instead of writing access rules for individual devices, you can:
- Tag devices by purpose (e.g., `tag:clawdbot`, `tag:server`, `tag:dev`)
- Write ACL rules that apply to all devices with that tag
- Easily add/remove devices from groups

### Setting Up Tags

1. Go to **Access controls** in the Admin Console
2. Add a `tagOwners` section to define who can use each tag:

```json
{
  "tagOwners": {
    "tag:clawdbot": ["autogroup:admin"]
  }
}
```

This allows admins to assign the `tag:clawdbot` tag to devices.

## Access Control Rules

To access Clawdbot from your devices, you need ACL rules that permit the traffic.

### Example: Allow Your User Access to Clawdbot

Add this to your access control policy:

```json
{
  "tagOwners": {
    "tag:clawdbot": ["autogroup:admin"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["your-email@example.com"],
      "dst": ["tag:clawdbot:*"]
    }
  ]
}
```

Replace `your-email@example.com` with your Tailscale login email.

### Example: Complete ACL Policy

Here's a working policy that allows SSH and HTTP access to tagged devices:

```json
{
    "grants": [
        {
            "src": ["*"],
            "dst": ["*"],
            "ip":  ["*"]
        }
    ],
    "tagOwners": {
        "tag:clawdbot": ["autogroup:admin"]
    },
    "ssh": [
        {
            "action": "accept",
            "src":    ["autogroup:member"],
            "dst":    ["tag:clawdbot"],
            "users":  ["*"]
        }
    ],
    "acls": [
        {
            "action": "accept",
            "src":    ["autogroup:member"],
            "dst":    ["tag:clawdbot:22", "tag:clawdbot:8080", "tag:clawdbot:18789"]
        }
    ],
    "nodeAttrs": [
        {
            "target": ["autogroup:member"],
            "attr":   ["funnel"]
        }
    ]
}
```

**Key points:**
- `"users": ["*"]` allows SSH as any user (root, clawdbot, etc.)
- Port 18789 is the Clawdbot gateway
- `tag:clawdbot` should match your `TS_AUTHKEY` tag

### Enabling SSH in the Container

Tailscale SSH must be enabled inside the container after deployment:

1. **Via DO App Platform Console** (doctl or web console):
   ```bash
   doctl apps console <app-id> <component>
   # Then run: tailscale set --ssh
   ```

2. **Via SDK** (for automation):
   ```python
   from do_app_sandbox import Sandbox
   app = Sandbox.get_from_id(app_id="<app-id>", component="clawdbot")
   app.exec("tailscale set --ssh")
   ```

After enabling, verify in Tailscale Admin Console — the machine should show an "SSH" badge.

## Connecting Your Devices

### Install Tailscale on Your Devices

To access Clawdbot, install Tailscale on the devices you'll use:

| Platform | Install |
|----------|---------|
| **macOS** | [Download from Mac App Store](https://apps.apple.com/app/tailscale/id1475387142) or `brew install tailscale` |
| **Windows** | [Download installer](https://tailscale.com/download/windows) |
| **iOS** | [App Store](https://apps.apple.com/app/tailscale/id1470499037) |
| **Android** | [Play Store](https://play.google.com/store/apps/details?id=com.tailscale.ipn) |
| **Linux** | [Installation guide](https://tailscale.com/download/linux) |

### Verify Connection

Once Clawdbot is deployed and your device has Tailscale running:

1. Open the Tailscale app and ensure you're connected
2. Go to the Admin Console → **Machines** to see all devices
3. Find `clawdbot` in the list (it should show "Connected")
4. Access Clawdbot at: `https://clawdbot.<your-tailnet>.ts.net`

## Enterprise Laptop Considerations

If you're on a corporate laptop with existing VPN software:

> **Recommendation:** Use a VM (virtual machine) for Tailscale to avoid conflicts with enterprise VPN configurations.

Common issues on enterprise devices:
- Split tunneling conflicts
- DNS resolution problems
- Network policy restrictions

**Solutions:**
1. **Use a VM** (Parallels, VMware, VirtualBox) with Tailscale installed inside
2. **Use a personal device** (phone, personal laptop) for accessing Clawdbot
3. **Check with IT** if Tailscale is approved for your organization

## Accessing Clawdbot

Once everything is set up:

| Access Method | URL |
|---------------|-----|
| **Web UI** | `https://clawdbot.<your-tailnet>.ts.net` |
| **SSH** | `ssh clawdbot@clawdbot.<your-tailnet>.ts.net` |
| **Direct IP** | Find in Admin Console → Machines (e.g., `100.x.y.z`) |

Your tailnet name is shown in the Admin Console header (e.g., `yourname.github` → `yourname-github.ts.net`).

## Troubleshooting

### Device not appearing in Machines list

- Check that `TS_AUTHKEY` is correct in your deployment
- Verify the auth key hasn't expired
- Check App Platform logs for Tailscale errors

### Can't connect to Clawdbot

- Ensure Tailscale is running on your device (check the menu bar/system tray icon)
- Verify ACL rules allow your user to reach `tag:clawdbot`
- Try accessing via direct IP first: `http://100.x.y.z:443`

### Auth key expired

1. Generate a new auth key in Settings → Keys
2. Update `TS_AUTHKEY` in App Platform environment variables
3. Redeploy the app

## Links

- [Tailscale Getting Started](https://tailscale.com/kb/1017/install/)
- [Auth Keys Documentation](https://tailscale.com/kb/1085/auth-keys/)
- [ACL Tags](https://tailscale.com/kb/1068/acl-tags/)
- [Access Control Lists](https://tailscale.com/kb/1018/acls/)
- [Tailscale SSH](https://tailscale.com/kb/1193/tailscale-ssh/)
