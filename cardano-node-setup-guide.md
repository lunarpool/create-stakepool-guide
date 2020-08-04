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

## Change default ssh port
Make sure to you proceed the ufw section as well which is required  to allow communication to your new ssh port, otherwise your server may end up unaccessible
```
# Changing this setting REQUIRES also opening the same port with ufw (next section of this guide)
# Don't skip the ufw section, or else you will be locked out.

# Note: there is also a file called "ssh_config"... don't edit that one
nano /etc/ssh/sshd_config

# Change the line "#Port 22", to "Port <CHOOSE A PORT BETWEEN 1024 AND 65535>"
# Remember to uncomment = remove the leading "#"

# While we're here, let's give ourselves just a bit more time before getting disconnected, ie "broken pipe".
# Change the line "#TCPKeepAlive yes" to "TCPKeepAlive no"
# Change the line "#ClientAliveInterval 0" to "ClientAliveInterval 1800"

# Save & close the file
ctrl+o
ctrl+x
```

## Configure firewall (ufw)
```
# Set defaults for incoming/outgoing ports
ufw default deny incoming
ufw default allow outgoing

# Open ssh port (rate limiting enabled - max 10 attempts within 30 seconds)
ufw limit proto tcp from any to any port <THE PORT YOU JUST CHOSE IN sshd_config IN STEPS ABOVE>

# Open a port for your public_address. This is the port other nodes will connect to.
ufw allow proto tcp from any to any port 3000

# Re-enable firewall
ufw enable

# Double-check the port you chose for ssh was the same as what you set in /etc/ssh/sshd_config
# First command returns ssh port you set up
# Second command lists firewall rules, you should see your new ssh port with ALLOW IN
grep "Port " /etc/ssh/sshd_config
ufw status verbose

# Double-check your new user is in the sudo group
# This should return your username
grep '^sudo:.*$' /etc/group | cut -d: -f4

# Reboot (You will be kicked off... wait a couple of seconds before logging in)
reboot
```

## Login with your new user
Use terminal or New Remote Connection with following setup, use the user, ssh port and private key you just created
```
ssh -p <SSH PORT> -i ~/.ssh/<YOUR SSH PRIVATE KEY> <USERNAME>@<YOUR VPS PUBLIC IP ADDRESS>
```

## Finalize ssh setting
```
# Edit sshd_config one more time
sudo nano /etc/ssh/sshd_config

# Reduce logging to save resources
(Change "LogLevel" from "INFO" to "VERBOSE"

# Disabling root login is considered a security best-practice
(Change "PermitRootLogin" from "yes" to "no")

# Disabling log-in via password helps mitigate brute-force attacks
(Change "PasswordAuthentication" to "no")

(ctrl+o to save, ctrl+x to exit)

# Reload the ssh daemon
# NOTE: Once you do this, you will only be able to log-in using your SSH private key as non-root user
sudo service sshd reload
```

## Configuring SWAP memory
If you run out of physical memory, you use virtual memory, which stores the data in memory on the disk (so not in RAM but in your HDD/SDD). Many VPS come with already pre-configured swap so we will firstly check with:
```
# Show current swap configuration
sudo swapon --show
```
If you get back a line with current setup then you can skip this section, if you have limited diskspace skip as well, otherwise continue with the commands below:
```
# Check what swap is currently active, if any
free -h

# Check current disk usage
df -h

# Create swap file (Don't forget the "G")
sudo fallocate -l <SIZE EQUAL TO RAM>G /swapfile

# Verify swap settings
ls -lh /swapfile

# Only root can access swapfile
sudo chmod 600 /swapfile

# Mark the file as swap space
sudo mkswap /swapfile

# Enable swap settings every time we log in
# Make a backup of /etc/fstab
sudo cp /etc/fstab /etc/fstab.bak

# Type this command from the command-line to add swap settings to the end of fstab
echo '/swapfile none swap sw 00' | sudo tee -a /etc/fstab

# Enable swap
sudo swapon -a

# Verify swap is enabled
free -h
```

## Setup your Bash environment
We have to append the following code to your ~/.bashrc file, it will basically setup the default paths within your environment. So everytime you run the shell these settings will be applied.

```
# Edit Bash configuration script
nano ~/.bashrc
```

