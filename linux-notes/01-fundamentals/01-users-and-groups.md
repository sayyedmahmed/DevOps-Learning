# Users & Groups

Here are your concise notes:

---

## 1. Users & Groups Basics

### Commands
```
useradd -m -s /bin/bash username    # Create user with home dir and bash shell
groupadd groupname                  # Create a group
usermod -aG groupname username      # Add user to a group (-a = append, -G = supplementary)
visudo                              # Safely edit sudo permissions
id username                         # Check user's UID, GID, groups
su - username                       # Switch to user with full login environment
```

### Key Concepts
- **`-m`** → Creates home directory (`/home/username`)
- **`-s /bin/bash`** → Sets shell to Bash (scripts work, history, tab completion, `.bashrc` loads)
- **`-s /sbin/nologin`** → Blocks interactive login, user only runs services
- **`-aG`** → Always use together; `-G` alone removes user from other groups

### Grant Sudo
- Add user to `wheel` group: `usermod -aG wheel username`
- Or via `visudo`: `username ALL=(ALL:ALL) ALL`

---

## 2. Three User Types

| User | Shell | Sudo | Purpose |
|------|-------|------|---------|
| `adminuser` | `/bin/bash` | Yes (`wheel` group) | Human admin |
| `guestuser` | `/bin/bash` | No | Limited human user |
| `serviceuser` | `/sbin/nologin` | No | Run services, no login |

### Creation
```bash
sudo useradd -m -s /bin/bash adminuser
sudo useradd -m -s /bin/bash guestuser
sudo useradd -m -s /sbin/nologin serviceuser
sudo usermod -aG wheel adminuser
```

### Why nologin Still Has a Home Directory
- Services need `$HOME` for config, cache, SSH keys, temp files
- Even though no human logs in, the **process** lives there

### nologin Safety
- Hacker who compromises a service gets no shell
- Can't install tools, can't move around, limited damage
- Combine with: no password, no SSH access, minimal file permissions
