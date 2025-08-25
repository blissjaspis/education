# Setting Up Your First Ubuntu Server for Production

This guide walks you through the essential steps to configure a new Ubuntu server, making it secure and ready for a production environment.

## 1. Log in as Root

First, connect to your server as the `root` user. You'll need your server's IP address.

```bash
ssh root@YOUR_SERVER_IP
```

You will be prompted for the root password provided by your hosting provider.

## 2. Create a New User

Running commands as `root` is risky. It's best practice to create a non-root user with administrative privileges.

Replace `username` with your desired username.

```bash
adduser username
```

You'll be asked to create a password and provide some information for the new user.

### Grant Administrative Privileges

Add the new user to the `sudo` group. This allows the user to run commands with `sudo`.

```bash
usermod -aG sudo username
```

## 3. Set Up SSH Key-Based Authentication

Using SSH keys is more secure than using passwords.

### Generate SSH Keys on Your Local Machine

If you don't have SSH keys yet, generate them on your local computer.

```bash
ssh-keygen
```

Press Enter to accept the default file location and options. You can optionally set a passphrase for extra security.

### Copy Your Public Key to the Server

The easiest way to copy your public key is with `ssh-copy-id`.

```bash
ssh-copy-id username@YOUR_SERVER_IP
```

If `ssh-copy-id` is not available, you can do it manually. First, display your public key:

```bash
cat ~/.ssh/id_rsa.pub
```

Copy the output. Now, log in to your server as the new user and create the `.ssh` directory and `authorized_keys` file.

```bash
# Log in as the new user in a new terminal window
ssh username@YOUR_SERVER_IP

# On the server
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste your public key into the `authorized_keys` file, then save and exit (`Ctrl+X`, `Y`, `Enter`).

Set the correct permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Now, you should be able to log in as the new user without a password.

## 4. Secure SSH

Now that you can log in with SSH keys, you should disable password-based authentication and root login over SSH.

### Edit the SSH Configuration File

Open the SSH daemon configuration file with `nano`.

```bash
sudo nano /etc/ssh/sshd_config
```

Find these lines and change them as follows:

```
PermitRootLogin no
PasswordAuthentication no
```

Save and close the file.

### Restart SSH Service

To apply the changes, restart the SSH service.

```bash
sudo systemctl restart sshd
```

**Important:** Before logging out, open a new terminal window and test that you can still log in as your new user.

```bash
ssh username@YOUR_SERVER_IP
```

If it works, your SSH setup is secure.

## 5. Enable the Firewall (UFW)

UFW (Uncomplicated Firewall) is an easy-to-use firewall for Ubuntu.

### Allow SSH Connections

First, and most importantly, you must allow SSH connections so you don't get locked out of your server.

The standard SSH port is `22`. You can allow it by name using `OpenSSH` or by port number. Using the port number is more explicit.

```bash
# Allow by service name (common)
sudo ufw allow OpenSSH

# Or, allow by port number (more explicit)
sudo ufw allow 22/tcp
```

If you configured SSH to run on a different port, you must allow that specific port number.

### Enable the Firewall

**⚠️ Warning:** Before you enable the firewall, double-check that you have allowed your SSH port. If the SSH port is not allowed, **you will be locked out of your server** the moment you enable the firewall. It is also wise to have a direct console or VNC access method available (provided by your hosting provider) as a backup in case you get locked out.

Once you are certain you have allowed SSH traffic, you can enable UFW.

```bash
sudo ufw enable
```

Press `y` to confirm.

### Check Firewall Status

You can check the status at any time.

```bash
sudo ufw status
```

If you plan to run a web server, you should also allow HTTP and HTTPS traffic:

```bash
sudo ufw allow 'Nginx Full' # or 'Apache Full'
# Or just by port
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

## 6. Install Fail2ban

Fail2ban protects your server from brute-force attacks by monitoring logs for suspicious activity and banning IPs that show malicious signs.

```bash
sudo apt update
sudo apt install fail2ban -y
```

Fail2ban will start automatically and provide a good default configuration.

## 7. Keep Your Server Updated

It's crucial to keep your system's packages up-to-date.

```bash
sudo apt update && sudo apt upgrade -y
```

You might want to set up automatic updates for security patches.

## Conclusion

Your Ubuntu server is now set up with a solid security baseline. You have a non-root user, a firewall, and secure SSH access. You can now proceed with installing and configuring your application stack.
