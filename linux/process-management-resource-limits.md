# top, kill, and ulimit: Managing Processes Under Pressure

There's a particular kind of pressure that comes with a host running hot — load average climbing, something eating memory, and a growing sense that whatever you're about to do next needs to be right the first time. This is about the tools for that moment: figuring out what's actually consuming resources, deciding what to do about it, and understanding the limits that were quietly shaping the outcome the whole time.

## top and htop Aren't Just for Watching Numbers Scroll

Everyone's opened `top` at some point and stared at the screen without really reading it. The columns that matter most during an actual incident are usually `%CPU`, `RES` (resident memory, the actual physical RAM in use, as opposed to virtual memory which can be misleadingly large), and `S` (process state).

```bash
top -o %MEM
```

Sorting by memory up front, rather than the default CPU sort, matters because memory pressure and CPU pressure look completely different in how urgently they need attention. A process pinned at 100% CPU is annoying but usually not about to take the whole host down. A process steadily climbing in memory is often minutes away from triggering the OOM killer, which doesn't ask politely — it picks a victim based on its own scoring heuristic, and that victim isn't always the process that was actually the problem.

`htop`, where it's installed, adds a few things worth knowing about beyond the nicer UI — specifically the ability to see a process tree (`F5` in most builds) and to filter (`F4`), both of which matter when you're trying to find one specific runaway process among hundreds on a busy host.

## Process States: The Letter Next to the PID That People Skip Past

That `S` column — process state — is one of the more informative things on the screen, and one of the most ignored. `R` is running. `S` is sleeping, which is normal and fine. `D` is uninterruptible sleep, usually waiting on disk I/O, and a process stuck in `D` state for a long time is a strong signal of an actual I/O problem, not an application bug — and critically, a process in `D` state **cannot be killed**, not even with `kill -9`, until whatever I/O it's waiting on resolves. That's a genuinely useful thing to know before you spend ten minutes confused about why `kill -9` isn't working.

```bash
ps -eo pid,stat,cmd | grep " D"
```

`Z` is zombie — a process that's already finished but whose exit status hasn't been collected by its parent yet. A handful of zombies briefly appearing and disappearing is completely normal. A large and growing number of them sticking around is usually a sign that the parent process isn't calling `wait()` properly, which circles back to the same container-init problem covered elsewhere in this series — an application acting as PID 1 without properly reaping its children.

## kill Sends Signals, Not Just Death

`kill` is one of the most misunderstood commands in daily use, mostly because people only ever reach for `kill -9` and never learn what the other signals actually do. `kill` doesn't inherently mean "terminate" — it means "send this process a signal," and what the process does with that signal is up to it.

```bash
kill -15 12345    # SIGTERM: "please shut down gracefully" — the default, and the one to try first
kill -9 12345      # SIGKILL: immediate, unconditional termination — the process gets no say
kill -1 12345      # SIGHUP: traditionally "reload your config," honored by many long-running services
```

The difference between `-15` and `-9` matters a lot more than people treat it as mattering. SIGTERM gives a well-behaved process the chance to close database connections cleanly, flush buffers, finish an in-flight request, and exit in an orderly way. SIGKILL gives it none of that — the kernel just removes it, mid-whatever-it-was-doing, and if it was mid-write to a file or holding a lock, that lock or that half-written file is now someone else's problem. `kill -9` should be the second thing you try, after SIGTERM has had a few seconds to work and clearly hasn't, not the reflexive first move.

```bash
# The pattern worth defaulting to
kill -15 12345
sleep 5
if kill -0 12345 2>/dev/null; then
  echo "Still alive, escalating"
  kill -9 12345
fi
```

That `kill -0` trick is worth knowing on its own — sending signal 0 doesn't actually send anything, it just checks whether the process exists and you have permission to signal it, which makes it a clean way to test liveness without side effects.

## nice and Priority: Sharing CPU Without Starving Anyone

On a host running a mix of latency-sensitive services and background batch work, CPU priority is worth actually thinking about rather than leaving everything at the same default, especially if a backup job or a batch analytics process shares a host with something user-facing.

```bash
# Start a process with lower priority (higher nice value = lower priority)
nice -n 10 ./backup_job.sh

# Change the priority of something already running
renice -n 10 -p 12345
```

Nice values run from -20 (highest priority) to 19 (lowest), and the default is 0. Setting a backup or batch job to a positive nice value means it yields CPU to anything more time-sensitive without needing to be paused or scheduled around manually — the kernel just naturally deprioritizes it under contention, and it speeds back up the moment the host isn't busy.

## ulimit: The Ceiling Nobody Notices Until They Hit It

Every process operates under a set of resource limits — open file descriptors, max processes, memory — and when one of those limits gets hit, the failure often looks nothing like "you hit a limit." It looks like a mysterious `EMFILE` error, or a fork that silently fails, or a service that just stops accepting new connections without an obvious reason in its own logs.

```bash
# What limits does the current shell operate under?
ulimit -a

# The two people hit most often in practice
ulimit -n    # max open file descriptors
ulimit -u    # max user processes
```

The gotcha worth knowing: a limit set with `ulimit` in an interactive shell doesn't apply to services started by systemd or launched from cron, because those don't inherit your shell's environment. For a systemd-managed service, the limit needs to be set in the unit file itself.

```ini
[Service]
LimitNOFILE=65536
LimitNPROC=4096
```

This exact mismatch — someone testing a fix by raising `ulimit -n` in their SSH session, confirming it works, and then being confused when the actual systemd service still hits the old limit in production — is common enough that it's worth flagging explicitly rather than assuming it's obvious.

## Bringing It Together

Under real pressure — a host that's genuinely struggling — the sequence that tends to work is: `top` sorted by memory to find the likely culprit, a check of its process state to make sure it's not stuck in uninterruptible sleep, SIGTERM before SIGKILL to give it a chance to exit cleanly, and a mental note to check whether a ulimit or a nice value was quietly shaping the whole situation before you even got paged. None of these tools are new or exotic. What makes the difference is reaching for the right one first, instead of the blunt one you remember best.
