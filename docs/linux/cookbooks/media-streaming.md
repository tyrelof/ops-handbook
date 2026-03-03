# Cookbook: Media Streaming (Jellyfin & Navidrome)

> [!IMPORTANT]
> This guide replaces legacy Ampache with modern standards: **Jellyfin** for universal media (Video/Audio) and **Navidrome** for lightweight, dedicated music streaming. Optimized for Ubuntu 22.04/24.04 LTS.

## 1. Jellyfin: The Open Source Media System

Jellyfin is the leading free and open-source alternative to Plex and Emby. It handles Movies, TV Shows, Music, and Live TV.

### Installation (Official Repository)
```bash
sudo apt update
sudo apt install curl gnupg -y
# Add Jellyfin GPG key and repo
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://repo.jellyfin.org/jellyfin_team.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/jellyfin.gpg
echo "deb [arch=$( dpkg --print-architecture ) signed-by=/etc/apt/keyrings/jellyfin.gpg] https://repo.jellyfin.org/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/jellyfin.list

sudo apt update
sudo apt install jellyfin -y
```

### Initial Setup
1.  Access the web interface at `http://your-server-ip:8096`.
2.  Follow the wizard to set up your admin account.
3.  **Library Creation**: Point Jellyfin to your media folders (e.g., `/srv/media/movies`).
4.  **Permissions**: Ensure the `jellyfin` user has read access to your media:
    `sudo chown -R jellyfin:jellyfin /srv/media`

---

## 2. Navidrome: Dedicated Audio Streaming

Navidrome is a lightweight, high-performance music server compatible with the Subsonic API. It is ideal for very large music collections.

### Installation (Binary)
```bash
# Create directory
sudo mkdir -p /opt/navidrome /var/lib/navidrome
# Download latest release
wget https://github.com/navidrome/navidrome/releases/download/v0.52.5/navidrome_0.52.5_Linux_x86_64.tar.gz
sudo tar -xvzf navidrome_0.52.5_Linux_x86_64.tar.gz -C /opt/navidrome
```

### Configuration (`/var/lib/navidrome/navidrome.toml`)
```toml
MusicFolder = "/srv/music"
DataFolder = "/var/lib/navidrome"
LogLevel = "info"
```

### Service Setup
Create `/etc/systemd/system/navidrome.service`:
```ini
[Unit]
Description=Navidrome Music Server
After=remote-fs.target network.target

[Service]
User=navidrome
Group=navidrome
Type=simple
ExecStart=/opt/navidrome/navidrome --config /var/lib/navidrome/navidrome.toml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
*Start:* `sudo systemctl enable --now navidrome`

---

## 3. Hardware Acceleration (Transcoding)

To stream high-definition video without pinning your CPU, you should enable hardware transcoding.

### Intel QuickSync (QSV)
Install drivers for Intel integrated graphics:
```bash
sudo apt install intel-media-va-driver-non-free vainfo -y
# Add jellyfin user to render group
sudo usermod -aG render jellyfin
```

### NVIDIA (NVENC)
Install the proprietary NVIDIA drivers and the container toolkit if using Docker:
```bash
sudo apt install nvidia-driver-535 -y
```

---

## 4. Modern Clients

Use these modern apps to connect to your servers:
- **Jellyfin**: Jellyfin mobile app (iOS/Android), Findroid (Android), Swiftfin (iOS/AppleTV).
- **Navidrome (Subsonic API)**: Symfonium (Android - Highly Recommended), Ample (iOS), Substreamer.

---

## 5. Troubleshooting

- **Check Process**: `sudo systemctl status jellyfin`
- **Error Log**: `journalctl -u jellyfin -f`
- **FFmpeg Issues**: If transcoding fails, check `/var/log/jellyfin/` for FFmpeg logs.
