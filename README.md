# alabs-mod-diagnostics-logs

Diagnostics report of the MOD Dwarf, ready to be pasted in a
forum post: device and image versions, services, audio, USB devices and whatever
other component installs a *provider* (e.g. the WiFi section of `alabs-mod-wifi`).

The report is **anonymized by default**: no passwords or configuration files
are ever included, network names, MAC addresses and the device serial number
are replaced by placeholders.

User documentation (the Diagnostics page of the web interface settings) is in
the [Diagnostics report](https://github.com/sejerpz/alabs-mod-custom-images/wiki/Diagnostics)
page of the aLabs custom images wiki.

## Layout

| Path | Installed as | Content |
|---|---|---|
| `diagnostics-logs` | `/usr/bin/diagnostics-logs` | the runner (Python 3.4) |
| `providers.d/` | `/usr/lib/diagnostics-logs/providers.d/` | the base providers: system, services, audio, usb |
| `VERSION` | `/usr/lib/diagnostics-logs/VERSION` | version shown in the report header |

## Usage

```
diagnostics-logs --list                       sections, as JSON
diagnostics-logs [--sections a,b] [--raw]     the report, as text
```

- `--list` prints `[{"id": "...", "title": "...", "description": "..."}, ...]`.
- `--sections` limits the report to the given section ids (default: all).
- `--raw` disables the anonymization.

The report starts with a header (image version, system and controller
revisions, version of this tool, date, uptime), followed by one section per
provider, as a markdown `## Title` and a code block:

````
# MOD Dwarf diagnostics

image:            v1.13.5.3315-stls17
system:           4dae598cd22da7ef9e49169837a4dada7d954b45
controller:       e99ddadb5b85e44bd3837b22b0b5df4eb93d71c1
diagnostics-logs: 0.1.0
date:             1970-01-01 00:39:55 (device clock)
uptime:           0d 00:39
anonymized:       yes

## USB

```
1-1      0bda:8179  Realtek 802.11n NIC  (USB 2.00, 480 Mbit/s)
         1-1:1.0 class ff: rtl8xxxu
```
````

## Web interface

`mod-ui` shows the report in the **Diagnostics** tab of the
settings page. It only calls this tool, so a new provider shows up in the
web interface without changing `mod-ui`:

| Request | Runs |
|---|---|
| `GET /system/diagnostics/list` | `diagnostics-logs --list`: one checkbox per section, `title` as the label and `description` as the tooltip |
| `GET /system/diagnostics/report?sections=a,b&anonymize=1` | `diagnostics-logs --sections a,b`, plus `--raw` when `anonymize=0` |

- The tab is hidden when `/usr/bin/diagnostics-logs` is not installed, or
  when there are no providers.
- `mod-ui` accepts only section ids made of `a-z`, `0-9`, `_` and `-`: a
  request with any other id fails (HTTP 400). Use only these characters in
  the `id` of a provider.
- The list is read when the settings page is opened: reload the page to see
  a provider just installed.

## Provider contract

A provider is any executable file in `providers.d/`; providers are run in
name order (`NN-name`), as root, with `LC_ALL=C`, a minimal `PATH` and a
**10 s timeout** each. A provider that times out or fails doesn't stop the
report: its section shows the error.

- `provider --describe` prints `key=value` lines: `id`, `title` and
  `description` of the section. Without it, the id is the file name without
  the `NN-` prefix.
- `provider` (no arguments) prints the section content on stdout, as plain
  text.
- A provider **must never print secrets**: passwords, keys, configuration
  files.
- A provider declares the sensitive strings it prints (e.g. network names) by
  appending them to the file named by `$ALABS_DIAG_REDACT`, one per line, as
  `<kind><TAB><value>`; the runner replaces every occurrence in the report
  with `<kind-N>` (e.g. `<ssid-1>`), the same placeholder for the same value.
  A line without a TAB is replaced with `<redacted-N>`.

The runner also anonymizes, in the whole report:

- MAC addresses: only the vendor part is kept (`d4:d6:df:xx:xx:xx`);
- the device serial number (`/var/cache/mod/tag`): `<serial>`.

Details of the anonymization:

- Values are replaced as **whole words** only: a value is not replaced when it
  is part of a longer word, i.e. when a letter, a digit, `_` or `-` is right
  before or after it. A network named `dwarf` becomes `<ssid-1>` in
  `ssid=dwarf`, but `moddwarf` and `dwarf-pedal` are left as they are.
- The values declared by all the providers are replaced in the whole report,
  not only in the section of the provider that declared them.
- Only the providers of the selected sections are run: a provider must declare
  every sensitive value it prints itself, without relying on another provider.

## Writing a provider

A provider is a script that prints some text. To add a section to the report:

1. **Choose the name**: `NN-name`, where `NN` gives the position in the
   report. The base providers use `10` to `40`; components installed on top
   of the image use `50` and above (e.g. `50-wifi` of `alabs-mod-wifi`).
2. **Implement `--describe`**: print `id`, `title` and `description`. Keep it
   fast and static: it is run every time the settings page is opened.
3. **Print the content** on stdout. Read the state, never change it.
4. **Declare the sensitive values** you print in `$ALABS_DIAG_REDACT`
   *before* or while printing them.
5. **Install it** in `/usr/lib/diagnostics-logs/providers.d/`, executable.
   Nothing has to be registered: the next `--list` or report picks it up.

### Shell template

```bash
#!/bin/bash
#
# diagnostics-logs provider: example (see the alabs-mod-diagnostics-logs project)

if [ "$1" = "--describe" ]; then
    echo "id=example"
    echo "title=Example"
    echo "description=State, users and last log lines of the example service"
    exit 0
fi

redact()
{
    [ -n "$2" ] && [ -n "$ALABS_DIAG_REDACT" ] && printf '%s\t%s\n' "$1" "$2" >> "$ALABS_DIAG_REDACT"
}

# sensitive values printed below: replaced by <user-1>, <user-2>, ...
users=$(sed -n 's/^user=//p' /data/example.conf 2>/dev/null | tr '\n' ' ')
for user in $users; do
    redact user "$user"
done

echo "service: $(systemctl is-active example.service 2>/dev/null)"
echo "users:   ${users:-none}"

# journalctl prints "-- Logs begin ..." / "-- No entries --": drop them
echo
echo "journal (last 20 lines):"
lines=$(journalctl -b --no-pager -n 20 -u example.service 2>/dev/null | grep -v '^-- ')
echo "${lines:-  none}"

exit 0
```

### Python template

The Dwarf has Python 3.4: no f-strings, no `subprocess.run`.

```python
#!/usr/bin/env python3
#
# diagnostics-logs provider: example (see the alabs-mod-diagnostics-logs project)

import os
import subprocess
import sys

if sys.argv[1:] == ['--describe']:
    print('id=example')
    print('title=Example')
    print('description=State and users of the example service')
    sys.exit(0)


def redact(kind, value):
    path = os.environ.get('ALABS_DIAG_REDACT')
    if path and value:
        with open(path, 'a') as fh:
            fh.write('%s\t%s\n' % (kind, value))


def output(args):
    """stdout of a command, also when it fails (e.g. systemctl is-active)."""
    try:
        out = subprocess.check_output(args, stderr=subprocess.DEVNULL, timeout=5)
    except subprocess.CalledProcessError as e:
        out = e.output
    except (OSError, subprocess.TimeoutExpired):
        return ''
    return out.decode('utf-8', 'replace').strip()


users = []
try:
    with open('/data/example.conf') as fh:
        users = [l.strip()[5:] for l in fh if l.startswith('user=')]
except (IOError, OSError):
    pass
for user in users:
    redact('user', user)

print('service: %s' % (output(['systemctl', 'is-active', 'example.service']) or 'unknown'))
print('users:   %s' % (', '.join(users) or 'none'))
```

### Guidelines

- **Read only**: a provider must never change the state of the device
  (no restarts, no writes outside `$ALABS_DIAG_REDACT`), the report can be
  generated at any time, also while playing.
- **Fast**: the whole section must be ready well within the 10 s timeout.
  When the timeout expires the provider and all its children are killed and
  the section shows what was printed so far.
- **Short**: the report is pasted in a forum post. Limit logs to the last
  lines (`tail -n`, `journalctl -n`) and filter them (`grep`).
- **No secrets**: never print configuration files that can contain
  passwords or keys, print the single values you need instead.
- **Exit code 0**: a non-zero exit code is shown in the section as
  `[provider failed with exit code N]`, followed by the last 5 lines of
  stderr. Use it for real errors only; a missing optional component is
  better reported as text (e.g. `not installed`).
- **Environment**: stdin is `/dev/null`, and the environment contains only
  `PATH=/usr/sbin:/usr/bin:/sbin:/bin`, `LC_ALL=C` and `ALABS_DIAG_REDACT`
  (no `HOME`, no `USER`). During `--describe`, `ALABS_DIAG_REDACT` is
  `/dev/null`.
- **Dwarf tools**: `bash`, busybox utilities (e.g. `lsusb` prints IDs only:
  read `/sys/bus/usb/devices/` for names and drivers), systemd 219
  (`journalctl` fails when `-t` and `-u` are combined and nothing matches:
  use one call for each).

### Testing

Run the provider alone, as root, on the Dwarf:

```
./50-example --describe
ALABS_DIAG_REDACT=/tmp/redact ./50-example; cat /tmp/redact
```

Then run it through the runner, from a test directory, to check the
anonymization:

```
mkdir -p /tmp/diag/providers.d
cp 50-example /tmp/diag/providers.d/
DIAGNOSTICS_LOGS_DIR=/tmp/diag diagnostics-logs --list
DIAGNOSTICS_LOGS_DIR=/tmp/diag diagnostics-logs --sections example
DIAGNOSTICS_LOGS_DIR=/tmp/diag diagnostics-logs --sections example --raw
```

The runner also works on a Linux PC (`python3 diagnostics-logs`), useful to
test providers that don't depend on the Dwarf.

### Shipping a provider with another component

A component keeps its provider in its own repository and installs it in
`/usr/lib/diagnostics-logs/providers.d/`, for example `alabs-mod-wifi`:

```
cp diagnostics/50-wifi /usr/lib/diagnostics-logs/providers.d/
```

The provider must work, and print something useful, also when the component
is not running or its hardware is not connected.

## Install

```
cp diagnostics-logs /usr/bin/
mkdir -p /usr/lib/diagnostics-logs/providers.d
cp providers.d/* /usr/lib/diagnostics-logs/providers.d/
cp VERSION /usr/lib/diagnostics-logs/
```

`DIAGNOSTICS_LOGS_DIR` overrides `/usr/lib/diagnostics-logs`, for testing.

## Licensing

MIT, see `LICENSE`: the runner `diagnostics-logs` and the providers in
`providers.d/` (`SPDX-License-Identifier` in each file).

A provider shipped by another component keeps the licence of that component:
the runner only executes it, as a separate program (e.g. `50-wifi` of
`alabs-mod-wifi` is GPL-2.0-only).
