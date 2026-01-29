# DigitalOcean Spaces Setup Guide

DigitalOcean Spaces provides S3-compatible object storage for persisting Clawdbot's state. Without Spaces, your configuration, sessions, and Tailscale identity are lost on every redeploy.

## Why Spaces?

App Platform containers are ephemeral — they have no persistent disk storage. Spaces solves this by:

- **Real-time SQLite backup** via Litestream (memory/search index)
- **Periodic state backup** every 5 minutes (config, sessions, JSON state)
- **Tailscale identity preservation** (avoids re-authentication on redeploy)

## Quick Start

### 1. Create a Space (Bucket)

1. Log in to your [DigitalOcean Dashboard](https://cloud.digitalocean.com)
2. Click **Spaces Object Storage** in the left sidebar
3. Click **Create a Spaces Bucket**
4. Configure your Space:

| Setting | Recommended Value | Notes |
|---------|-------------------|-------|
| **Datacenter Region** | Same as your App Platform app | Reduces latency (e.g., `tor1` for Toronto) |
| **CDN** | ❌ Disable | Not needed for backups |
| **Bucket Name** | `clawdbot-backup` | Must be globally unique |
| **Project** | Your project | Keep resources organized |

5. Click **Create a Spaces Bucket**

> **Note:** Bucket names are globally unique across all DigitalOcean customers. If `clawdbot-backup` is taken, try `clawdbot-backup-yourname`.

### 2. Generate Spaces Access Keys

Spaces uses access keys (similar to AWS S3) for authentication.

1. Go to **API** in the left sidebar (or [click here](https://cloud.digitalocean.com/account/api/spaces))
2. Scroll down to **Spaces Keys**
3. Click **Generate New Key**
4. Enter a name: `clawdbot` or `litestream`
5. Click **Create Access Key**
6. **Copy both keys immediately** — the secret is only shown once!

You'll get two values:

| Key | Example | Used For |
|-----|---------|----------|
| **Access Key** | `DO00XXXXXXXXXXXXXX` | `LITESTREAM_ACCESS_KEY_ID` |
| **Secret Key** | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | `LITESTREAM_SECRET_ACCESS_KEY` |

### 3. Find Your Spaces Endpoint

The endpoint depends on which region you created your Space in:

| Region | Endpoint |
|--------|----------|
| New York (nyc3) | `nyc3.digitaloceanspaces.com` |
| San Francisco (sfo3) | `sfo3.digitaloceanspaces.com` |
| Amsterdam (ams3) | `ams3.digitaloceanspaces.com` |
| Singapore (sgp1) | `sgp1.digitaloceanspaces.com` |
| Frankfurt (fra1) | `fra1.digitaloceanspaces.com` |
| Toronto (tor1) | `tor1.digitaloceanspaces.com` |
| Sydney (syd1) | `syd1.digitaloceanspaces.com` |
| Bangalore (blr1) | `blr1.digitaloceanspaces.com` |

You can also find the endpoint in your Space's **Settings** tab.

## Configure Clawdbot

Add these environment variables to your App Platform deployment:

| Variable | Value | Example |
|----------|-------|---------|
| `SPACES_ENDPOINT` | Your region endpoint | `tor1.digitaloceanspaces.com` |
| `SPACES_BUCKET` | Your bucket name | `clawdbot-backup` |
| `LITESTREAM_ACCESS_KEY_ID` | Access Key | `DO00XXXXXXXXXXXXXX` |
| `LITESTREAM_SECRET_ACCESS_KEY` | Secret Key | (your secret key) |

### Via Deploy Button

When deploying via the "Deploy to DO" button, you'll be prompted to enter these values.

### Via doctl CLI

```bash
doctl apps update <app-id> \
  --set-env SPACES_ENDPOINT=tor1.digitaloceanspaces.com \
  --set-env SPACES_BUCKET=clawdbot-backup \
  --set-env LITESTREAM_ACCESS_KEY_ID=DO00XXXXXXXXXXXXXX \
  --set-env LITESTREAM_SECRET_ACCESS_KEY=your-secret-key
```

### Via App Platform Dashboard

1. Go to your app in the [App Platform dashboard](https://cloud.digitalocean.com/apps)
2. Click **Settings** → **App-Level Environment Variables**
3. Add each variable (mark secrets as "Encrypt")
4. Click **Save** and redeploy

## How Backup Works

Once configured, Clawdbot automatically handles backup and restore:

### On Startup
```
1. Check for state-backup.tar.gz in Spaces
2. If found, restore JSON state files
3. Restore SQLite via Litestream (if replica exists)
4. Start the gateway
```

### While Running
```
- Litestream: Continuous SQLite replication (~1 second lag)
- s3cmd: State backup every 5 minutes
```

### On Shutdown
```
1. Trap SIGTERM signal
2. Create final state backup
3. Exit gracefully
```

## What Gets Backed Up

| Data | Method | Location in Spaces |
|------|--------|-------------------|
| SQLite memory DB | Litestream | `clawdbot/memory-main/` |
| Config, sessions | tar + s3cmd | `clawdbot/state-backup.tar.gz` |

## Verify Backup is Working

### Check Spaces Bucket Contents

1. Go to your Space in the DigitalOcean dashboard
2. You should see a `clawdbot/` folder after first deployment
3. Inside: `memory-main/` (Litestream) and `state-backup.tar.gz`

### Check App Logs

Look for these messages in App Platform logs:

```
Restoring state from Spaces backup...
Downloading state backup...
Restoring SQLite from Litestream...
Mode: Litestream + state backup enabled
```

If you see `Mode: ephemeral (no persistence)`, the Spaces credentials aren't configured correctly.

## Cost

Spaces pricing (as of 2024):

| Resource | Cost |
|----------|------|
| Storage | $5/mo for 250 GB |
| Bandwidth | 1 TB outbound included |
| Overage | $0.02/GB storage, $0.01/GB bandwidth |

For Clawdbot backups, you'll typically use < 1 GB, so the minimum $5/mo covers it.

## Troubleshooting

### "No state backup found (first deployment)"

This is normal on first deploy. Backups will start after 5 minutes.

### "Mode: ephemeral (no persistence)"

One or more Spaces variables are missing or incorrect. Check:
- All 4 variables are set (`SPACES_ENDPOINT`, `SPACES_BUCKET`, `LITESTREAM_ACCESS_KEY_ID`, `LITESTREAM_SECRET_ACCESS_KEY`)
- No typos in the endpoint (e.g., `tor1` not `tor`)
- Access key and secret are correct

### "Access Denied" errors

- Verify the access key has permission to the bucket
- Check that the bucket name matches exactly
- Ensure the endpoint matches the bucket's region

### Bucket doesn't exist

- Bucket names are case-sensitive
- Double-check the exact name in the Spaces dashboard

## Security Best Practices

1. **Use dedicated keys** — Create Spaces keys specifically for Clawdbot, not your main account keys
2. **Don't share keys** — Each deployment should have its own keys
3. **Rotate periodically** — Generate new keys every few months
4. **Restrict bucket access** — Don't make the bucket public

## Links

- [Spaces Documentation](https://docs.digitalocean.com/products/spaces/)
- [Spaces Pricing](https://www.digitalocean.com/pricing/spaces)
- [How to Create a Space](https://docs.digitalocean.com/products/spaces/how-to/create/)
- [How to Manage Access Keys](https://docs.digitalocean.com/products/spaces/how-to/manage-access/)
- [Litestream Documentation](https://litestream.io/guides/digitalocean-spaces/)
