# SSH Server 
Walkthrough of setting up and securing SSH Server.

## Table of Contents
- [Step 1 - Upgrading the Server](#step-1---upgrading-the-server)
- [Step 2 - Creating a new User](#step-2---creating-a-new-user)
- [Step 3 - Removing Root Login](#step-3---removing-root-login)
- [Step 4 - Recommended - SSH Keys](#step-4---ssh-keys)
- [Step 5 - Copying The Public Key to the Sever](#step-5---copying-the-public-key-to-the-server)
- [Step 6 - Disabling Password Authentication](#step-6---disabling-password-authentication)
- [Step 7 - Setting up UFW](#step-7---setting-up-ufw)
- [Step 8 - Setting up Fail2ban](#step-8---setting-up-fail2ban)
- [Resources/Credits](#resources)


### Step 1 - Upgrading the Server.
First thing we should do is update and upgrade. 

`sudo apt update`<br>
`sudo apt upgrade`<br>
`sudo apt autoremove`<br>

### Step 2 - Creating A New User.
Now we need to create a user so we can login via ssh. 

`sudo useradd -m newusername`<br>
`sudo passwd newusername`<br>

### Step 3 - Removing Root Login.
Next we should remove root login. 

`sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config`<br>
`sudo systemctl restart sshd`<br>

### Step 4 - Creating SSH Keys

While this step is optional it is highly recommended. If you skip this step, you must set-up fail2ban as per step 7. Here, the first goal is to create a key pair on the client machine, usually your computer.

`ssh-keygen`

Enter to save the key pair into the .ssh/ subdirectory in your home directory, or specify an alternate path. After you optionally may enter a secure passphrase, which is highly recommended. A passphrase adds an additional layer of security to prevent unauthorized users from logging in. 

### Step 5 - Copying the Public Key to the Server.
The quickest way to copy your public key to the server is to use a utility called ssh-copy-id. 

`ssh-copy-id username@remote_host`

If you don't have ssh-copy-id you can use cat and ssh to manually copy it.

`cat ~/.ssh/id_rsa.pub | ssh username@remote_host "mkdir -p ~/.ssh && touch ~/.ssh/authorized_keys && chmod -R go= ~/.ssh && cat >> ~/.ssh/authorized_keys`

### Step 6 - Disabling Password Authentication 
Now with a non-root user and ssh keys we can finally disable password authentication for our ssh server.

`sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config`<br>
`sudo systemctl restart ssh`

### Step 7 - Setting up UFW
Install UFW, by default there are no rules set. We should allow SSH to prevent locking ourselves out.
`sudo apt install ufw`
`sudo ufw allow ssh`

After confirming your have a rule to allow incoming SSH connections, you can enable the firewall with:
`sudo ufw enable`

### Step 8 - Setting up Fail2ban 
If you didn't follow step 4, the next step is a must to prevent brute force password attacks on your server.

`sudo apt install fail2ban`

With fail2ban installed we should now set up our jail before we turn the service on. There are two jails, on gets updated by fail2ban, and the other (/etc/fail2ban/jail.local) is the one we configure.

[Here is an example SSH-Fail2ban Jail](./resources/jail.local)<br>
With our jail in place we can now enable fail2ban as service so it will run on startup.<br>
`sudo systemctl enable fail2ban.service`<br>
`sudo systemctl start fail2ban.service`<br>
`sudo systemctl status fail2ban.service`<br>


### Resources
I found these resources helpful.<br>
[Digital Ocean - SSH Essentials](./resources/DigitalOcean_SSH_Essentials.md)<br>
[Digital Ocean - SSH Keys](./resources/DigitalOcean_SSH_Keys.md)<br>
[Github - SSH Agent](./resources/Github_SSH_Agent.md)