Append the following lines (don't forget to replace placeholder text)
```
export PS1="<SERVER NAME> \[\e[36m\]\w\[\e[m\]\[\e[35m\] \`parse_git_branch\`\[\e[m\] \[\e[36m\]:\[\e[m\] "
export PATH="~/.cabal/bin:$PATH"
export PATH="~/.local/bin:$PATH"
export PATH="~/scripts:$PATH"
export LD_LIBRARY_PATH="/usr/local/lib:$LD_LIBRARY_PATH"
export CARDANO_NODE_SOCKET_PATH="/home/<YOUR USERNAME>/cardano-node/node1/db/node.socket"
export CNODE_HOME=/opt/cardano/cnode
```
Save and exit using ctrl+o, ctrl+x

```
# Edit Bash profile
nano ~/.bash_profile
```
Paste the following into .bash_profile file
```
export ARCHFLAGS="-arch x86_64"
test -f ~/.bashrc && source ~/.bashrc
```
Save and exit using ctrl+o, ctrl+x

Load configuration immmediately
```
source ~/.bashrc
```

## Time synchronization via chrony
Edit the configuration file /etc/chrony/chrony.conf
```
sudo nano /etc/chrony/chrony.conf
```
Paste the following into /etc/chrony/chrony.conf (don't forget to replace placeholder text)
```
# Welcome to the chrony configuration file. See chrony.conf(5) for more
# information about usuable directives.

# This will use (up to):
# - 4 sources from ntp.ubuntu.com which some are ipv6 enabled
# - 2 sources from 2.ubuntu.pool.ntp.org which is ipv6 enabled as well
# - 1 source from [01].ubuntu.pool.ntp.org each (ipv4 only atm)
# This means by default, up to 6 dual-stack and up to 2 additional IPv4-only
# sources will be used.
# At the same time it retains some protection against one of the entries being
# down (compare to just using one of the lines). See (LP: #1754358) for the
# discussion.
#
# About using servers from the NTP Pool Project in general see (LP: #104525).
# Approved by Ubuntu Technical Board on 2011-02-08.
# See http://www.pool.ntp.org/join.html for more information.
#pool ntp.ubuntu.com        iburst maxsources 4
#pool 0.ubuntu.pool.ntp.org iburst maxsources 1
#pool 1.ubuntu.pool.ntp.org iburst maxsources 1
#pool 2.ubuntu.pool.ntp.org iburst maxsources 2

pool <CLOSEST NTP SERVER>              iburst minpoll 1 maxpoll 1 maxsources 3
pool <SECOND CLOSEST NTP SERVER>       iburst minpoll 1 maxpoll 1 maxsources 3
pool time.google.com                   iburst minpoll 1 maxpoll 1 maxsources 3
pool ntp.ubuntu.com                    iburst minpoll 1 maxpoll 1 maxsources 3

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Uncomment the following line to turn logging on.
#log tracking measurements statistics

# Log files location.
logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 5.0

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can’t be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 0.1 -1

# Get TAI-UTC offset and leap seconds from the system tz database
leapsectz right/UTC

# Serve time even if not synchronized to a time source.
local stratum 10

(ctrl+o to save, ctrl+x to exit)
```
Restart chrony to apply the configuration
```
sudo systemctl restart chrony
```
## Configure tmux
Prepare config file to enable mouse control for multi terminal environment. The following will append a line to .tmux.conf file
```
echo "set -g mouse on" >> ~/.tmux.conf
```

# You finished your linux setting! Buy Chris Graffagnino a beer!
Up to this point almost all the steps and settings were adoptet from Chris's [Jormungandr-for-Newbs](https://github.com/Chris-Graffagnino/Jormungandr-for-Newbs/blob/master/docs/jormungandr_node_setup_guide.md) guide.
```
DdzFFzCqrht3kMqsjpaLjr3L8tw5Jn2E9Vr9id9R33jB1P4TqRKZ87UVkzrF9NMarNLNKx5fuahvHiaD4Cz9K71CD69QQDBzS5mExsMr
```

# Let's continue installing cardano-node from source
Primary source for next steps was github repo used for F&F testnet as [Node Setup Tutorials](https://github.com/input-output-hk/cardano-tutorials/tree/master/node-setup) which has moved to current official documentation [https://docs.cardano.org/en/latest/](https://docs.cardano.org/en/latest/)

## Install dependencies
```
sudo apt-get update -y
sudo apt-get install automake build-essential pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make g++ tmux git jq wget libncursesw5 libtool autoconf -y
```

## Install Cabal
```
wget https://downloads.haskell.org/~cabal/cabal-install-3.2.0.0/cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz
tar -xf cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz
rm cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz cabal.sig
mkdir -p ~/.local/bin
mv cabal ~/.local/bin/
```
## Update Cabal
```
cabal update
```

## Edit cabal config
```
# Open config file
nano ~/.cabal/config

# Locate line with "overwrite-policy"
# Uncomment by deleting leading --
# (Change "overwrite-policy" to "always")

(ctrl+o to save, ctrl+x to exit)
```

## Install GHC
```
wget https://downloads.haskell.org/~ghc/8.6.5/ghc-8.6.5-x86_64-deb9-linux.tar.xz
tar -xf ghc-8.6.5-x86_64-deb9-linux.tar.xz
rm ghc-8.6.5-x86_64-deb9-linux.tar.xz
cd ghc-8.6.5
./configure
sudo make install
```

## Install Libsodium
```
cd
git clone https://github.com/input-output-hk/libsodium
cd libsodium
git checkout 66f017f1
./autogen.sh
./configure
make
sudo make install
```

## Download cardano-node
Don't forget to replace <TAGGED VERSION> with the latest tag (in the time of writing 1.18.0)
```
cd
git clone https://github.com/input-output-hk/cardano-node.git
cd cardano-node
git fetch --all --tags
git tag
git checkout tags/<TAGGED VERSION>
```

## Build and install cardano-node
This will take some time (a lot of) so in the meantime you can run second ssh session and continue further while cardano-node is building.
```
cabal build all
```

# Configuring cardano-node

## Configuring more nodes on a server
This tutorial configures 1 node (instance of cardano-node) on the server that operates with following condidtions:
* Node port: 3000
* Node folder: ~/cardano-node/node1

If you would like to run multiple instance on the same server you just need to open another firewall ports:
```
ufw allow proto tcp from any to any port <PORT NUMBER>
```
and repeat the steps below for folders like ~/cardano-node/node2, ~/cardano-node/node3 and so on.

# Configuring cardano-node "node1"

## Download default configuration, genesis and topology files
```
# Create node folder (for config and db files)
cd ~/cardano-node
mkdir node1
cd node1

# Download config, genesis, topology files
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-config.json
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-byron-genesis.json
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-shelley-genesis.json
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-topology.json
```

## Edit topology.json
Add peers to your topology.json, this depends on your desired infrastructure and the node you currently set up. By default the topology file consist of a pair of nodes provided by IOHK, which is sufficient in early stages of Shelley. Recommended topology is core –> private relay –> public relay –> other public relays but this is out of scope od this guide.
```
nano ~/cardano-node/node1/topology.json
```
