---

# 🛠️ Downloader Stack Runbook

## 🟢 Daily Health Check
Run `docker ps` and verify:
* **Status:** All containers should be `(healthy)`.
* **Uptime:** Apps should have roughly the same uptime as the `vpn` container. If the VPN uptime is much lower, a "Zombie Network" event occurred and was (hopefully) healed.
* **Storage:** Verify the 14TB Knightrider mount: `df -h /mnt/knightrider`.

---

## 🔄 Maintenance & Updates
Since we utilize Proxmox daily snapshots, the update strategy is **"Update First, Ask Questions Later."**

### 1. Pre-Update (The Safety Net)
* Verify a Proxmox snapshot exists from the last 24 hours.
* Create a config backup: `cp docker-compose.yml docker-compose.yml.bak`.

### 2. Execution
```bash
# Pull newest images
docker compose pull

# Recreate containers (respects dependencies)
docker compose up -d

# Clean up old image bloat (keep that 256GB SSD lean)
docker image prune -f
```

### 3. Post-Update Triage (The 15-Minute Rule)
If the UI is unreachable (`Connection Reset`):
1. **Force Reconnect:** `docker compose up -d --force-recreate`.
2. **Check VPN Logs:** `docker logs vpn --tail 50` (Look for auth or handshake errors).
3. **Check Firewall:** `docker exec vpn iptables -L -n` (Ensure LAN subnets aren't dropped).
4. **The Revert:** If not fixed in 15 mins, **Restore Proxmox Snapshot**.

---

## 🆘 Troubleshooting Common Issues

### 🧌 The "Zombie" Network (Containers Up, No Internet)
* **Symptoms:** Sonarr can't find indexers; `curl` from host results in `Connection Reset`.
* **RCA:** VPN restarted, but apps didn't re-tether.
* **Fix:** `docker compose restart sonarr radarr prowlarr qbittorrent`.

### 🟠 The "Orange Flame" (Passive Mode)
* **Symptoms:** qBittorrent shows yellow connection icon; slow/no uploads.
* **RCA:** Port Manager failed to update the port in qBit or VPN didn't assign one.
* **Fix:** Check `docker logs port-manager`. Ensure `QBIT_USER` and `QBIT_PASS` match your UI credentials.

### 📁 "Path Not Found" (Sonarr/Radarr)
* **Symptoms:** Red bars in UI; "Missing" files that exist on disk.
* **RCA:** Case-sensitivity mismatch or hardlink "bubble" break.
* **Fix:** Ensure paths start with `/data/...` inside the app. Check if folder is `Band of Brothers` vs `Band Of Brothers`.

---

## 📈 Useful Commands
* **Inspect Health Detail:** `docker inspect --format='{{json .State.Health}}' sonarr | jq`
* **Check Disk Usage:** `docker system df`
* **Restart Autoheal:** `docker restart autoheal`
* **Inside VPN Shell:** `docker exec -it vpn /bin/sh`

---

### Pro-Tip for the LXC Host:
If you ever lose access to the **10.0.0.x** web UIs entirely after an LXC reboot, ensure the host-level tun device is ready:
`ls -l /dev/net/tun` (Should show `crw-rw-rw-`)

## 🏗️ Proxmox / Bare-Metal Recovery

### 1. The "Total Dark" Scenario (LXC won't start)
If the LXC fails to start after a Proxmox reboot or a kernel update:
* **Check the TUN Device:** Your VPN requires `/dev/net/tun`. If Proxmox lost this mapping, the VPN container will prevent the stack from starting.
  * *Fix:* Check the LXC configuration file on the Proxmox host: `vi /etc/pve/lxc/101.conf` (replace 101 with your ID). Ensure the mount for `/dev/net/tun` is still present.
* **Check Bind Mounts:** If the 14TB "Knightrider" drive isn't mounted on the Proxmox host, the LXC might hang on boot. 
  * *Fix:* Run `mount -a` on the Proxmox host and verify `/mnt/knightrider` is visible.

### 2. Restoring from Snapshot (The "Reset" Button)
If an update broke the database or you’ve made a configuration error you can't find:
1. Go to the **Proxmox Web UI**.
2. Select your **Downloader LXC (101)**.
3. Click the **Snapshots** tab.
4. Select the most recent **Daily Snapshot** (usually from 4:00 AM).
5. Click **Rollback**. 
   * *Warning:* This will revert all settings and the Docker database to that exact time. Any torrents added *after* the snapshot will need to be re-added.

### 3. Restoring from External Backup (Disaster Recovery)
If the Proxmox host's internal 256GB SSD fails, use your 5TB external backup drive:
1. In Proxmox, go to your **Backup Storage** (the 5TB drive).
2. Select the **Backups** tab.
3. Find the latest backup for the Downloader LXC.
4. Click **Restore**.
5. Assign a New ID (or the old one) and click **Restore**.
   * *Note:* Since the media lives on the 14TB Knightrider drive (which is not part of this backup), you will just need to re-mount that drive to the new LXC using a bind mount.

### 4. Re-Mounting the 14TB Drive (Post-Restore)
If you restore to a fresh LXC, you must re-link the media:
```bash
# On the Proxmox Host:
pct set <LXC_ID> -mp0 /mnt/knightrider,mp=/mnt/knightrider
```

---

### Pro-Tip: The "LXC Shell" Fail-Safe
If you can't SSH into the LXC but the Proxmox summary shows it is running, use the Proxmox host terminal to "jump" in:
```bash
pct enter <LXC_ID>
```
Once inside, you can run `docker compose down` or `vi docker-compose.yml` even if the network is completely broken.

