# winvm-launcher

A single Bash script that turns a **Windows virtual machine on libvirt** into something that behaves like a normal desktop application on Linux.

Click the menu entry and the script makes sure the VM is running (starting or resuming it if needed), waits until Windows is really ready, then opens a **Remote Desktop (RDP)** session with clipboard, audio, a shared folder, your host keyboard layout and automatic resolution. When you are done and nobody is connected any more, the VM is **suspended** after a configurable idle time instead of being shut down.

It is designed for KDE Plasma and prefers **KRDC**, but works with **FreeRDP** too and installs a standard `.desktop` entry, so it also shows up in other desktop environments.

## Features

- **One script, three modes:** `install`, `configure`, `uninstall`.
- **Application menu entry** with an icon (the Windows 11 logo, embedded in the script as multi-size PNGs). Right-click actions: *Suspend VM now* and *Configure…*.
- **Smart start:** handles a VM that is running, paused, suspended by the guest (S3), shut off with a managed-save image, or still shutting down.
- **Real readiness detection** (no fixed sleeps): waits for the QEMU guest agent to answer, reads the VM's IPv4 address from the agent, then waits until the RDP port accepts connections. A progress dialog (kdialog) shows each step.
- **Idle suspend:** a background watcher suspends the VM when there are no established RDP connections to it for a configurable number of minutes. Two methods: `pause` (instant resume, RAM stays allocated) or `managedsave` (RAM saved to disk, survives host reboots).
- **Safe credentials:** the password is stored in the desktop keyring (KWallet / GNOME Keyring through the Secret Service API, using `secret-tool`). A `chmod 600` file is available only as an explicit, less safe fallback.
- **RDP with:**
  - clipboard and audio (optional microphone redirection),
  - a **shared folder** — your localized Documents folder (e.g. `Documenti`, `Dokumente`), another existing folder, or a new one,
  - your **host keyboard layout** (auto-detected, or chosen from a menu),
  - **automatic resolution** (follows the window size) and **HiDPI scaling** taken from the host,
  - windowed mode by default, fullscreen optional.
- **Clock sync:** sets the Windows clock from the host clock on every launch through the guest agent, which fixes the drift that appears after the VM has been paused.
- Works regardless of the system language (all `virsh` calls run with `LC_ALL=C`).

## Requirements

On the **host**:

| Needed | Notes |
|---|---|
| `bash` 4+, `python3` | used for JSON parsing |
| `virsh` (libvirt client) | your user must be allowed to manage the VM (usually the `libvirt` group for `qemu:///system`) |
| `secret-tool` | package `libsecret` or `libsecret-tools`; a running keyring (KWallet or GNOME Keyring) |
| An RDP client | **KRDC** (`krdc`), or **FreeRDP** (`xfreerdp`, `xfreerdp3`, `sdl-freerdp`, `sdl-freerdp3`, `wlfreerdp`) |
| `kdialog` + `qdbus` (optional) | progress dialog; falls back to desktop notifications |
| `kwriteconfig6`/`5`, `kreadconfig6`/`5` (optional) | KRDC per-host settings and KDE keyboard detection |
| `kscreen-doctor` (optional) | HiDPI scale detection on KDE |
| `notify-send` (optional) | notifications |

