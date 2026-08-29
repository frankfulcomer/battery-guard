# Battery Guard

Battery Guard is a lightweight Linux battery-monitoring service. It reads battery telemetry from UPower and can automatically power off the computer when the battery is discharging at or below a configured threshold.

## Safety

The repository defaults to dry-run mode:

```ini
THRESHOLD=20
DRY_RUN=true
```

In dry-run mode, Battery Guard reports when the shutdown condition is met but does not power off the computer. When `DRY_RUN=false`, a discharging battery at or below `THRESHOLD` causes Battery Guard to run:

```text
/usr/bin/systemctl poweroff
```

Test the complete dry-run workflow before enabling live mode.

## Requirements

- Linux with systemd
- UPower and the `upower` command
- Bash
- A battery exposed by UPower as `battery_BAT1`

Confirm the battery device on your system with:

```bash
upower -e
```

If the battery uses a different device name, update `BATTERY` near the top of `src/battery-guard`.

## Configuration

Edit `config/battery-guard.conf`:

```ini
THRESHOLD=20
DRY_RUN=true
```

- `THRESHOLD` must be a whole number from 1 through 100.
- `DRY_RUN` must be either `true` or `false`.

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
- the live low-battery branch invokes the simulated command.

Run it with:

```bash
./tests/test-battery-guard
```

Expected result:

```text
All Battery Guard tests passed.
```

## Install the systemd units

The included unit currently expects the repository at:

```text
/home/frank/Projects/battery-guard
```

If the repository is elsewhere, update `ExecStart` and `WorkingDirectory` in `systemd/battery-guard.service` first.

Install the units and reload systemd:

```bash
sudo install -m 0644 systemd/battery-guard.service /etc/systemd/system/battery-guard.service
sudo install -m 0644 systemd/battery-guard.timer /etc/systemd/system/battery-guard.timer
sudo systemctl daemon-reload
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

Immediately disable Battery Guard automation:

```bash
sudo systemctl disable --now battery-guard.timer
```

Disabling the timer prevents future automatic checks but does not remove the unit files or repository.
