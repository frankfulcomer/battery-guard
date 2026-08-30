# System Guard

System Guard is a set of lightweight Linux health-monitoring services:

- **Low Battery Guard** reads UPower telemetry, warns before the battery is
  exhausted, and can power off safely at a critical charge level.
- **Thermal Guard** watches every Linux thermal zone, warns when the hottest
  sensor is too warm, and can power off safely at a critical temperature.
- **Storage Guard** watches a configured filesystem and sends escalating
  capacity alerts. It never removes files automatically.

## Safety

Any guard capable of powering off the computer supports a dry-run mode. Use
this safe setting while validating a new installation:

```ini
WARNING_THRESHOLD=20
THRESHOLD=15
DRY_RUN=true
WARNING_USER=frank
```

In dry-run mode, Low Battery Guard reports when the shutdown condition is met but does not power off the computer. When `DRY_RUN=false`, a discharging battery at or below `THRESHOLD` causes Low Battery Guard to run:

```text
/usr/bin/systemctl poweroff
```

Test the complete dry-run workflow before enabling live mode.

## Requirements

- Linux with systemd
- UPower and the `upower` command
- Bash
- `notify-send` for desktop notifications
- `canberra-gtk-play` for audible alerts
- A battery exposed by UPower as `battery_BAT1`
- Linux thermal-zone sensors exposed below `/sys/class/thermal`
- GNU `df` (from coreutils) for filesystem-capacity readings
- `smartctl` from `smartmontools` for physical disk health monitoring

Confirm the battery device on your system with:

```bash
upower -e
```

If the battery uses a different device name, update `BATTERY` near the top of `src/battery-guard`.

## Configuration

Edit `config/battery-guard.conf`:

```ini
WARNING_THRESHOLD=20
THRESHOLD=15
DRY_RUN=true
WARNING_USER=frank
```

- `WARNING_THRESHOLD` is the percentage at which Low Battery Guard displays a
  desktop notification and plays an alert sound while discharging.
- `THRESHOLD` must be a whole number from 1 through 100.
- `DRY_RUN` must be either `true` or `false`.
- `WARNING_USER` is the signed-in desktop user who receives the notification
  and sound.

`WARNING_THRESHOLD` must be higher than `THRESHOLD`. Low Battery Guard sends one
warning during each discharge cycle rather than repeating it every minute. The
warning resets when the battery stops discharging.

Keep `DRY_RUN=true` until the script and systemd service have been tested successfully on the target computer.

## Run manually

From the repository root:

```bash
./src/battery-guard
```

A normal safe result resembles:

```text
State: pending-charge
Percentage: 79%
Battery condition is safe.
```

## Tests

The test suite supplies simulated UPower readings and a harmless replacement for the power-off command. It verifies that:

- dry-run mode never invokes power-off;
- a safe battery condition never invokes power-off; and
- the live low-battery branch invokes the simulated command;
- the visual and audible warning path runs at the configured warning level;
- a warning is sent only once during a discharge cycle; and
- charging resets the warning for the next discharge cycle.

Run it with:

```bash
./tests/test-battery-guard
./tests/test-thermal-guard
./tests/test-storage-guard
./tests/test-uptime-guard
./tests/test-history
./tests/test-email-report
```

Expected result:

```text
All Low Battery Guard tests passed.
All Thermal Guard tests passed.
All Storage Guard tests passed.
All Uptime Guard tests passed.
All structured history tests passed.
All weekly email report tests passed.
```

The thermal tests use simulated sysfs sensor readings and a harmless fake
power-off command. The storage tests use simulated filesystem readings and
verify safe, warning, critical, deduplication, and reset behavior.

## Thermal Guard

Configure `config/thermal-guard.conf`:

```ini
WARNING_TEMP_C=85
CRITICAL_TEMP_C=95
DRY_RUN=true
WARNING_USER=frank
```

Temperatures are whole degrees Celsius from 1 through 150, and the critical
threshold must be higher than the warning threshold. Thermal Guard examines all
`thermal_zone*/temp` sensors and acts on the hottest valid reading. It sends one
alert per warning level while the temperature remains elevated, resetting after
temperatures fall below the warning level. At the critical level it either
reports the dry-run action or requests a clean system power-off.

