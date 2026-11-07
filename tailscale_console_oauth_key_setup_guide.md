*Reference: [https://github.com/nextcloud/all-in-one/discussions/5439](https://github.com/nextcloud/all-in-one/discussions/5439)*

## 1. Access Control Lists (ACLs) & Service Tags

Devices deployed as server containers should be owned by **Tags** rather than individual user accounts. This disables key expiration for the service and enforces strict access rules.

### 1.1 Add Service Tags to ACL Policy

1. Open the [Tailscale ACL Editor](https://login.tailscale.com/admin/acls/file).
2. Locate the `"tagOwners"` block (or create it if it doesn't exist).
3. Add your service tags (e.g., `tag:nextcloud`, `tag:vaultwarden`, `tag:plex`):

```JSON
"tagOwners": {
    "tag:nextcloud": ["autogroup:admin"],
    "tag:vaultwarden": ["autogroup:admin"],
    "tag:plex": ["autogroup:admin"]
}
```

4. Click **Save** to apply the policy.

## 2. Generate Service OAuth Keys

OAuth Client keys allow Docker containers (like `tailscale/tailscale`) to automatically authenticate and register themselves to your Tailnet under a specific tag without manual browser logins.

### 2.1 Create OAuth Client

1. Navigate to [Tailscale Settings -> OAuth Clients](https://login.tailscale.com/admin/settings/oauth) (or **Trust Credentials** -> **OAuth**).
2. Click **Generate OAuth client**.
3. Configure the settings:
    - **Description:** Enter a clear name (e.g., `Docker - Nextcloud`).
    - **Scopes:** Under **Auth Keys**, select **Write**.
    - **Tags:** Select the service tag created in Step 1 (e.g., `tag:nextcloud`).
4. Click **Generate Client**.
5. Copy the generated secret key (starts with `tskey-client-...`).

> ⚠️ **Important:** Store this key safely. You cannot view it again after closing the window. Use this key as the `TS_AUTH_KEY` environment variable in your `compose.yml` files.

## 3. Using OAuth Keys in Docker Compose

When adding Tailscale sidecars or containers to your `compose.yml` stacks, pass the OAuth key and specify the matching tag in `TS_EXTRA_ARGS`:

```YAML
  tailscale:
    image: tailscale/tailscale:stable
    environment:
      TS_HOSTNAME: nextcloud                            # Hostname shown in Tailscale console
      TS_AUTH_KEY: tskey-client-YOUR_OAUTH_KEY_HERE     # OAuth secret key from Step 2
      TS_EXTRA_ARGS: --advertise-tags=tag:nextcloud --reset
    restart: unless-stopped
    volumes:
      - tailscale_data:/var/lib/tailscale
```

## 4. Tailscale Device Cleanup & Reset (If Redeploying)

If you delete and rebuild a container service, Tailscale may retain the old machine entry, preventing new instances from binding cleanly.

### 4.1 Remove Stale Machines

1. Go to the [Tailscale Machines Page](https://login.tailscale.com/admin/machines).
2. Locate the old machine entry (e.g., `nextcloud`).
3. Click the **three dots (...)** on the far right -> select **Remove...** -> confirm deletion.
4. Restart the client machine or flush your DNS/route cache if connection issues persist.