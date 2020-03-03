---
layout: "post"
title: "Setting up git running under WSL"
description: ''
category: null
tags: []
---

I recently ran into a problem running git from inside a WSL (Windows Subsystem Linux) command prompt to manage a git repository stored in a folder outside the WSL file system. In my case that location was C:\projects but I believe the problem would exist for any location outside the WSL directory.

The error message looked like this when I tried to `git clone`:
```bash
error: chmod on /mnt/c/projects/<repo>/.git/config.lock failed: Operation not permitted
fatal: could not set 'core.filemode' to 'false'
```

I could overcome the issue by using `sudo` but other git operations would fail when I tried to use the local repository. The fix ([detailed here on askubuntu](https://askubuntu.com/questions/1115564/wsl-ubuntu-distro-how-to-solve-operation-not-permitted-on-cloning-repository/1118158#answer-1118158)) was to remount the C:\ drive with the metadata flag enabled to allow `chmod`/`chown` commands to work on the C:\ partition.

```bash
sudo umount /mnt/c
sudo mount -t drvfs C: /mnt/c -o metadata
```

[More details are in the Microsoft release notes](https://docs.microsoft.com/en-us/windows/wsl/release-notes#build-17063) for setting default file permissions in the WSL partition and help configuring other mount options.
