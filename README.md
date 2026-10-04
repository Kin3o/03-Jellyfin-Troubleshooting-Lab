# Jellyfin Troubleshooting Lab

This companion write-up records the problems encountered while building the Debian Jellyfin VM. These were normal first-lab issues: installer interfaces were unfamiliar, minimal Debian behaved differently from a desktop distribution, and several paths that looked similar served very different purposes.

The purpose of this document is not merely to list fixes. It explains what each symptom meant, how the cause was isolated, how the solution was verified, and what a beginner can carry into the next project.

Return to the complete build procedure in [02-Jellyfin-Server-Build-Walkthrough](https://github.com/Kin3o/02-Jellyfin-Server-Build-Walkthrough), or review the project summary in [01-JellyFin-Over-Proxmox-Home-Lab-Overview](https://github.com/Kin3o/JellyFin-Over-Proxmox-Home-Lab-Overview).

## 1. Understanding asterisks in Debian software selection

### Symptom

The Debian installer displayed a numbered list of software collections. Debian desktop environment, GNOME, and standard system utilities had asterisks beside them. It was not obvious how to deselect individual entries.

### Cause

An asterisk means an item is selected. This particular text-mode prompt accepts a replacement list of selected item numbers rather than requiring graphical checkbox navigation.

### Solution

Enter:

```text
11 12
```

This selects:

- `11`: SSH Server
- `12`: standard system utilities

It leaves the Debian desktop environment and GNOME unselected.

### Verification

Continue the installer and confirm Debian boots to a text login rather than a graphical desktop. After installation, confirm SSH is available or install `openssh-server` if needed.

### Lesson learned

Read the prompt legend before treating a text installer like a graphical menu. Installing only the packages required by a server reduces disk use and background resource consumption.

## 2. `sudo: command not found`

### Symptom

Running:

```bash
sudo apt update
```

returned:

```text
-bash: sudo: command not found
```

### Cause

The shell prompt already showed `root@jellyfin`, meaning the session had full administrative privileges. The minimal Debian installation also did not include the `sudo` package initially.

### Solution

While logged in as root, remove `sudo` from the command:

```bash
apt update
apt full-upgrade -y
apt install -y sudo curl qemu-guest-agent openssh-server
```

If a normal user should receive sudo privileges, add that account to the `sudo` group and then sign out and back in:

```bash
usermod -aG sudo YOUR_USERNAME
```

### Verification

From a new normal-user login, run:

```bash
sudo -v
```

Enter the normal user's password. No error indicates that sudo authorization is working.

### Lesson learned

Always check the shell prompt and current identity before adding `sudo`:

```bash
whoami
```

Root does not need sudo. A normal account does.

## 3. Jellyfin reported insufficient free space in `/tmp`

### Symptom

The Jellyfin installer reported approximately 2,007,672 KB free in `/tmp` but required 2,097,152 KB. At the same time, it reported more than 27 GB free for the Jellyfin data directory.

### Cause

The VM's system disk was not full. Debian had mounted `/tmp` as a memory-backed temporary filesystem. With 4 GB assigned to the VM, `/tmp` was approximately 2 GB and its usable free space fell slightly below Jellyfin's check.

The official installer checks both `/var/lib` and `/tmp` independently. Free disk space in one location cannot satisfy a capacity check for the other.

### Solution

> **Warning:** Shut down a VM before changing its memory allocation.

1. Shut down Debian:

   ```bash
   poweroff
   ```

2. In Proxmox, select VM `100`.
3. Open **Hardware > Memory > Edit**.
4. Increase memory from `4096 MiB` to `8192 MiB`.
5. Start the VM.
6. Check `/tmp`:

   ```bash
   df -h /tmp
   ```

7. Download and verify the installer again if rebooting cleared `/tmp`.

### Verification

Run the installer again and confirm the temporary-directory check reports at least 2 GB free. The Jellyfin installation should continue instead of exiting at the capacity check.

### Lesson learned

An error naming a path should be investigated at that path. Use `df -h /tmp` for `/tmp`, not only `df -h /` for the root disk.

## 4. Confusing `/tmp` with `/srv/media`

### Symptom

After running the Jellyfin installer from `/tmp`, it was unclear whether the Movies, TV Shows, and Music folders should also be created there.

### Cause

The current shell directory and the target path of a command are separate concepts. `/tmp` stores temporary working files and may be cleaned during reboot. `/srv` is appropriate for data served by the system.

Paths beginning with `/` are absolute. This command creates the folder under `/srv` regardless of the shell's current directory:

```bash
mkdir -p /srv/media/movies
```

### Solution

Create the library roots outside `/tmp`:

```bash
mkdir -p /srv/media/movies
mkdir -p /srv/media/tv
mkdir -p /srv/media/music
```

### Verification

```bash
ls -la /srv/media
```

Confirm `movies`, `music`, and `tv` are listed.

### Lesson learned

Temporary installation files and permanent application data belong in different locations. Never build a persistent media library under `/tmp`.

## 5. `cd *` returned `too many arguments`

### Symptom

Running `cd *` or `cd **` from `/tmp` returned:

```text
-bash: cd: too many arguments
```

### Cause

The shell expands `*` into every matching item in the current directory. The `cd` command accepts one destination, so several expanded names become too many arguments.

### Solution

Do not use a wildcard. Enter one exact path:

```bash
cd /srv/media
```

Changing directories was not required for the original `mkdir -p /srv/media/...` commands because those commands already used absolute paths.

### Verification

Run:

```bash
pwd
```

The result should be `/srv/media` if the `cd` command was used.

### Lesson learned

Use wildcards when a command intentionally accepts several files. Use an exact directory when changing the shell's working location.

## 6. Copying and pasting into the Proxmox console

### Symptom

Long commands had to be typed manually in the Proxmox noVNC console, increasing the chance of mistakes.

### Cause

QEMU Guest Agent improves host-to-guest management, but it does not add reliable clipboard sharing to a text-only Linux console. Proxmox noVNC is valuable for installation and recovery, but it is not the most convenient everyday terminal.

### Solution

Enable SSH inside Debian:

```bash
apt install -y openssh-server
systemctl enable --now ssh
systemctl status ssh --no-pager
```

From Windows PowerShell, connect with:

```powershell
ssh YOUR_USERNAME@192.168.8.4
```

Use `Ctrl+Shift+V` or right-click to paste into Windows Terminal or PowerShell.

### Verification

Confirm the PowerShell window displays a Debian shell prompt and that a harmless command such as `hostname` returns `jellyfin`.

### Lesson learned

Use the Proxmox console for initial installation and recovery. Use SSH for routine server administration and reliable command pasting.

## 7. The root SSH password was rejected

### Symptom

The command:

```powershell
ssh root@192.168.8.4
```

repeatedly returned `Permission denied` even when the known root password was entered.

### Cause

Debian's SSH configuration normally prevents password-based direct root login. The rejection did not necessarily mean the password itself was wrong.

### Solution

Find the normal account created during installation:

```bash
ls /home
```

Connect using that account:

```powershell
ssh YOUR_USERNAME@192.168.8.4
```

Then enter a root login shell only when administrative work is necessary:

```bash
su -
```

### Verification

Run:

```bash
whoami
```

Before `su -`, it should show the normal account. After `su -`, it should show `root`.

### Lesson learned

An authentication rejection can be caused by account policy rather than an incorrect password. Do not weaken SSH security before checking which account is permitted to log in.

## 8. The libraries were empty

### Symptom

Movies, TV Shows, and Music appeared in Jellyfin but contained no playable content.

### Cause

Jellyfin organizes and streams media supplied by the administrator. It does not include commercial movies or subscription-service content. Creating a library only tells Jellyfin which folder to scan.

### Solution

Use media that can legally be stored and streamed. For this lab, download Blender's openly licensed *Big Buck Bunny* into a correctly named folder:

```bash
mkdir -p "/srv/media/movies/Big Buck Bunny (2008)"
curl -L "https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_480p_h264.mov" \
  -o "/srv/media/movies/Big Buck Bunny (2008)/Big Buck Bunny (2008).mov"
```

Then select **Dashboard > Libraries > Scan All Libraries**.

### Verification

Confirm the Movies library displays *Big Buck Bunny* with artwork and metadata.

### Lesson learned

A media server is not a content subscription. Use personal media, openly licensed media, public-domain media whose status has been checked, or other files the administrator is legally permitted to store.

## 9. The movie was visible in the administrator page but could not be played

### Symptom

The Movies library tile displayed *Big Buck Bunny* artwork in **Dashboard > Libraries**, but there was no Play button.

### Cause

The administrator library page configures folders, metadata, and scans. It is not the normal browsing and playback interface.

### Solution

1. Select the back arrow to leave the administrator page.
2. Open **Movies** from the normal Jellyfin home screen.
3. Select *Big Buck Bunny*.
4. Select **Play**.

### Verification

The movie played successfully in the Jellyfin client.

### Lesson learned

Administration and playback are intentionally separated. When an item is detected but cannot be played from the current screen, first confirm whether the interface is an administrative view.

## What I Would Do Differently Next Time

1. Record the exact processor model before planning hardware acceleration.
2. Assign 8 GB RAM before running the current Jellyfin installer so `/tmp` clearly exceeds its capacity check.
3. Confirm SSH immediately after Debian installation and use it for the remaining commands.
4. Use the normal Debian account for SSH from the start rather than testing direct root login.
5. Write down the exact static-network configuration method while it is being performed.
6. Decide the permanent NAS and share protocol before adding a large media collection.
7. Verify and record `/srv/media` ownership with `stat` after applying permissions.
8. Create a backup and perform a restore test before treating the server as production-ready.
9. Keep a small openly licensed test file available for validating future library, storage, and transcoding changes.

## Compact troubleshooting reference

| Problem | Cause | Solution |
|---|---|---|
| Asterisks beside Debian tasks | Items were selected | Enter `11 12` to select SSH Server and standard utilities only |
| `sudo: command not found` | Minimal install lacked sudo; session was already root | Run commands directly as root and install `sudo` |
| `/tmp` below 2 GB free | Memory-backed `/tmp` was slightly undersized | Shut down and raise VM RAM from 4 GB to 8 GB |
| Unsure where to create media folders | `/tmp` was confused with persistent service data | Use `/srv/media/...` absolute paths |
| `cd *: too many arguments` | Wildcard expanded to several names | Use one exact path such as `cd /srv/media` |
| Cannot paste reliably in noVNC | Text console has no dependable shared clipboard | Connect from Windows PowerShell with SSH |
| Root SSH password rejected | Direct password-based root login was restricted | SSH as `YOUR_USERNAME`, then use `su -` |
| Empty libraries | No media files had been added | Add authorized media and scan the libraries |
| No Play button in Dashboard | Administrator page is not the playback view | Return to Home, open Movies, and select Play |

