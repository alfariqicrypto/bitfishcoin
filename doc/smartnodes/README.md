# Build Smartnodes to get POS rewards

# A: VPS security and other settings

For the following part of the guide you need to be root. depending on your VPS provider they may have only provided you with a "sudo" user. You can change to root by doing:

```bash
sudo su
```

### 1. Update Server

```bash
apt update && apt upgrade -y
apt install unzip fail2ban -y
```

### 2. Add 4GB of SWAP

First check to make sure there is not already swap active:

```bash
free -h
```

If no swap it will return:

```bash
Swap:            0B          0B          0B
```

Create SWAP and Activate:

```bash
dd if=/dev/zero of=/swapfile bs=1k count=4096k
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile       swap    swap    auto      0       0" | tee -a /etc/fstab
sysctl -w vm.swappiness=10
echo vm.swappiness = 10 | tee -a /etc/sysctl.conf
```

<div class="alert alert--info" role="alert">
  <sub>
    Note: <code>swappiness = 10</code> tells system only to use it if absolutely
    needed.
  </sub>
</div>

### 3. Enable UFW & Open Ports

(Optional)

```bash
apt install ufw -y
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 21002/tcp
ufw enable
```

### 4. Configure Fail2Ban

As I am not bothering with setting up SSH login with keys rather then password based login need to keep something from the bots and kiddies brute forcing our SSH service.

Setup jail for bad guys hitting SSH, and set it to ban after three failed logins to SSH:

```bash
nano /etc/fail2ban/jail.local
```

Copy and paste the following into the file:

```bash
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
```

Reboot the server:

```bash
reboot
```

Add a system user to run bitfishcoind:

```bash
adduser <username_here>
```

Get Bitfishcoin wallet / daem on and bootstrap:

```bash
wget https://github.com/Altcoin-Master/Bitfishcoin/releases/download/v1.1.1.1/btsc-linux-1.1.1.1.zip
unzip btsc-linux-1.1.1.0.zip
chmod -R 755 btsc-linux-1.1.1.0
mkdir ~/.bitfishcoincore && touch ~/.bitfishcoincore/bitfishcoin.conf
echo '
rpcuser=username
rpcpassword=password
rpcallowip=127.0.0.1
rpcport=11002
server=1
txindex=1
daemon=1
' >> ~/.bitfishcoincore/bitfishcoin.conf
wget -P /.bitfishcoincore https://github.com/Altcoin-Master/Bitfishcoin/releases/download/v1.1.1.1/bootstrap.zip && unzip /.bitfishcoincore/bootstrap.zip -d /.bitfishcoincore
wget -P /.bitfishcoincore https://github.com/Altcoin-Master/Bitfishcoin/releases/download/v1.1.1.1/powcache.dat
~/btsc-linux-1.1.1.0/./bitfishcoind -daemon
```
```bash
#Common commands:
~/btsc-linux-1.1.1.0/./bitfishcoin-cli getblockchaininfo
~/btsc-linux-1.1.1.0/./bitfishcoin-cli getnetworkinfo
~/btsc-linux-1.1.1.0/./bitfishcoin-cli getbalance
~/btsc-linux-1.1.1.0/./bitfishcoin-cli stop
```

You should see Bitfishcoind server starting. You can confirm it is running <code>top -c</code> it will be using 100% CPU (1 core). This shows you it is working, we will return to it later.

# B: Wallet settings, lock the coins in your own wallet.

### 1. Install and Sync Local Wallet:

- Make sure you are using the latest version of the wallet and have synced, don't forget to encrypt and backup your wallet.dat to multiple places.
- Unlock your QT wallet, or use the command below to unlock

```
walletpassphrase YouPassword 600
Explanation: Unlock wallet with password for 600 seconds
```

- Send 200,000 BTSC to self (this is the minimum collateral amount), Wait for 2 confirmations
- Smartnodes rules:
```
        0 - 100000    200,000 BTSC
   100001 - 200000    400,000 BTSC
   200001 - 300000    600,000 BTSC
   300001 - 400000    800,000 BTSC
   400001 - forever   1,000,000 BTSC
```

The private key only allows you to restore the matching receiving address. If you setup multiple nodes you should dump the private key for each collateral receiving address.

### 2. Build protx command for control wallet {#build-protx}

Here is an example protx quick_setup command:

```bash
protx quick_setup "Transaction ID" "Collateral index" "Your smartnode server IP:21002" "Fee address"
Example:
protx quick_setup "c4bbcde9771668fa640c263d4b964f688b0f039f7b684e715d92e4012369fea6" "1" "127.0.0.1:21002" "BFbWv94ZfueciwVVpHLMdqFayaXAS4sBxP"
```

The structure from left to right is:

- <code>Transaction ID</code>: In your wallet go to "Transactions" right click
  the one you sent yourself earlier and "Copy Transaction ID". Replace the
  Transaction ID in example.
- <code>Collateral index</code>: Tools > Debug console. Type "smartnode outputs"
  to check if 1 or 0. Adjust example command if needed.
- <code>Your smartnode server IP and port</code>. Replace example IP with your
  Smartnode server IP, leave port as is.
- <code>Fee address</code>: This is any address in your wallet which contains
  enough BTSC to pay the fee (cannot be the address which you sent the 1 million
  BTSC to). When you enter the protx quick_setup command it is a transaction and
  needs to be paid for. It is a very small amount 1/2 an BTSC is more than
  enough. In Debug console do "listaddressbalances" to show all addresses with a
  balance, choose one and replace address in example command.

Enter the protx quick_setup command in Debug console. This will create a .conf file for that node in the same directory you ran the wallet from. Open it and copy the contents.

# C: VPS setup (skip if you don't have a VPS)

Back to the smartnode server:

```bash
~./bitfishcoin-cli stop
nano ~/.bitfishcoincore/bitfishcoin.conf
```

Paste in what you copied from the .conf file made during the protx command, save and exit.

When done with this, run bitfishcoind from the terminal:

```bash
~/./bitfishcoind -daemon
```

When this is done, wait a couple minutes until running `~/./bitfishcoin-cli smartnode status` to check the status of your smartnode. This should return: `Ready Ready`

# D: View node status in QT wallet
Settings - Options - Wallet - Show Smartnode Tab
