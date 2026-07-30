# systemctl Beyond restart: The Service Management Habits Worth Building

Most people's relationship with `systemctl` starts and ends with three commands: `start`, `stop`, `restart`. That covers maybe sixty percent of what actually comes up day to day, and the other forty percent is exactly the part that matters when a service is misbehaving in a way that a restart doesn't fix, or when a restart fixes it temporarily and you need to understand why it'll break again in six hours.

## status Is More Informative Than People Give It Credit For

`systemctl status` gets treated as a yes/no check — is it running or not — when it's actually carrying a lot more information than that, if you read past the first two lines.

```bash
systemctl status my-app.service
```

The output tells you the process's actual PID, memory usage right now, how long it's been running since the last restart, and — critically — the last several lines of its journal output, right there without needing a separate command. If a service is crash-looping, that tail of recent log lines is often enough to spot the pattern immediately: same error, every eight seconds, which tells you it's crashing on startup rather than failing under load sometime later.

The "Active" line is worth reading carefully too. `active (running)` and `active (exited)` mean very different things — a one-shot service that's supposed to run once and exit will correctly show `exited`, and treating that as a failure because you expected `running` is a common source of false alarm.

## Reading Unit Files Instead of Guessing at Behavior

When a service isn't behaving the way you'd expect — not restarting after a crash, not waiting for a dependency, starting in the wrong order relative to something else — the unit file almost always explains why, and guessing is slower than just reading it.

```bash
systemctl cat my-app.service
```

`cat` here is a systemd subcommand, not the coreutils one — it shows you the fully resolved unit file, including any drop-in overrides layered on top of the base file, which matters because a lot of production systems configure services through drop-ins rather than editing the base unit directly.

```ini
[Service]
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3
```

That last pair is the one people miss most often. `StartLimitBurst=3` within `StartLimitIntervalSec=60` means systemd will only attempt 3 restarts in a 60-second window before giving up entirely and marking the service failed — which explains the specific and confusing situation where a service was crash-looping, and then just... stopped trying, sitting there in a failed state instead of continuing to retry. That's not a bug. That's the rate limit doing exactly what it was configured to do, and the fix is either `systemctl reset-failed` to clear the counter and try again, or actually addressing why it's crashing in the first place.

## Drop-In Overrides: Editing Behavior Without Editing the Original File

A habit worth building early: never edit a vendor-provided unit file directly, because a package update will silently overwrite your changes the next time it runs. The right way to customize behavior is a drop-in override, which systemd merges on top of the original automatically.

```bash
systemctl edit my-app.service
```

That command opens an editor for a drop-in file in `/etc/systemd/system/my-app.service.d/override.conf`, without you needing to remember the exact path. Anything you put there layers on top of the original unit, and survives package upgrades cleanly.

```ini
# /etc/systemd/system/my-app.service.d/override.conf
[Service]
Environment="LOG_LEVEL=debug"
MemoryMax=2G
```

After editing, systemd needs to know the unit files changed before it'll pick them up:

```bash
systemctl daemon-reload
systemctl restart my-app.service
```

Forgetting `daemon-reload` is a very common gotcha — you edit the file, restart the service, and nothing changes, because systemd is still working off its cached view of the unit from before your edit.

## journalctl: The Log Viewer That's Actually a Full Query Tool

If your service logs to the systemd journal rather than a flat file, `journalctl` is where you go, and it's a lot more capable than the default `journalctl -u my-app` most people stop at.

```bash
# Just this service, following live like tail -f
journalctl -u my-app.service -f

# Only since the last boot — useful after a host restart
journalctl -u my-app.service -b

# A specific time window, which is usually what you actually want during an incident
journalctl -u my-app.service --since "10 minutes ago"

# Only errors and above, filtering out the noise
journalctl -u my-app.service -p err

# Correlate multiple services in one chronological stream
journalctl -u my-app.service -u nginx.service --since "10 minutes ago"
```

That last one is underused and genuinely useful — when you suspect two services are interacting badly, seeing their logs interleaved chronologically, rather than in two separate terminal windows you're mentally syncing up by eye, makes the causal relationship a lot easier to spot.

## Dependency Ordering: After vs. Requires, Revisited

This is worth repeating because it's such a common source of confusing startup failures: `After=` in a unit file controls ordering only, not a hard dependency. A service can declare `After=postgresql.service` and still start up fine even if postgres never actually comes up — systemd just makes sure it *tries* postgres first, not that postgres succeeded.

```ini
[Unit]
After=postgresql.service
Requires=postgresql.service
```

You typically want both together: `After` for ordering, `Requires` for the actual hard dependency. Without `Requires`, you get services that start "successfully" against a database that isn't really there yet, and then spend their first several seconds failing connection attempts before retry logic catches up — which looks like a flaky app bug, when the real issue is a dependency declaration that was only ever half-written.

## enable vs. start: A Distinction That Costs People a Reboot

Last one, because it's such a common mix-up for anyone newer to systemd: `start` runs a service right now. `enable` configures it to start automatically on the next boot. They're independent of each other, and conflating them is how you end up with a service that's running fine today but mysteriously doesn't come back after a maintenance reboot.

```bash
systemctl start my-app.service    # running now, but only now
systemctl enable my-app.service   # will start on boot, but not right now

systemctl enable --now my-app.service   # both, in one command
```

That `--now` flag is the one worth defaulting to for anything you actually want running persistently — it removes the gap where you did one but forgot the other, and only found out during the next unplanned reboot at the worst possible time.

## Why All of This Matters More Than It Seems

None of this is glamorous. But systemd is the layer that decides whether your service comes back after a crash, after a reboot, after a dependency hiccup — and a lot of production reliability that looks like "the application is just solid" is actually "someone configured the unit file correctly." Understanding it well enough to read a unit file and predict its behavior, rather than trial-and-error restarting until something works, is one of those unglamorous skills that quietly saves hours over a career.