Run a check manually from the repository root:

```bash
./src/thermal-guard
```

## Storage Guard

Configure `config/storage-guard.conf`:

```ini
MOUNT_POINT=/
WARNING_USAGE_PERCENT=85
CRITICAL_USAGE_PERCENT=95
WARNING_USER=frank
HEALTH_MONITORING=true
HEALTH_DEVICE=auto
MAX_DEVICE_TEMP_C=70
MAX_PERCENTAGE_USED=90
```

Usage thresholds are whole percentages from 1 through 100, and the critical
threshold must be higher than the warning threshold. Storage Guard sends one
warning at the first threshold and a new critical alert if usage rises further.
The alert state resets after usage falls below the warning level. Automatic
deletion is intentionally outside this guard's responsibilities.

When `HEALTH_MONITORING=true`, Storage Guard resolves the physical disk behind
`MOUNT_POINT` and reads its SMART or NVMe health report. It checks:

- the drive's overall health result and NVMe critical-warning flags;
- device temperature;
- SSD life percentage used;
- NVMe media and data-integrity errors; and
- ATA reallocated, pending, reported-uncorrectable, and offline-uncorrectable
  sector counts.

`HEALTH_DEVICE=auto` works for ordinary disks, partitions, and device-mapper
stacks. Set an explicit path such as `/dev/sda` for RAID, USB bridges, or other
layouts that cannot be resolved automatically. Unsupported devices and missing
SMART data are reported as unavailable without interrupting capacity checks.
Health inspection is read-only and never starts a self-test or repair.

On Ubuntu, install the health dependency with:

```bash
sudo apt install smartmontools
```

Run a check manually from the repository root:

```bash
./src/storage-guard
```

## Structured history and trends

Each successful telemetry check appends one row to a daily CSV file below:

```text
/var/lib/system-guard/history/
├── battery/YYYY-MM-DD.csv
├── thermal/YYYY-MM-DD.csv
├── storage/YYYY-MM-DD.csv
└── uptime/YYYY-MM-DD.csv
```

Battery history includes state, percentage, and guard status. Thermal history
includes the hottest zone, its millidegree reading, and guard status. Storage
history includes capacity, free space, device health, temperature, SSD wear,
and error counts when the drive exposes them.

Uptime Guard samples the kernel's monotonic uptime and boot ID every five
minutes. The trend report uses those samples to show observed online time,
estimated offline time, availability, reboot transitions, and current uptime.
Because shutdown can occur between samples, each reboot boundary has up to
approximately five minutes of uncertainty.

History is enabled in each component configuration with:

```ini
HISTORY_ENABLED=true
HISTORY_RETENTION_DAYS=90
```

Daily files older than the retention period are removed automatically. File
locking prevents overlapping timer runs from corrupting a history file, and a
history-write failure is logged without interrupting any protection action.

Show a seven-day trend summary:

```bash
./src/system-guard-report
```

Choose a different reporting window by supplying the number of days:

```bash
./src/system-guard-report 30
```

The CSV files remain directly usable by spreadsheet software, plotting tools,
or a future dashboard.

## Weekly email reports

System Guard can email the trend summary once per week through Gmail. The
hourly timer checks the configured local weekday and hour, while a state marker
ensures that only one message is sent in each ISO week.

Install the mail client:

```bash
sudo apt install msmtp msmtp-mta
```

Copy the configuration templates outside the repository:

```bash
sudo install -d -m 0755 /etc/system-guard
sudo install -m 0644 config/email-report.conf.example /etc/system-guard/email-report.conf
sudo install -m 0600 config/msmtprc.example /etc/msmtprc
```

Edit `/etc/system-guard/email-report.conf` to set the recipient, sender,
weekday, local hour, reporting window, subject, and msmtp account. For Sunday
at 9 AM with a seven-day report, use:

```ini
EMAIL_ENABLED=true
EMAIL_TO=YOUR_GMAIL_ADDRESS
EMAIL_FROM=YOUR_GMAIL_ADDRESS
REPORT_DAY=Sunday
REPORT_HOUR=9
REPORT_DAYS=7
EMAIL_SUBJECT="System Guard weekly report"
SMTP_ACCOUNT=gmail
```

