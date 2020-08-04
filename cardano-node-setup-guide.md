# Setup cardano-node Stakepool (Ubuntu 20.04)

This guide is for educational purposes only. Do not use in production with real funds or do it at your own risk. By using this guide, you assume sole risk and waive any claims of liability against the author.  

-- Note: This guide is for running cardano-node on a virtual private server (VPS), running Ubuntu 20.04.  
-- Note: This guide assumes your local machine is a Mac, but most instructions are executed on the remote machine.  
-- Note: anything preceded by "#" is a comment.   
-- Note: anything all-caps in between "<>" is an placeholder; e.g. `"<FILENAME>"` could be `"foo.txt"`.   
-- Note: anything in between "${}" is a variable that will be evaluated by your shell.  

* Author: **lunarpool.io** (stake-pools: **MOON1** | **MOON2** | **MOON9**)  

* Thanks to these expert contributors!  
Input Output HK - [Cardano Tutorials](https://docs.cardano.org/projects/cardano-node/en/latest/getting-started/install.html)  
Chris Graffagnino - **MASTR** ([Jörmungandr node setup guide](https://github.com/Chris-Graffagnino/Jormungandr-for-Newbs/blob/master/docs/jormungandr_node_setup_guide.md))

## Private ssh keys
This tutorial is trying to setup secured access to your server, therefore we will (among other things) change default ssh ports, disable root login and disable password authentication. That is why we need these ssh keys which will be used for authentication. So everytime you would like to login to your server you would need these keys.

**Amazon's AWS or Digital Ocean** use their own solution so please follow the official guide provided by your hosting center. AWS gives you the option to setup your keys when creating first instance, Digital Ocean is described here [How to Upload SSH Public Keys to a DigitalOcean Account](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/to-account/)

If you have your own security settings you can skip this section and amend all the further related, but this is assuming you have your own way of access and know what you are doing.

This guide assumes your are working on a Mac so will provide steps for Mac, for Windows you can generate your key pair following e.g. this guide: [https://www.ssh.com/ssh/putty/windows/puttygen](https://www.ssh.com/ssh/putty/windows/puttygen)

On your Mac open Terminal application and run following commands:

```
# Generate private and public keys on your computer
# You will be prompted for file name to save the key, use your own name, don't use "id_rsa" so you can easily
# recognize the purpose of the file in future, remember the path and make sure you can browse the file, when
# prompted for password, select one you remember with at least 10 characters
ssh-keygen

# Change permission on your private key so it is locked and protected for changes, put below the path and file
# name you used in the previous command
chmod 400 ~/.ssh/<YOUR KEY>

# Copy your key to your VPS
# If using Amazon or Digital Ocean follow the official guidelines as stated above
ssh-copy-id -i ~/.ssh/<YOUR KEYNAME>.pub root@<YOUR VPS PUBLIC IP ADDRESS>
```

## Login to VPS via ssh using your new ssh keys
This will store your keys on remote server machine and make it trusted. When prompted answer yes to store your keys on the server.
```
# Amend the path and file name to the steps before and add your VPS public IP address
ssh -i ~/.ssh/<PATH TO SSH PRIVATE KEY ON YOUR LOCAL MACHINE> root@<YOUR VPS PUBLIC IP ADDRESS>
```

## Create new user account
Create a non-root user that will run all the cardano-nodes on this VPS. Many users create "ubuntu" so we don't recommend that for security users, feel free to create any and replace in the code below.

Keep logged in to ssh session until you are asked to log out 
```
# Create user and password
useradd <USERNAME> && passwd <USERNAME>

# Add non-root user to sudo group
usermod -aG sudo <USERNAME>

# Give permissions to new user
sudo visudo

# You should now be in the editor called "nano"
# ctrl+o to save, ctrl+x to quit
# add entry for new user under "User privilege specification" under root account
<USERNAME> ALL=(ALL:ALL) ALL

# Now that you've saved & quit the file above...
# Add dir and permissions
mkdir /home/<USERNAME>
chown <USERNAME>:<USERNAME> /home/<USERNAME> -R

# Copy pub key to new user
rsync --archive --chown=<USERNAME>:<USERNAME> ~/.ssh /home/<USERNAME>

# Set new user shell to bash
chsh -s /bin/bash <USERNAME>
```

## Update your Linux operating system
Still working under root account, if not possible (e.g. AWS) ad "sudo" before each command below
```
apt update
apt upgrade -y
```

## Install general necessary packages
Some of these might be already installed on your system.
```
# Install jq for parsing json
apt install jq

# Install chrony for time synchronization
apt-get install chrony
```

## Increase open file limits
Useful for Jörmungandr, might be obsolete for cardano-node, but can not harm the system so we will increase the limits
```
nano /etc/security/limits.conf

# Add the following at the bottom of the file
<USERNAME> soft nofile 800000
<USERNAME> hard nofile 1048576

# Save & close the file
ctrl+o
ctrl+x
```

## Disable firewall
We will change the ssh port and therefore will disable the firewall
```
# To avoid any lockouts, disable the firewall
ufw disable
```
