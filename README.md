[Русский](README_RU.md)
# [guard.sh](https://github.com/Hohlas/solana-guard/blob/main/guard.sh)

Solana Guard is a multifunctional script for seamless Solana validator failover. It continuously monitors your validator’s status and automatically fixes node and network issues by switching voting to a backup server, keeping your node online and the Solana network resilient.

![Guard Overview](https://github.com/user-attachments/assets/fe88aafe-5b4a-4cbf-911e-27bb94b77f8b)


## Key Features

- **Automatic Failover**: Instantly switches voting to the secondary (backup) server when the primary server node fails.
- **Forced Failover**: Switches voting while node upgrade with the \`guard p\` argument in just 1–2 seconds, with the minimum TCV loss. The script checks if there is sufficient time before the next block and verifies the secondary server's health and Behind status before automatically initiating voting.
- **Continuous Operation**: Automatically adjusts server roles (Primary/Secondary) based on node voting status without requiring a script restart.
- **Mutual Monitoring**: The primary server monitors the \`guard.sh\` script on the secondary server, and vice versa, preventing accidental monitoring interruptions.
- **Node Health Monitoring**: Tracks critical metrics for both servers, including:
  - Next slot minutes.
  - Health status (\`health\`).
  - Behind status for both local and remote servers.

  ![Node Status](https://github.com/user-attachments/assets/ecc683bf-252e-4c52-86e0-c60e9bf6f876)


- **Comprehensive Logging**: Records all events, including Behind incidents and node switches, in a log file.

  ![Log Example](https://github.com/user-attachments/assets/f7b1e38e-a728-4728-8256-bb9cc657feb3)

- **Telegram Info & Alert messages**: Sends notifications for node switches and Behind events on both servers via Telegram.

  ![Telegram Info](https://github.com/user-attachments/assets/39973c04-1557-47f1-9234-361a32c449ba)
  
  ![Telegram Alert](https://github.com/user-attachments/assets/bb70a748-9c7e-4e81-90da-a741645402c1)

  ![Telegram Start voting](https://github.com/user-attachments/assets/a9321895-9523-4e89-a155-5872fe11fe2a)

  ![Telegram behind](https://github.com/user-attachments/assets/4efc9ece-b212-415f-89f5-0f2b41af0c0c)



- **Priority Server Assignment**: Designates a priority server for voting with the \`guard p\` argument, automatically reverting voting to the priority server once its health and Behind status normalize.

  ![Permanent Primary](https://github.com/user-attachments/assets/47dba555-5daf-49ce-a0ff-51bea7a7ba2b)


- **Proactive Failover**: Triggers voting failover if the primary server is Behind for \`X*5\` seconds (configured with \`guard X\`), without waiting for Delinquent status.

  ![Guard with Threshold](https://github.com/user-attachments/assets/4c121091-6b62-4e81-85b6-d2458ccc1e88)


- **Connection Loss Handling**: Restarts the Solana service in "No Voting" mode on the primary server if internet connectivity is lost for over 15 seconds, in order to prevent duplicate voting.
- **Service Management**: Synchronizes \`telegraf\` and \`relayer\` services with the Primary/Secondary status of each server.

## Secondary Server Workflow

- **Monitoring**: Continuously tracks the health and Behind status of both the secondary and voting nodes.
- **Role Detection**: Compares the voting node's IP address with the local IP. If they match, the server transitions to Primary status.
- **Failover Triggers**: Initiates voting failover to the secondary server when the secondary node has no Behind and one of the following occurs:
  - The voting node is in Delinquent status.
  - The voting node's Behind exceeds the threshold (\`X*5\` seconds) when started with \`guard X\`.
  - The \`permanent_primary\` flag is set with \`guard p\`.
- **Failover Process**:
  - Disables voting on the primary server by switching to a non-voting key.
  - Copies the tower file from the primary server (updated every 5 seconds to handle potential internet outages).This way, if the internet connection on the Primary server becomes unavailable, the latest up-to-date tower file will be available for failover and launching the Solana validator on the backup server.
  - Enables voting on the secondary server.
  - Disables \`telegraf\` and \`relayer\` services on the primary server and enables them on the secondary.
  - Waits for the network to update the node's IP address, transitioning from Secondary to Primary status.
- **Error Handling**:
  - If switching the key on the primary server fails, restarts the Solana service in "No Voting" mode.
  - If the restart fails, attempts to reboot the primary server via SSH.
  - If reboot fails, checks ping to the primary server.
  - If the primary server is unreachable, assumes it has restarted in "No Voting" mode due to connectivity loss.
  - If the primary server is reachable, continues attempts to disable voting.
  - Before enabling voting, verifies the primary server's voting status using \`solana-validator --ledger '\$LEDGER' contact-info\` to prevent duplicate voting.

## Primary Server Workflow

- **Connection Monitoring**: Restarts the Solana service in "No Voting" mode if internet connectivity is lost for over 15 seconds.
- **Role Detection**: Compares the voting node's IP with the local IP. If they differ, transitions to Secondary status.
- **Health Monitoring**: Tracks health and Behind status for both the voting and secondary nodes.

## Installation

### Download \`guard.sh\`
Fetch the latest version and set up an alias:
```bash
# install curl jq bc
if ! command -v curl >/dev/null || ! command -v jq >/dev/null || ! command -v bc >/dev/null; then
    echo "install curl jq bc..."
    sudo apt update && sudo apt install -y curl jq bc
fi

# download guard.sh
LATEST_TAG_URL=https://api.github.com/repos/Hohlas/solana-guard/releases/latest
TAG=$(curl -sSL "$LATEST_TAG_URL" | jq -r '.tag_name')

if [ -d ~/solana-guard ]; then 
	echo "update latest release $TAG"
	curl -sSL https://raw.githubusercontent.com/Hohlas/solana-guard/$TAG/guard.sh > $HOME/solana-guard/guard.sh
	curl -sSL https://raw.githubusercontent.com/Hohlas/solana-guard/$TAG/check.sh > $HOME/solana-guard/check.sh
else 
	echo "clone latest release $TAG"
	git clone https://github.com/Hohlas/solana-guard.git ~/solana-guard
	cd ~/solana-guard
	git fetch --tags 
	git checkout tags/$TAG
	git submodule update --init --recursive
fi

chmod +x ~/solana-guard/guard.sh
chmod +x ~/solana-guard/check.sh

# set aliases
BASHRC_FILE="$HOME/.bashrc"
if [ -f "$HOME/.bashrc" ]; then
	NEW_ALIAS="alias guard='source ~/solana-guard/guard.sh'"
	if ! grep -qxF "$NEW_ALIAS" "$BASHRC_FILE"; then
		echo "$NEW_ALIAS" >> "$BASHRC_FILE"
		echo "add new alias: [$NEW_ALIAS]"
	fi
	NEW_ALIAS="alias check='source ~/solana-guard/check.sh'"
	if ! grep -qxF "$NEW_ALIAS" "$BASHRC_FILE"; then
		echo "$NEW_ALIAS" >> "$BASHRC_FILE"
		echo "add new alias: [$NEW_ALIAS]"
	fi
else
    echo "file $HOME/.bashrc not found."
fi

source $HOME/.bashrc
```

### Configure Solana Service
The Solana service must start with a non-voting key (\`empty-validator.json\`).

Generate a non-voting key:
```bash
if [ ! -f ~/solana/empty-validator.json ]; then 
    solana-keygen new -s --no-bip39-passphrase -o ~/solana/empty-validator.json
fi
```

Update \`solana.service\` configuration:
```bash
--identity /root/solana/empty-validator.json \
--authorized-voter /root/solana/validator-keypair.json \
--vote-account /root/solana/vote.json \
```

### Set Up Keys Directory
Create a RAM disk for the keys directory:
```bash
if [ ! -d "/mnt/keys" ]; then
    mkdir -p /mnt/keys
    chmod 600 /mnt/keys 
	echo "# KEYS to RAMDISK 
	tmpfs /mnt/keys tmpfs nodev,nosuid,noexec,nodiratime,size=1M 0 0" | sudo tee -a /etc/fstab
	mount /mnt/keys
	echo "create and mount /mnt/keys in RAMDISK"
else
    echo "/mnt/keys exist"
fi
```

Create symbolic links to the keys directory:
```bash
# create links
ln -sf /mnt/keys ~/keys
ln -sf /mnt/keys/vote-keypair.json ~/solana/vote.json
ln -sf /mnt/keys/validator-keypair.json ~/solana/validator-keypair.json
```

### Configure \`guard.cfg\`
Settings are stored in \`~/guard.cfg\`. For Telegram notifications, a \`BOT_TOKEN\` is required.

To improve reliability, the script queries an alternative RPC ([Helius RPC](https://dashboard.helius.dev)) alongside Solana's RPC. If responses match, the result is accepted. If they differ, 10 additional queries are made to each RPC, accepting the answer if 75% of responses agree. A free Helius account lasts about three days, so the script cycles through the \`RPC_LIST\` when one server stops responding.

Example \`guard.cfg\':
```bash
PORT='22' # remote server ssh port
KEYS=$HOME/keys
SOLANA_SERVICE=$HOME/solana/solana.service
BEHIND_WARNING=false # 'false'- send telegramm INFO missage, when behind. 'true'-send ALERT message
WARNING_FREQUENCY=12 # max frequency of warning messages (WARNING_FREQUENCY x 5) seconds
BEHIND_OK_VAL=3 # behind, that seemed ordinary
RELAYER_SERVICE=false # use restarting jito-relayer service
CHAT_ALARM=-100..684
CHAT_INFO=-100..888
BOT_TOKEN=507625...VICllWU
RPC_LIST=(
"https://mainnet.helius-rpc.com/..."
"https://mainnet.helius-rpc.com/..."
"https://mainnet.helius-rpc.com/..."
)
```

### SSH Key Setup
Place private SSH keys (\`any_name.ssh\`) for both servers in \`~/keys\`.

### Launch Order
Run \`guard.sh\` on the secondary server first to copy its IP to the primary server. Subsequent launches can be in any order.

### Update
The script automatically notifies when a new guard version is available. To update, simply run \`guard u\` — the latest version will be downloaded and installed automatically. 

![Update](https://github.com/user-attachments/assets/e7e20789-a0b0-46a6-aab6-f11c7f519232)


## Companion Script: \`check.sh\`
A utility for convenient server status monitoring.

![Check Status](https://github.com/user-attachments/assets/efe5076e-ead2-4ec9-841a-e9ed61a4d309)


## Notes
- Ensure \`bc\` is installed (\`apt install bc\`) for numerical calculations.
- The secondary server must be fully operational and reachable via SSH before running \`guard.sh\` on the primary server.
- Regularly update \`guard.sh\` to the latest version to benefit from bug fixes and improvements.
