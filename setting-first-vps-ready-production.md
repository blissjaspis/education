# Setting Up Your First Ubuntu Server for Production

This guide walks you through the essential steps to configure a new Ubuntu server, making it secure and ready for a production environment.

## 1. Initial Server Access

How you first connect to your server depends on how it was provisioned by your hosting provider. Choose the method that matches your setup.

### Method A: Logging in with a Root Password

This method is common if your provider gave you a password for the `root` user. First, connect to your server. You'll need its IP address.

```bash
ssh root@YOUR_SERVER_IP
```

You will be prompted for the root password.

### Method B: Logging in with an SSH Key

This is a modern and more secure method where you provide your public SSH key to the hosting provider during server creation. In this case, password login is often disabled by default.

Your provider will tell you the default username (it could be `root`, `ubuntu`, `ec2-user`, etc.).

```bash
# For root access
ssh root@YOUR_SERVER_IP

# Or for a default user on providers like AWS/GCP
ssh ubuntu@YOUR_SERVER_IP
```
If you use this method, you can skip to step 2, as your initial SSH connection is already secure.

## 2. Create a New User

Once you are logged in, the first step is to create a new user account that you will use for everyday administration. Running commands as `root` is risky.

Replace `username` with your desired username.

```bash
adduser username
```

You'll be asked to create a password and provide some information for the new user.

### Grant Administrative Privileges

Add the new user to the `sudo` group. This allows the user to run commands with administrative privileges by typing `sudo` before the command.

```bash
usermod -aG sudo username
```

## 3. Configure SSH Access for Your New User

The final goal is to be able to log in directly to the server as your new user with an SSH key. The method depends on how you first logged in.

### If You Logged in with a Password (Method A)

If you logged in as `root` using a password, you need to copy your local public SSH key to your new user's account on the server. This is the key that lives on your personal computer.

The easiest way to do this is with the `ssh-copy-id` command from your local machine.

```bash
# Run this command on your local computer
ssh-copy-id username@YOUR_SERVER_IP
```

You will be prompted for the new user's password that you created in step 2. After this, you can log in as the new user without a password.

### If You Logged in with an SSH Key (Method B)

If you already logged in as `root` (or `ubuntu`) using an SSH key, you can simply copy the authorized key from the root/default user to your new user. This grants your new user access using the same SSH key.

**Perform these commands on your server while logged in as root.**

```bash
# Create the .ssh directory for the new user
mkdir -p /home/username/.ssh

# Copy the authorized keys file
cp ~/.ssh/authorized_keys /home/username/.ssh/authorized_keys

# Set the correct ownership and permissions
chown -R username:username /home/username/.ssh
chmod 700 /home/username/.ssh
chmod 600 /home/username/.ssh/authorized_keys
```

**Important:** After completing either of the steps above, open a **new** terminal window on your local machine and test that you can log in directly as your new user before proceeding.

```bash
ssh username@YOUR_SERVER_IP
```
You should connect successfully without a password.

## 4. Secure SSH

Now that you can log in as your non-root user with SSH keys, it's time to harden the SSH service. This involves preventing the `root` user from logging in over SSH and disabling password-based authentication entirely.

**Perform these steps while logged in as your new user (e.g., `ssh username@YOUR_SERVER_IP`).**

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
sudo systemctl restart ssh
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
