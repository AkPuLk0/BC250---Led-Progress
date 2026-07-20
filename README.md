# BC250 --- Led-Progress
Animated LED progress bar for Steam downloads and KDE file transfers on SteamOS using a Corsair Commander Duo.


# Steam Download RGB — SteamOS + OpenLinkHub

## Version v0.7.1 — Dolphin file copies

This version fully preserves the Steam progress tracking validated in v0.7 and
adds monitoring for file copies started from Dolphin. KDE publishes these jobs
through `org.kde.JobViewServer`, including their percentage, copied bytes, total
size, and completion message.

- Steam download: blue progress, with priority.
- Dolphin copy: purple progress.
- Multiple copies: overall progress weighted by their size.
- When all jobs finish or are cancelled: automatic return to
  `gpu-temperature` after the configured delay.
- A copy continues to be monitored even if Steam/CEF is temporarily closed.

Dolphin monitoring uses `dbus-monitor`, which is already available on SteamOS,
and adds no extra Python dependency.

## v0.7 fix — smooth visual progress

The next LED is now brightness-modulated according to the fractional progress.
The fill no longer appears to advance only in large steps. On a channel exposed
as a single LED, brightness follows the percentage directly. On the eight-LED
channel, fully lit LEDs still advance by segments, while the leading LED changes
at every percentage point.

## v0.6 fix

RGB output now uses a single global write to the COMMANDER DUO.
OpenLinkHub's OpenRGB server accepts this method, whereas zone-by-zone writes
could leave the LEDs turned off. Progress is still calculated separately for
each channel, then the colors are concatenated before the global write.

## v0.5 fix

This version prioritizes real-time progress from `DownloadOverview` when its
AppID matches the active download. On SteamOS, `update_type_info` can remain
stuck at 0 or 1%, while `overview.overall_percent_complete` and
`overview.progress` continue to update correctly every second.

## Safety measures

- The diagnostic and first test modes are read-only.
- Fans and pumps are never modified.
- Current RGB profiles are read and stored at startup.
- On normal exit, `Ctrl+C`, network error, or `SIGTERM`, the program disables
  OpenRGB integration and reapplies the stored profiles.
- The systemd service is not enabled automatically by the installer.

## 1. Enable the OpenRGB server in OpenLinkHub

The main OpenLinkHub configuration file must contain:

```json
{
  "enableOpenRGBTargetServer": true,
  "openRGBPort": 6743
}
```

The location depends on your installation. To search for it without making any
changes:

```bash
grep -R '"enableOpenRGBTargetServer"\|"openRGBPort"' \
  ~/OpenLinkHub ~/.config/OpenLinkHub 2>/dev/null
```

After editing the file, restart the OpenLinkHub service using the command that
matches your installation, then verify the port:

```bash
ss -ltn | grep ':6743'
```

The script will enable and disable the COMMANDER DUO's **OpenRGB integration**
by itself during downloads. Do not leave it enabled manually for the initial
test sequence.

## 2. Install

Download the latest archive from the GitHub repository, then run:

```bash
unzip steam-download-rgb-*.zip
cd steam-download-rgb
chmod +x install.sh uninstall.sh
./install.sh
```

Everything is installed inside your home directory. Disabling SteamOS read-only
mode is not required.

## 3. Run the diagnostic without modifying the LEDs

```bash
~/.local/share/steam-download-rgb/.venv/bin/python \
  ~/.local/share/steam-download-rgb/steam_download_rgb.py --diagnose
```

Expected output:

```text
[OK] Steam CEF / SharedJSContext
[OK] OpenLinkHub: COMMANDER DUO
[OK] OpenRGB server: 127.0.0.1:6743
[OK] Zones: ... LEDs
```

## 4. Check Steam and Dolphin monitoring only

Run the command below, then start a Steam download:

```bash
~/.local/share/steam-download-rgb/.venv/bin/python \
  ~/.local/share/steam-download-rgb/steam_download_rgb.py --monitor-only
```

The terminal should display something similar to:

```text
Steam: 37.4% | 12.2 GiB / 32.6 GiB | 54.1 MiB/s | 6 min 18 s remaining
Dolphin copy: 62.0% | 107.9 MiB / 172.5 MiB
```

Stop with `Ctrl+C`.

Emergency restore command, available at any time:

```bash
~/.local/share/steam-download-rgb/.venv/bin/python \
  ~/.local/share/steam-download-rgb/steam_download_rgb.py --restore
```

## 5. Progressive LED test

This command only runs a 0 → 25 → 50 → 75 → 100% test, then automatically
restores the OpenLinkHub profiles:

```bash
~/.local/share/steam-download-rgb/.venv/bin/python \
  ~/.local/share/steam-download-rgb/steam_download_rgb.py --led-test
```

## 6. Real-world test

```bash
~/.local/share/steam-download-rgb/.venv/bin/python \
  ~/.local/share/steam-download-rgb/steam_download_rgb.py
```

Expected behavior:

1. No activity: OpenLinkHub keeps `gpu-temperature`.
2. A Dolphin copy starts: the bar fills in purple.
3. A Steam download starts: it takes priority and the bar turns blue.
4. When Steam finishes, any still-active copy resumes in purple.
5. Two seconds after all activity ends, OpenRGB integration is disabled and
   OpenLinkHub restores the stored temperature profiles.

## 7. Enable automatic startup only after validation

```bash
systemctl --user enable --now steam-download-rgb.service
```

View the logs:

```bash
journalctl --user -u steam-download-rgb.service -f
```

Stop and disable:

```bash
systemctl --user disable --now steam-download-rgb.service
```

## Configuration

File:

```text
~/.config/steam-download-rgb/config.json
```

Useful options:

- `"zones": [0, 1]`: OpenRGB channels to control.
- `"filled_color": [28, 125, 255]`: blue progress color.
- `"copy_filled_color": [180, 60, 255]`: purple Dolphin copy color.
- `"empty_color": [0, 0, 0]`: unfilled LEDs.
- `"reverse": true`: reverses the fill direction.
- `"idle_restore_delay_seconds": 2.0`: delay before returning to temperature
  profiles.
- `"monitor_only": true`: prevents any LED writes, even without the CLI option.
- `"monitor_dolphin_copies": false`: disables Dolphin copy monitoring only.

## Known limitation

Steam's internal JavaScript APIs are not a stable public API.
The script dynamically searches for `SharedJSContext` on every reconnection and
uses both available sources (`DownloadItems` and `DownloadOverview`) for improved
robustness, but a future Steam update may require an adaptation.

## Checks performed before release

- Python compilation check (`compileall`).
- Bash script syntax validation (`bash -n`).
- systemd service file verification.
- Simulated tests for download detection, progress calculation, zone filling,
  and profile restoration after errors.
- Verification of the OpenLinkHub
  `/api/color/setOpenRgbIntegration` endpoint in the official source code.

No software can be guaranteed to be completely bug-free. The first use should
therefore remain progressive: diagnostic, read-only monitoring, LED test, then
a real-world test.

## AI-assisted development

The code in this project was generated entirely with the assistance of
ChatGPT by OpenAI, based on descriptions, real-world tests, and feedback
collected on SteamOS.

The different features were then tested and corrected progressively until they
worked with:

- Steam CEF;
- KDE JobViewServer;
- OpenLinkHub;
- OpenRGB;
- the Corsair Commander Duo.

This note is included for transparency for anyone who wants to know how the
project was created.

## License

This project is distributed under the MIT License.
