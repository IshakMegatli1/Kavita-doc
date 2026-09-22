# Self-Hosted Kavita on an Ubuntu Mini-PC

Notes on how I set up [Kavita](https://www.kavitareader.com/) (a self-hosted reading server for books, manga, and comics) with Docker Compose on a mini-PC, and how I read the library on my phone with Readest over OPDS.

## Overview

| Item | Details |
|---|---|
| Hardware | Lenovo ThinkCentre M710q mini-PC |
| OS | Ubuntu |
| Runtime | Docker + Docker Compose |
| App | Kavita (`jvmilazz0/kavita`) |
| Phone reader | Readest (via OPDS) |
| Web UI | `http://<mini-pc-ip>:5000` |

### Storage layout

The mini-PC has two SSDs, and each one has a different job:

| Drive | Mount | Used for |
|---|---|---|
| NVMe (~115 GB) | `/` | OS and Kavita's config folder (database, covers, settings) |
| SATA SSD (~469 GB) | `/mnt/storage` | Media library (books, manga, comics) |

The config folder is small but does a lot of small reads, so it lives on the faster NVMe. The media is large and read sequentially, so it lives on the bigger drive. Keeping media on its own drive also means an OS reinstall doesn't touch the library.

```
~/kavita/
├── compose.yml
└── config/            # Kavita database, covers, settings (back this up)

/mnt/storage/
├── manga/
├── comics/
└── books/
    └── Library/       # books go in a subfolder, not the library root
```

## 1. Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

Log out and back in so the `docker` group applies. Until then, prefix docker commands with `sudo`.

Verify:

```bash
docker --version
groups        # should list "docker"
```

## 2. Create the folders first

Create the folders before starting the container. If a bind-mounted host folder doesn't exist, Docker creates it as **root**, and the container (running as UID 1000) then can't write to it.

```bash
mkdir -p ~/kavita/config
sudo mkdir -p /mnt/storage/{manga,comics,books}
sudo chown -R 1000:1000 /mnt/storage/{manga,comics,books}
```

Only chown the three media folders, not all of `/mnt/storage`, so other data on the drive is left alone.

## 3. Compose file

`~/kavita/compose.yml`:

```yaml
services:
  kavita:
    image: jvmilazz0/kavita:latest
    container_name: kavita
    user: "1000:1000"                    # match `id -u` / `id -g`
    environment:
      - TZ=America/Toronto
    volumes:
      - ./config:/kavita/config          # on the NVMe
      - /mnt/storage/manga:/manga:ro
      - /mnt/storage/comics:/comics:ro
      - /mnt/storage/books:/books:ro
    ports:
      - "5000:5000"
    restart: unless-stopped
```

Notes:

- `user: "1000:1000"` runs Kavita as my normal user. Check yours with `id`.
- The `:ro` mounts make the media read-only inside the container. Kavita only reads and serves files; it doesn't download anything. Remove `:ro` only if I want Kavita to delete or change files on disk.
- The path after the colon (`/books`) is what Kavita sees inside the container. That's the path used when adding a library in the UI.

## 4. Start and verify

```bash
cd ~/kavita
docker compose config        # validate the YAML
docker compose up -d
docker compose ps
docker compose logs --tail 50
```

Find the mini-PC's IP and open the UI from another device on the network:

```bash
hostname -I
```

If the page doesn't load and UFW is enabled:

```bash
sudo ufw allow 5000/tcp
```

## 5. First-time setup in Kavita

1. Open `http://<mini-pc-ip>:5000` and register. **The first account becomes the admin**, so do this right away.
2. Go to **Server Settings → Libraries → Add Library**.
3. Add one library per media type, using the container paths:
   - `/manga` as type **Manga**
   - `/comics` as type **Comic**
   - `/books` as type **Book**
4. Run **Scan Library** from the library's ⋮ menu.
5. Optional: enable folder watching in Server Settings so new files appear without a manual scan.

## 6. Folder structure rules

Kavita **does not scan files sitting directly in the library root**. If it finds one, it shows:

> One or more folders contains files at the root. Kavita does not support this.

Every file needs at least one folder above it:

- **Books:** any subfolder works. I use `/mnt/storage/books/Library/` as a catch-all. Epubs are identified by their embedded metadata (title, author), so the folder name matters less.
- **Manga and comics:** one folder per series, e.g. `manga/Series Name/Series Name v01.cbz`. Kavita groups volumes by folder, so this matters.

Supported formats include EPUB, PDF, CBZ, CBR, and ZIP.

To move a file into the library:

```bash
mkdir -p /mnt/storage/books/Library
mv "/path/to/book.epub" /mnt/storage/books/Library/
```

Quote filenames that contain spaces or special characters. Bind mounts are live, so no container restart is needed after adding files. Just rescan.

## 7. Reading on a phone

### Browser

Open `http://<mini-pc-ip>:5000` on the phone (same Wi-Fi) and use Kavita's web reader. "Add to Home Screen" makes it behave like an app.

### Readest via OPDS

OPDS is a catalog format that lets reader apps browse and download from the server. [Readest](https://readest.com/) is a free, open-source ebook reader (Android, iOS, desktop, and web) with built-in OPDS support, so it can browse the Kavita library and download books straight to the phone for offline reading.

#### Prerequisites

- Kavita is running and the library has been scanned (books show up in the web UI).
- The phone is on the same Wi-Fi as the mini-PC. Away from home, see [Remote access](#optional-next-steps).
- The URL test in step 3 below works from the phone's browser.

#### Step 1: Get the OPDS URL from Kavita

1. Log in to the Kavita web UI.
2. Click the profile icon → **Settings** and find the **OPDS** section. If OPDS is disabled, an admin needs to enable it first under **Server Settings**.
3. Copy the full URL. It looks like this:
   ```
   http://<mini-pc-ip>:5000/api/opds/<api-key>
   ```

The API key in the URL is what authenticates the request, so no separate username or password is needed.

#### Step 2: Add the catalog in Readest

1. Open Readest and go to its OPDS catalog settings. The exact menu location varies between versions, so look under the main menu or settings for **OPDS Catalogs**.
2. Add a new catalog.
3. Name it `Kavita` and paste the URL from step 1.
4. Leave username and password empty.
5. Save, then open the catalog.

#### Step 3: Test and download

Before adding the URL to Readest, it helps to test it in the phone's browser. If the page shows XML, the URL is fine and any problem is in the app.

In Readest, open the Kavita catalog, browse to a library or search for a book, and download it. The book is saved into Readest's local library and works offline.

#### Readest cloud sync (optional)

Readest has an optional cloud sync service that only applies if you sign in to a Readest account. If you never sign in, nothing is uploaded. If you do sign in, sync is controlled per category (books, reading progress, annotations, app settings, fonts) under the **Data Sync** settings.

For this setup, cloud sync isn't needed, because Kavita already holds the books. Things to know if you enable it:

- **Book sync** uploads book files to Readest's cloud. I turned it off, since Kavita already stores the originals and they can be re-downloaded through OPDS at any time.
- **App-settings sync** may also sync saved OPDS catalogs across devices, which would store the Kavita URL (including the API key) in the Readest account. With a local `192.168.x.x` address, the URL only works on the home network, so the risk is low. It matters more if Kavita is ever exposed to the internet.
- **Without sync**, reading progress and highlights stay on that one device.

#### Security notes

- The API key works like a password. Don't share the URL, and never commit it to a repo.
- To invalidate a leaked key, reset it in Kavita's user settings, then update the catalog URL in Readest.

#### Troubleshooting

| Problem | Fix |
|---|---|
| Catalog won't load in Readest | Test the URL in the phone's browser. If that fails too, check that the phone is on the same Wi-Fi and that UFW allows the port: `sudo ufw allow 5000/tcp` |
| Worked before, now fails | The mini-PC's IP may have changed. Check with `hostname -I`, update the URL, and set a DHCP reservation in the router |
| App rejects `http://` addresses | Use Tailscale or an HTTPS reverse proxy instead of plain HTTP |
| Catalog loads but a library is empty | Rescan the library in Kavita and check that files aren't sitting at the library root (see section 6) |
| No OPDS section in Kavita settings | An admin needs to enable OPDS under Server Settings |

## Troubleshooting (issues I hit)

**`permission denied while trying to connect to the docker API`**
My user wasn't in the `docker` group yet. Fix:
```bash
sudo usermod -aG docker $USER
```
Then log out and back in. In the meantime, use `sudo docker ...`.

**Container restart-looping with `cp: cannot create regular file '/kavita/config/appsettings.json': Permission denied`**
Docker had created `~/kavita/config` as root, so the container (UID 1000) couldn't write to it.
```bash
ls -ld ~/kavita/config                  # owner was root
sudo chown -R 1000:1000 ~/kavita/config
sudo docker compose restart
```

**Library scan shows nothing / "files at the root" error**
Files were sitting directly in `/mnt/storage/books`. Move them into a subfolder (see section 6) and rescan.

**Checking what the container can see**
```bash
docker compose exec kavita ls -R /books
```

**Compose file edits not taking effect**
`docker compose restart` ignores changes to the compose file. Use:
```bash
docker compose up -d
```

## Maintenance

- **Update Kavita:**
  ```bash
  cd ~/kavita
  docker compose pull && docker compose up -d
  ```
- **Back up `~/kavita/config`.** It holds the database, users, and reading progress. The media lives on a separate drive.
- **Reserve the mini-PC's IP** in the router (DHCP reservation) so the saved OPDS URL doesn't break if the address changes.

## Optional next steps

- **Remote access with [Tailscale](https://tailscale.com/):** install it on the mini-PC and the phone, then use the mini-PC's Tailscale address instead of the local IP. This avoids exposing port 5000 to the internet.
- **Reverse proxy with HTTPS** (e.g. Caddy) if I ever want to share the library with others. Don't forward port 5000 directly, since it's plain HTTP.
- **Automated downloads:** Kavita doesn't fetch content itself. A separate tool such as Suwayomi could write CBZ files into `/mnt/storage/manga`, and Kavita would pick them up on the next scan.