Inside the **Windows guest** (details in [Preparing the VM](#preparing-the-vm)):

1. The **guest agent channel** added to the VM's libvirt XML.
2. The **VirtIO guest tools** installed in Windows (this brings the QEMU guest agent and the VirtIO serial driver it talks through).
3. **Remote Desktop enabled**, and your user allowed to log in via RDP (member of *Remote Desktop Users* or an administrator).
4. The **keyboard layout** you send (e.g. Italian) installed in Windows.
5. A password-protected account (Windows does not accept RDP logins for accounts without a password by default).

## Preparing the VM

The script finds the VM's IP address and knows when Windows is ready by talking to the **QEMU guest agent**. That needs a small piece of configuration on the host and the guest tools inside Windows.

### 1. Add the guest agent channel (host)

Shut the VM down, then edit its definition:

```bash
virsh -c qemu:///system edit <vm-name>
```

Make sure this block exists inside `<devices>` (add it if it does not):

```xml
<channel type='unix'>
  <target type='virtio' name='org.qemu.guest_agent.0'/>
</channel>
```

libvirt fills in the socket path itself and adds the required `virtio-serial` controller automatically. Many VMs created with virt-manager already have this channel; check with:

```bash
virsh -c qemu:///system dumpxml <vm-name> | grep -A2 guest_agent
```

In virt-manager you can add the same thing with *Add Hardware → Channel*, name `org.qemu.guest_agent.0`, device type *UNIX socket*.

Optional but recommended for Windows: keep the clock mode that matches how Windows stores the hardware time (Windows normally uses local time):

```xml
<clock offset='localtime'/>
```

### 2. Install the VirtIO guest tools (Windows guest)

1. Download the **virtio-win** guest tools from the Fedora virtio-win project: <https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/> (use the `stable-virtio` or `latest-virtio` folder). Either get `virtio-win-guest-tools.exe`, or the `virtio-win.iso` and run `virtio-win-guest-tools.exe` from it.
2. Run the installer inside Windows, accept the defaults and finish. It installs the VirtIO drivers, the **VirtIO Serial Driver** and the **QEMU Guest Agent** service.
3. Reboot Windows (or at least start the VM again so the new channel is picked up).

Check inside Windows that:
- *Device Manager → System devices* lists **VirtIO Serial Driver**, and
- *Services* shows **QEMU Guest Agent** as *Running* (startup type *Automatic*).

### 3. Test from the host

```bash
virsh -c qemu:///system start <vm-name>
virsh -c qemu:///system qemu-agent-command <vm-name> '{"execute":"guest-ping"}'
virsh -c qemu:///system qemu-agent-command <vm-name> '{"execute":"guest-get-osinfo"}' --pretty
```

`{"return":{}}` from the first command means the agent works. If it says the agent is not connected, the channel is missing in the XML, the VirtIO Serial Driver is not installed, or the QEMU Guest Agent service is not running.

## Installation

```bash
git clone https://github.com/<your-user>/winvm-launcher.git
cd winvm-launcher
chmod +x winvm-launcher.sh
./winvm-launcher.sh install
```

Run it as a normal user, not as root. The wizard asks for:

1. libvirt connection URI (default `qemu:///system`) and the **VM name**
2. Windows **username and password** (stored in the keyring)
3. **Idle minutes** before suspending (`0` = never) and the suspend method
4. **Shared folder** (Documents / other / new / none)
5. **RDP client** (Auto prefers KRDC, otherwise FreeRDP)
6. **Keyboard layout** (auto-detect or pick from a list)
7. Windowed or fullscreen
8. Menu entry name
9. Optional advanced settings (clock sync, scaling, Unicode keyboard mode, microphone, timeouts)

The script copies itself to `~/.local/share/winvm-launcher/`, links it as `~/.local/bin/winvm-launcher`, installs the icons and creates the `.desktop` entry. Look for it in your application menu (default name: **Windows VM**).

## Usage

```text
winvm-launcher install      install and configure
winvm-launcher configure    change any setting (VM, credentials, idle time, ...)
winvm-launcher uninstall    remove everything this script installed
winvm-launcher launch       start/resume the VM and open RDP (what the menu entry runs)
winvm-launcher suspend      suspend the VM right now
winvm-launcher status       show configuration, VM state, detected layout and client
```

The `launch` command is what the menu entry runs. A second click while a session is starting or open does nothing except show a notification.

## Configuration

The configuration lives in `~/.config/winvm-launcher/config` (permissions `600`). You can edit it by hand; it is re-read on every launch. Re-run `winvm-launcher configure` afterwards if you changed `MENU_NAME`, so the menu entry is regenerated.

| Variable | Default | Meaning |
|---|---|---|
| `LIBVIRT_URI` | `qemu:///system` | libvirt connection |
| `VM_NAME` | – | domain name |
| `RDP_USER` | – | Windows user (`DOMAIN\user` is supported) |
| `CRED_BACKEND` | `secret-service` | `secret-service` or `file` |
| `IDLE_MINUTES` | `15` | idle time before suspend, `0` disables |
| `SUSPEND_MODE` | `pause` | `pause` or `managedsave` |
| `SHARE_DIR` / `SHARE_NAME` | – / `Share` | shared folder and the name shown in Windows (`\\tsclient\<name>`) |
| `RDP_CLIENT` | `auto` | `auto` (KRDC first), `krdc`, `freerdp`, or a full path |
| `FULLSCREEN` | `0` | `1` for fullscreen |
| `KBD_LAYOUT` | `auto` | `auto`, `none`, or a Windows layout id such as `0x410` |
| `KBD_UNICODE` | `0` | `1` sends typed characters instead of scancodes (FreeRDP) |
| `SCALE` | `auto` | `auto` (from the host) or a percentage such as `100`, `140`, `180` |
| `MICROPHONE` | `1` | redirect the microphone (FreeRDP) |
| `TIME_SYNC` | `always` | `always`, `resume` (only after pause/restore) or `off` |
| `TIME_SYNC_AUTOFIX` | `1` | if clock sync needs it, start the Windows Time service through the guest agent |
| `READY_TIMEOUT` | `300` | seconds to wait for the agent / RDP port |
| `RDP_PORT` | `3389` | RDP port in the guest |
| `PASSWORD_MODE` | `stdin` | FreeRDP: pass the password via stdin; `arg` uses `/p:` (visible in `ps`) |
| `KRDC_PASSWORD_IN_URL` | `0` | KRDC: `1` puts the password in the `rdp://` URL (visible in `ps`) |
| `MENU_NAME` | `Windows VM` | name of the menu entry |

## How it works

1. **Lock** — a lock file prevents two launches at once; any running idle watcher is cancelled.
2. **VM state** — `virsh domstate` decides between `resume`, `dompmwakeup` or `start` (a managed-save image is restored automatically on `start`).
3. **Readiness** — polls `guest-ping` through the QEMU guest agent, then reads `guest-network-get-interfaces` (ignoring loopback and link-local addresses) and waits for the RDP port.
4. **Clock** — sends the host time to the guest with `guest-set-time`.
5. **RDP client** — starts KRDC or FreeRDP with the stored credentials and the options above.
6. **Idle watcher** — once the client is closed, a detached process checks every 20 seconds for established TCP connections to the VM's RDP port. When there are none for `IDLE_MINUTES`, it pauses or managed-saves the VM. Opening a new session cancels it.

## Host shutdown and reboot

**The script does nothing when the host shuts down or reboots.** It only suspends the VM from its idle watcher, which is a normal user process. What happens to the VM when the PC goes down is decided by libvirt, so set that up once.

| Situation at host shutdown | Result |
|---|---|
| `libvirt-guests` is **not** enabled | QEMU is killed like a power cut. A VM suspended with `pause` loses its RAM state and the next launch is a cold boot from a possibly unclean disk. |
| `libvirt-guests` is enabled with `ON_SHUTDOWN=suspend` | Running VMs get a managed save to disk and are restored by the next `winvm-launcher launch`. |
| `libvirt-guests` is enabled with `ON_SHUTDOWN=shutdown` | Windows is shut down cleanly (takes longer; set `SHUTDOWN_TIMEOUT`). |
| VM already suspended with `SUSPEND_MODE=managedsave` | Its state is already on disk, so it survives a reboot with no extra setup. |

Recommended setup for `qemu:///system`:

```bash
sudo systemctl enable --now libvirt-guests
```

and in its config file (`/etc/libvirt/libvirt-guests.conf`, or `/etc/default/libvirt-guests` / `/etc/sysconfig/libvirt-guests` on some distributions):

```ini
URIS="qemu:///system"
ON_SHUTDOWN=suspend
ON_BOOT=ignore
```

`ON_SHUTDOWN=suspend` makes libvirt do a managed save of running guests. `ON_BOOT=ignore` keeps the VM from starting at boot, so it only comes up when you launch it from the menu. When the launcher finds a VM that is shut off with a managed-save image, it starts it (libvirt restores the saved state) and syncs the clock afterwards.

If you want the safest behaviour regardless of how libvirt-guests treats a VM that is paused at the moment of shutdown, use `SUSPEND_MODE=managedsave` in the launcher configuration: the saved state is then already on disk before the host goes down.

## Client notes

### KRDC

KRDC has no command-line options for clipboard, audio, shared folder, keyboard layout or resolution. The script therefore writes them into KRDC's per-host settings (`~/.config/krdcrc`, group `hostpreferences`) before each launch. The keys and values follow KRDC's `RdpHostPreferences` (keyboard layout index, sound, resolution mode, shared directory).

- This part is **best effort**: if audio, keyboard or shared folder do not show up as expected, open KRDC's settings for that host once and check them. Please open an issue with your KRDC version.
- KRDC cannot read a password from stdin. By default it asks once (tick the option to remember it in KWallet). You can opt in to passing the stored password in the URL, but it is then visible in the process list while the client starts.

### FreeRDP

Full control through command-line options: clipboard, audio, microphone, drive redirection, keyboard layout, dynamic resolution and DPI scaling. FreeRDP 2 and 3 are both supported (the option syntax differs and is handled by the script). The password is passed through stdin so it does not appear in `ps`; if your build does not support that, set `PASSWORD_MODE=arg`.

Certificates are accepted without prompting (`/cert:ignore`), which is reasonable for a VM on a local libvirt network but worth knowing.

## Troubleshooting

Everything is logged to `~/.local/state/winvm-launcher/launcher.log`. `winvm-launcher status` shows the effective configuration.

| Symptom | Check |
|---|---|
| "Cannot query VM" | `virsh -c qemu:///system list --all` as your user; are you in the `libvirt` group? |
| Waits forever for the agent | Follow [Preparing the VM](#preparing-the-vm): channel in the XML, VirtIO guest tools installed, QEMU Guest Agent service running. Test with `guest-ping`. |
| Agent answers but RDP never opens | Is Remote Desktop enabled in Windows? Firewall on the network profile in use? |
| Wrong keyboard layout | `winvm-launcher status` shows the detected layout and its source. Pick the layout in `configure`, make sure it is installed in Windows, or try `KBD_UNICODE=1` with FreeRDP. |
| Clock not synced | `grep -i clock` in the log. Manual test: `virsh qemu-agent-command <vm> '{"execute":"guest-set-time","arguments":{"time":'$(date +%s%N)'}}'` |
| Clock sync error about the "Windows Time service" | Happens with `virsh domtime --sync` when W32Time is stopped (common after debloating). The script uses an explicit time instead and can start the service itself (`TIME_SYNC_AUTOFIX`). |
| Menu entry missing or stale | Run `kbuildsycoca6 --noincremental` (or `kbuildsycoca5`) or log out and in. |

## Security notes

- The password goes into the Secret Service keyring. The `file` backend stores it in plain text with `600` permissions and is only offered when no keyring is available.
- FreeRDP receives the password via stdin; KRDC by default asks for it itself.
- RDP certificates are not verified.
- With `TIME_SYNC_AUTOFIX=1` the script may run `sc config w32time start= auto & net start w32time` in the guest through the agent. Set it to `0` if you do not want that.

## Uninstall

```bash
winvm-launcher uninstall
```

It removes the menu entry, icons and program files, and optionally the configuration, logs and stored password. The virtual machine itself is never touched; if it is paused at that moment you are offered to resume it.

## Files

| Path | Purpose |
|---|---|
| `~/.local/share/winvm-launcher/winvm-launcher` | installed script |
| `~/.local/bin/winvm-launcher` | symlink to it |
| `~/.local/share/applications/winvm-launcher.desktop` | menu entry |
| `~/.local/share/icons/hicolor/*/apps/winvm-launcher.png` | icons |
| `~/.config/winvm-launcher/config` | configuration |
| `~/.local/state/winvm-launcher/launcher.log` | log |

## Contributing

Issues and pull requests are welcome. When reporting a problem, please include your distribution, desktop environment and session type (X11/Wayland), KRDC or FreeRDP version, the output of `winvm-launcher status`, and the relevant lines of the log (the log never contains the password).

## License

This project is licensed under the **GNU General Public License v3.0**. See the [LICENSE](LICENSE) file for the full text.

The Windows logo used for the menu icon is a trademark of Microsoft and is not covered by this license.
