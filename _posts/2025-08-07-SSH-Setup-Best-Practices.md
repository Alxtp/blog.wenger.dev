---
title: "SSH Authentication Best Practices"
date: 2025-08-07 21:00:00 +0200 
categories: [Network]
tags: [ssh]
---

When it comes to securely connecting to your SSH server or using SSH to authenticate to Git, there are several important things to consider. This guide covers generating SSH keys, configuring them on remote servers, disabling password authentication, and using SSH with Git.

## Generate SSH Key Pair

To generate a key pair, run the following command:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/key_name -C "comment"
```
- Give a useful key name and avoid using the default name
- Create a password (don't leave it blank)
- Use `ed25519` as the key type (more modern and secure than RSA acording to: [https://security.stackexchange.com/questions/90077/ssh-key-ed25519-vs-rsa](https://security.stackexchange.com/questions/90077/ssh-key-ed25519-vs-rsa)
- Include a useful comment (usually your email address)

This command generates a private key (`key_name`) and a public key (`key_name.pub`) in the `~/.ssh/`directory.

## Configure SSH Keys on Remote Server
To get the public key onto the remote server (which should have SSH server installed), use this command:

```bash
ssh-copy-id -i ~/.ssh/key_name.pub username@hostname
```

This copies the public key into the `~/.ssh/authorized_keys` file on the remote server.

Now, instead of using password authentication to connect to the server with `ssh username@hostname` you can connect with the key:

```bash
ssh -i ~/.ssh/key_name username@hostname
```

## Disable Password Authentication
After verifying that the key-based authentication works, you can disable password authentication on the remote server to improve security.

Edit the "/etc/ssh/sshd_config" file and change the line `#PasswordAuthentication yes` to `PasswordAuthentication no`.
Then restart the SSH service:

```bash
sudo systemctl restart ssh
```

To test this, exit from the SSH session and try to connect with a password:

```bash
ssh username@hostname # should not work

ssh -i ~/.ssh/key_name username@hostname # should work
```

## Using ssh authentication for git 

When using SSH for authentication with Git, the key can be generated the same way, and then the public key needs to be uploaded to the Git solution like GitHub. Here's the set up for GitHub:

1. Generate the SSH key pair as described above
2. Upload the public key to GitHub

The issue with using custom-named keys is that Git commands like `git clone git@github.com:username/repo_name.git` can't specify which key to use, causing SSH to try default key names. This can be seen when running:

```bash
ssh -vT git@github.com
...
debug1: Authentications that can continue: publickey
debug1: Next authentication method: publickey
debug1: Will attempt key: /home/username/.ssh/id_rsa
debug1: Will attempt key: /home/username/.ssh/id_ecdsa
debug1: Will attempt key: /home/username/.ssh/id_ecdsa_sk
debug1: Will attempt key: /home/username/.ssh/id_ed25519
debug1: Will attempt key: /home/username/.ssh/id_ed25519_sk
debug1: Will attempt key: /home/username/.ssh/id_xmss
debug1: Trying private key: /home/username/.ssh/id_rsa
debug1: Trying private key: /home/username/.ssh/id_ecdsa
debug1: Trying private key: /home/username/.ssh/id_ecdsa_sk
debug1: Trying private key: /home/username/.ssh/id_ed25519
debug1: Trying private key: /home/username/.ssh/id_ed25519_sk
debug1: Trying private key: /home/username/.ssh/id_xmss
debug1: No more authentication methods to try.
git@github.com: Permission denied (publickey).
```

The solution is to create an SSH config file to specify which key to use for GitHub:

```
Host github.com
    HostName github.com
    IdentityFile ~/.ssh/github
    IdentitiesOnly yes
```
{: file="~./.ssh/config" }

This configuration will make sure that the key named `github` is used when connecting to github.com.
Now the connection works:

```
ssh git@github.com
Enter passphrase for key '/home/username/.ssh/github':
PTY allocation request failed on channel 0
Hi Username! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

The same option can be used when connecting to remote server to omit the `-i` argument on the `ssh` command.
