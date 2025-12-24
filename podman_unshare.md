### Understanding the podman-unshare command.

Here are concrete, practical examples that show what podman unshare actually does and why you’d use it, step by step.

1. “Become root” — but only inside a user namespace
Command
```
podman unshare
```
What happens

- You get a shell ($SHELL) running in a new user namespace

- Inside that shell:
    - Your UID appears as 0 (root)
    - Your GID appears as 0

- Outside the shell, you are still an unprivileged user

Example

```
$ id
uid=1000(alice) gid=1000(alice)

$ podman unshare
# id
uid=0(root) gid=0(root)
```

👉 You look like root, but only for files and operations mapped to your user namespace.

2. Inspect Podman’s storage directories as “root”

Unprivileged Podman stores container data in directories your normal user cannot fully inspect.

Without unshare
```
ls ~/.local/share/containers/storage
# Permission denied
```
With unshare
```
podman unshare
# ls ~/.local/share/containers/storage
# ls overlay
# ls volumes
```

👉 You can now inspect, debug, or clean up container storage.

3. Manually delete broken container storage

If Podman storage becomes corrupted, normal users often can’t delete files.

Example
```
podman unshare
# rm -rf ~/.local/share/containers/storage/overlay/*
# exit
```

👉 This is one of the most common real-world uses of podman unshare.

4. Use podman mount as an unprivileged user

podman mount fails unless you are inside an unshare session.

Fails normally
```
podman mount mycontainer
# Error: not permitted
```
Works with unshare
```
podman unshare
# podman mount mycontainer
/home/alice/.local/share/containers/storage/overlay/.../merged
```

Now you can:
```
# cd $(podman mount mycontainer)
# ls
```

👉 This lets you browse and modify a container’s filesystem safely.

5. Run a single command instead of a shell

You don’t have to open a shell — you can run one command.

Example
```
podman unshare ls -l /var/lib/containers
```

Or:
```
podman unshare cat /proc/self/uid_map
```

Output shows how your UID is mapped inside the namespace.

6. Understand UID/GID mappings

Inside unshare, UID 0 maps to your real user ID on the host.

Example
```
podman unshare cat /proc/self/uid_map
```

Example output:
```

         0       1000          1
         1     100000      65536
```

Meaning:

UID 0 inside = UID 1000 outside (you)

Additional subuids come from /etc/subuid

7. Use environment variables set by unshare

Inside the session:
```
echo $CONTAINERS_GRAPHROOT
echo $CONTAINERS_RUNROOT
```

Example output:
```
/home/alice/.local/share/containers/storage
/run/user/1000/containers
```

👉 Useful for scripting cleanup or debugging.

8. Troubleshoot permission problems

If a container fails due to permission issues:
```
podman unshare
# touch ~/.local/share/containers/storage/testfile
# chown root:root testfile
```

This helps confirm whether the problem is UID mapping, not Podman itself.

Mental model (important)

Think of podman unshare as:

“Pretend I am root, but only for my own containers and files.”

It:

- ✅ Gives root-like powers within a sandbox

- ❌ Does not give real system root access

- ✅ Is safe for unprivileged users


To summurize:

podman unshare lets an unprivileged host user appear as root inside a user namespace, so they can act like root for Podman’s container files and mounts — without being real root on the host.

Important clarification

It does NOT make you root inside a running container.

Instead, it makes you:

- Root inside a user namespace on the host

- Root with respect to container storage, mounts, and files

- Still unprivileged on the actual host system

One-line mental model

“Fake root on the host, just enough to manage containers safely.”

Comparison
Context	                            Are you root?
Host OS (normal shell)	            ❌ No
Host OS (podman unshare)	        ✅ Root in namespace only
Running container	                ❌ No (unless the container itself runs as root)

Why this matters

This is why:

- You can delete container storage

- You can mount container filesystems

- You can debug permission issues

…but you cannot:

- Change system files

- Install system packages

- Affect other users

Your understanding is solid — this was just tightening the wording so it’s technically accurate.