# docker-qtum-raspberry-pi
Docker image that runs the QTUM qtumd node in a container on Raspberry Pi for easy QTUM staking.

This repository is fork of [trick77's excellent x86 repo](https://github.com/trick77/docker-qtum-stake)!

Just want to run a full node without staking? Please see the instructions below.

## Requirements
- Docker and Docker Compose installed on a Raspberry Pi
- Since QTUM is PoS you also need coins in order to stake
- The ability to remember a passphrase for your QTUM wallet

## A word of caution 
Since the QTUM wallet is held in a Docker volume you have to be careful not to delete the container's named volume. Make sure to backup the wallet to multiple mediums! 
You can use the provided ```backup-wallet.sh``` shell script in order to copy the wallet from the volume to the current directory. 

## Usage

Clone this repository first and cd to its directory.

### Starting the QTUM node
Thanks to docker-compose starting the node is as easy as it gets:
```docker-compose up -d```

The docker image will be built on the fly if it doesn't exist.
Once the container is up and running, qtumd will start syncing the blockchain which may take a while. To see the progress use something like ```docker-compose logs -f --tail="100"```

#### Just want to run a full node?

You don't need to have any QTUM in your wallet in order to just run a full node. Full nodes help the QTUM network by accepting transactions and blocks from other full nodes, validating those transactions and blocks, and then relaying them to further full nodes. If you plan to run a full node on a virtual private server (VPS), please contact your hosting provider first since 
some providers prohibit running anything related to crypto currencies/mining (even though it's not possible to PoW mine with QTUM). 
A ```docker-compose up -d``` will run a full node, you can skip the rest of this documentation if you don't want to stake QTUM.

### Encrypting the wallet
```docker-compose exec qtum qtum-cli -stdin encryptwallet```

Enter a passphrase and hit \<ENTER\> and \<CTRL-D\> to terminate the input. **Please be patient after pressing \<CTRL-D\> since it may take a while to encrypt the wallet.** There's an option to pass the passphrase via the command line but it's less secure since most shells save a command history (and the passphrase entered).

Once the wallet has been encrypted, the following message will be displayed: 
```wallet encrypted; Qtum server stopping, restart to run with encrypted wallet. The keypool has been flushed and a new HD seed was generated (if you are using HD). You need to make a new backup.```

Restart the container to use the encrypted wallet:
```docker-compose down && docker-compose up -d```

### Unlocking the wallet

To confirm the wallet was encrypted with the intended passphrase:
 ```./qtum-cli.sh -stdin walletpassphrase```

Followed by:
1. The wallet's passphrase and \<ENTER\>
2. An numeric unlock duration in seconds, i.e. 120 followed by \<ENTER\>
3. \<CTRL-D\>

If the passphrase was not accepted, an error message will be displayed. 
Alternatively, you could also check if the unlocked_until attribute is > 0 with ```./qtum-cli.sh getwalletinfo | grep unlocked_until```

A value > 0 means the wallet is encrypted and the above unlocking command was successful.

### Backing up the wallet

Now would be a good time to backup the encrypted wallet with the ```backup-wallet.sh``` shell script. Make sure to store the wallet on multiple mediums before continuing.

### Displaying the account's default address
```./qtum-cli.sh getaccountaddress '""'```

In order to stake, QTUM coins have to be transferred to this address.

### Starting to stake QTUM

Once the coins are available in the wallet, the staking requirement has been met (not moved for 500 blocks) and the wallet has been unlocked, staking will finally commence.
To unlock the wallet just for staking:
```./qtum-cli.sh -stdin walletpassphrase```

Followed by:
1. The wallet's passphrase and \<ENTER\>
2. A numeric unlock duration in seconds, but this time we will use a high value like 99999999 followed by \<ENTER\>
3. true \<ENTER\>
4. \<CTRL-D\>

The third argument (true) indicates the wallet will only be unlocked for staking. No coins can be moved if the wallet is opened this way. If the container is restarted, the wallet has to be unlocked for staking again. 

#### Staking state check

If you save this script to **the host's** ```/etc/cron.hourly``` directory (if available in your Linux distro) it will alert root periodically if staking is not enabled.

```
#!/bin/bash
CONTAINER_NAME=qtum
if [ ! $(docker ps -q -f name=${CONTAINER_NAME}) ]; then
    exit 0
fi
if docker exec ${CONTAINER_NAME} qtum-cli getstakinginfo | grep -qE '.*staking.*false.*'; then
    >&2 echo "Warning: wallet is not staking!"
    exit 1
fi
```

Don't forget to chmod +x it and don't use a file extension or it will not run.

### More CLI commands
To get the full list of supported CLI commands use ```./qtum-cli.sh help```


