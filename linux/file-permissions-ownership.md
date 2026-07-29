# chmod, chown, and the Permission Errors That Waste Everyone's Afternoon

Permission errors have a specific flavor of frustrating, because the fix is almost always a single command, but figuring out *which* single command requires actually understanding what the numbers and letters mean instead of pattern-matching a StackOverflow answer from three years ago. Most engineers know `chmod 777` makes an error go away. Fewer know why that's usually the wrong move, or what the right one actually is.

## The Three Numbers Aren't Arbitrary

Every `chmod` command with a number like `755` or `644` is really three separate numbers glued together — owner, group, everyone else — and each one is a sum of read (4), write (2), and execute (1). Once that clicks, you stop memorizing common combinations and start just computing them.

```bash
# 755: owner can read/write/execute, everyone else can read/execute
chmod 755 deploy.sh

# 644: owner can read/write, everyone else can only read
chmod 644 config.yaml

# 700: owner only, nobody else gets anything — common for private keys
chmod 700 ~/.ssh/id_rsa
```

That last one matters more than it looks. SSH will flatly refuse to use a private key with overly permissive permissions, and the error message it gives — `UNPROTECTED PRIVATE KEY FILE` — is one of the more genuinely helpful ones in the Unix world, because it tells you exactly what's wrong instead of just failing silently.

## Why 777 Is Almost Never the Answer

`chmod 777` gives read, write, and execute to literally everyone on the system, and it works as a fix in the sense that the permission error goes away. What it doesn't do is fix the actual problem, which is usually that a process is running as the wrong user, or a directory has the wrong group ownership for what's trying to write into it. Reaching for 777 is treating the symptom and leaving the underlying misconfiguration in place, which tends to resurface later as a much worse problem — a world-writable file in a shared environment is one bad actor away from being a real security incident, not just an annoyance.

The better instinct, almost every time, is to ask "who actually needs access here, and why don't they have it yet?" rather than "how do I make this error stop."

## chown and the Group Ownership People Forget

`chmod` controls what permissions exist. `chown` controls who they apply to, and it's the half of the equation that gets forgotten more often, especially the group half.

```bash
# Change owner only
chown deploy app.log

# Change owner and group in one shot
chown deploy:deploy app.log

# Change group without touching owner
chgrp deploy app.log
```

A pattern that comes up constantly in shared environments: a directory where multiple services need to write, but you don't want every user on the box to have access. The fix isn't loosening permissions for everyone — it's creating a shared group, adding the right users to it, and setting the group bit so new files inherit that group automatically.

```bash
groupadd shared-logs
usermod -aG shared-logs deploy
usermod -aG shared-logs monitoring

# The setgid bit means new files in this directory inherit the group,
# instead of defaulting to whatever user's primary group happens to be
chmod g+s /var/log/shared
chown :shared-logs /var/log/shared
```

That `g+s` bit is one of those things that solves a specific, recurring annoyance — files created by different users ending up with different groups, breaking access for everyone else — and almost nobody knows it exists until they've hit the problem it solves.

## umask: The Setting Nobody Configures Until It Bites Them

Every file you create gets a default permission, and that default is controlled by `umask`, which is one of those settings that quietly runs in the background until a script creates a file with permissions you didn't expect and something downstream chokes on it.

```bash
umask
# 0022 is a common default — it subtracts write access for group and other

umask 0027
# Now newly created files won't be readable by "other" at all
```

The reason this matters in automation specifically: a deploy script that creates config files inheriting an overly permissive umask from whatever shell or cron context it's running in can quietly leave secrets world-readable, without a single line of the script doing anything obviously wrong. Setting an explicit umask at the top of sensitive scripts, rather than trusting whatever the environment happens to default to, is cheap insurance.

## Debugging "Permission Denied" Like a Detective, Not a Guesser

The instinct when you hit a permission error is to immediately chmod something. The better instinct is to actually look at what's happening first, because there are at least four different layers a permission problem could be coming from, and guessing wastes time.

```bash
# What are the actual permissions, and who owns it?
ls -la /var/log/app/

# Which user is the process actually running as?
ps -eo user,pid,cmd | grep app

# Is this an ACL problem hiding behind normal-looking permissions?
getfacl /var/log/app/output.log

# Is SELinux or AppArmor silently blocking this, regardless of Unix permissions?
sudo ausearch -m avc -ts recent    # SELinux
sudo aa-status                     # AppArmor
```

That last check trips people up constantly, especially on RHEL-based systems where SELinux is enforcing by default. You can have completely correct Unix permissions — right owner, right mode, everything textbook — and still get denied because SELinux's security context doesn't allow it. The error message doesn't always make this obvious, so if a permission fix that should have worked didn't, checking `ausearch` or `aa-status` before spending another twenty minutes on chmod is worth the ten seconds it costs.

## ACLs for When Standard Permissions Aren't Expressive Enough

Standard Unix permissions only give you three buckets: owner, group, everyone. Sometimes that's not enough — you need one specific additional user to have write access without changing group ownership for everyone else. That's what ACLs exist for, and they're underused mostly because people don't know they're an option.

```bash
# Give a specific user read/write, without touching the file's actual group
setfacl -m u:contractor:rw /var/log/app/output.log

# Check what ACLs are actually set
getfacl /var/log/app/output.log
```

It's a small tool, reached for rarely, but exactly the right one for that one annoying case where the standard three-bucket model doesn't map cleanly onto who actually needs access.

## The Underlying Pattern

Permission problems are almost never actually about permissions — they're about ownership, group membership, or a security layer like SELinux operating underneath the Unix permission model you're used to thinking in. The fastest path to fixing them is resisting the urge to chmod first and instead spending thirty seconds figuring out which of those layers is actually the one saying no. That thirty seconds, spent consistently, adds up to a lot less 777 scattered across production over the course of a career.