Edit `/etc/msmtprc` with the same Gmail address and a Gmail App Password. Keep
this file at mode `0600`; it contains a credential and must never be committed.

Install and start the email units:

```bash
sudo install -m 0644 systemd/system-guard-email-report.service /etc/systemd/system/
sudo install -m 0644 systemd/system-guard-email-report.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now system-guard-email-report.timer
```

Send a safe immediate test without waiting for Sunday by temporarily invoking
the service script with its force switch:

```bash
cd /home/frank/Projects/system-guard
sudo SYSTEM_GUARD_REPORT_FORCE=true ./src/system-guard-email-report
```

View delivery logs with:

```bash
sudo journalctl -u system-guard-email-report.service -n 50 --no-pager
```

## Install the systemd units

The included unit currently expects the repository at:

```text
/home/frank/Projects/system-guard
```

If the repository is elsewhere, update `ExecStart` and `WorkingDirectory` in `systemd/battery-guard.service` first.

Install the units and reload systemd:

```bash
sudo install -m 0644 systemd/battery-guard.service /etc/systemd/system/battery-guard.service
sudo install -m 0644 systemd/battery-guard.timer /etc/systemd/system/battery-guard.timer
sudo install -m 0644 systemd/thermal-guard.service /etc/systemd/system/thermal-guard.service
sudo install -m 0644 systemd/thermal-guard.timer /etc/systemd/system/thermal-guard.timer
sudo install -m 0644 systemd/storage-guard.service /etc/systemd/system/storage-guard.service
sudo install -m 0644 systemd/storage-guard.timer /etc/systemd/system/storage-guard.timer
sudo install -m 0644 systemd/uptime-guard.service /etc/systemd/system/uptime-guard.service
sudo install -m 0644 systemd/uptime-guard.timer /etc/systemd/system/uptime-guard.timer
sudo systemctl daemon-reload
```

Each component has its own timer, so it can be tested and enabled independently:

```bash
sudo systemctl start thermal-guard.service storage-guard.service uptime-guard.service
sudo systemctl enable --now thermal-guard.timer storage-guard.timer uptime-guard.timer
```

Test the service while `DRY_RUN=true`:

```bash
sudo systemctl start battery-guard.service
sudo systemctl status battery-guard.service --no-pager
```

The service uses `Type=oneshot`, so `inactive (dead)` is normal after a successful run. Confirm that the status contains `status=0/SUCCESS` and the expected battery reading.

Enable periodic dry-run monitoring:

```bash
sudo systemctl enable --now battery-guard.timer
systemctl status battery-guard.timer --no-pager
```

A healthy timer shows `active (waiting)` and a future trigger time.

## Enable live protection

Live mode can power off the computer. Preserve open work before enabling it.

First stop automation:

```bash
sudo systemctl disable --now battery-guard.timer
systemctl is-active battery-guard.timer
systemctl is-enabled battery-guard.timer
```

Confirm the results are `inactive` and `disabled`. Then set:

```ini
DRY_RUN=false
```

Run one manual check while the battery is safely above the threshold:

```bash
sudo systemctl start battery-guard.service
sudo systemctl status battery-guard.service --no-pager
```

If the result is correct, enable live periodic protection:

```bash
sudo systemctl enable --now battery-guard.timer
systemctl status battery-guard.timer --no-pager
```

With the example configuration, Low Battery Guard shows a pop-up and plays a sound
at 20% while discharging. If discharge continues, it powers off the computer at
15% or below.

## Monitoring and troubleshooting

View recent service logs:

```bash
sudo journalctl -u battery-guard.service -n 50 --no-pager
```

Check the timer and its next scheduled run:

```bash
systemctl status battery-guard.timer --no-pager
systemctl list-timers battery-guard.timer --no-pager
```

Immediately disable Low Battery Guard automation:

```bash
sudo systemctl disable --now battery-guard.timer
```

Disabling the timer prevents future automatic checks but does not remove the unit files or repository.
