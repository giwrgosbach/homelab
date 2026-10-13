# SSH Hardening

**Goal:** fred only accepts SSH logins with keys. No passwords.

## Setup

| Machine | Role | Details |
|---------|------|---------|
| **bach** | my main workstation (client) | where the private key lives |
| **fred** | server | static IP on my LAN, login user `<user>` |

In the commands below, `<fred-ip>` is fred's address and `<user>` is the account I log in with. Swap in your own.

> fred needs a fixed address (static IP or DHCP reservation). The IP ends up in commands and in `~/.ssh/config`, so if it changes, everything breaks. I set mine with nmcli, see the lab log for 2026-09-27.

## Why keys and not passwords

With a password, I'm actually sending the secret to fred every time. Anyone who gets it (or guesses it) can log in as me.

With keys:
- bach has a **private key** that never leaves bach.
- fred has the matching **public key**.
- bach proves it owns the private key without ever sending it.

It works the other way too. fred has its own key pair, the **host key**. The first time I connected, SSH showed fred's fingerprint and asked if I trust it. After "yes", bach saved it in `~/.ssh/known_hosts` and now checks it quietly every time.

> If you ever get `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`, don't just delete the line in `known_hosts`. Either the server was reinstalled or something is pretending to be it. Figure out which one first.

## Where everything lives

| Machine | File | What it is | Mode |
|---------|------|---------|------|
| bach | `~/.ssh/id_ed25519_fred` | private key | `600` |
| bach | `~/.ssh/id_ed25519_fred.pub` | public key (fine to share) | `644` |
| bach | `~/.ssh/known_hosts` | fingerprints of servers I trust | `644` |
| bach | `~/.ssh/config` | shortcuts like `ssh fred` | `600` |
| fred | `~/.ssh/authorized_keys` | public keys allowed to log in as `<user>` | `600` |
| fred | `/etc/ssh/sshd_config.d/10-hardening.conf` | my server rules: no passwords, no root | `644` |

---

## 1. Make the key (on bach)

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_fred -C "bach-to-fred"
```

- `-t ed25519`: modern key type. Small, fast, strong. Basically the default now.
- `-a 100`: 100 rounds of hashing on the passphrase, so brute forcing a stolen key file is slow.
- `-f`: file name. One key per purpose keeps things clear.
- `-C`: comment, so I can tell which key is which in `authorized_keys`.

I set a passphrase. It encrypts the private key, so a stolen copy is useless without it. With `AddKeysToAgent` (step 5) I only type it once per session anyway.

## 2. Copy the public key to fred (on bach)

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_fred.pub <user>@<fred-ip>
```

This is the last time I log in with the password. `ssh-copy-id` appends the public key to `~/.ssh/authorized_keys` on fred, and creates the file/folder with the right permissions if they're missing.

## 3. Check permissions (on fred)

```bash
stat -c '%A %a %U:%G %n' ~ ~/.ssh ~/.ssh/authorized_keys   # want 700, 700, 600, owned by me
ls -Z ~/.ssh/authorized_keys                              # SELinux type should be ssh_home_t
cat ~/.ssh/authorized_keys                                # one line ending in "bach-to-fred"
```

This matters because sshd silently ignores `authorized_keys` if the home dir, `~/.ssh` or the file itself is writable by group/others, or if the SELinux label is wrong. From what I read, that's the #1 reason a key "doesn't work".

## 4. Test the key on its own (on bach)

```bash
ssh -v -i ~/.ssh/id_ed25519_fred \
    -o IdentitiesOnly=yes \
    -o PasswordAuthentication=no \
    -o KbdInteractiveAuthentication=no \
    <user>@<fred-ip>
```

Passwords are turned off on the client side here, so the only way in is the key. In the verbose output I looked for:

```
Authenticated to <fred-ip> ([<fred-ip>]:22) using "publickey".
```

Basically: make sure the new door works before locking the old one.

## 5. Add a shortcut (on bach)

```bash
cat >> ~/.ssh/config <<'EOF'
Host fred
    HostName <fred-ip>
    User <user>
    IdentityFile ~/.ssh/id_ed25519_fred
    IdentitiesOnly yes
    AddKeysToAgent yes
EOF

chmod 600 ~/.ssh/config
ssh -G fred | grep -E '^(user|hostname|identityfile) '    # see what ssh will actually use
ssh fred
```

Now `ssh fred` is enough. Ansible reads this file too, so later the inventory can just say `fred`.

## 6. Turn off password logins (on fred)

Up to here I only added a new way in (the key). Passwords still work, so anyone on the network can still try guessing. This step closes that.

sshd reads its rules from config files. I drop in one small file that says "no passwords", check it, and reload sshd. After that fred only lets in keys from `authorized_keys`.

> **Keep two SSH sessions open to fred.** Leave one alone as a lifeline and work in the other. Don't close the lifeline until step 7 passes.

The rules:

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
```

- `PasswordAuthentication no`: no typed passwords.
- `KbdInteractiveAuthentication no`: closes the other route that can still prompt for a password.
- `PermitRootLogin no`: no logging in directly as root.

**Why `sudo tee`:** `sudo echo ... > file` doesn't work, because the `>` redirect runs as my user, not root. `tee` runs under sudo so it can write the file.

**Why `10-`:** sshd keeps the *first* value it reads for each setting. On Fedora, `sshd_config` includes `sshd_config.d/*.conf` at the top and reads them in name order. `10-` comes before Fedora's own `50-redhat.conf`, so nothing later can quietly override my settings.

Check syntax, then apply:

```bash
sudo sshd -t                  # no output = config is fine
sudo systemctl reload sshd    # apply (existing sessions stay open)
```

## 7. Verify

On fred, check what sshd is really using:

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

Should print:

```
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
```

On bach, in a **new** terminal:

```bash
ssh fred                      # should log in with the key
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password fred
                              # should fail: Permission denied (publickey).
```

Both did what they should, so I closed the lifeline. Done.

## Rollback (if locked out)

From the lifeline session on fred:

```bash
sudo rm /etc/ssh/sshd_config.d/10-hardening.conf
sudo systemctl reload sshd
```

If there's no session left, go to fred's local console (keyboard + monitor) and run the same thing.
