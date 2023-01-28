# Sui node setup for tesnet wave 2 via Service method

Official documentation:
- Official manual: https://docs.sui.io/build/fullnode
- Experiment with Sui DevNet: https://docs.sui.io/explore/devnet
- Check you node health: https://node.sui.zvalid.com/

## Minimum hardware requirements
- CPU: 2 CPU
- Memory: 4 GB RAM
- Disk: 50 GB SSD Storage

## Recommended hardware requirements
- CPU: 2 CPU
- Memory: 8 GB RAM
- Disk: 50 GB SSD Storage



1. Install Linux dependencies.
```
sudo apt-get update \
&& sudo apt-get install -y --no-install-recommends \
tzdata \
ca-certificates \
build-essential \
libssl-dev \
libclang-dev \
pkg-config \
openssl \
protobuf-compiler \
cmake
```

2. Install Rust.
```
sudo curl https://sh.rustup.rs -sSf | sh -s -- -y
source $HOME/.cargo/env
rustc --version
```

3. Clone GitHub SUI repository.
```
cd $HOME
git clone https://github.com/MystenLabs/sui.git
cd sui
git remote add upstream https://github.com/MystenLabs/sui
git fetch upstream
git checkout -B testnet --track upstream/testnet
```

4. Create directory for SUI db and genesis.
```
mkdir $HOME/.sui
```

5. Download genesis file (instead of placeholder use the link to genesis you received in email).
```
wget -O $HOME/.sui/genesis.blob  https://github.com/MystenLabs/sui-genesis/raw/main/testnet/genesis.blob
```

6. Make a copy of fullnode.yaml and update path to db and genesis file in it.
```
cp $HOME/sui/crates/sui-config/data/fullnode-template.yaml $HOME/.sui/fullnode.yaml
sed -i.bak "s|db-path:.*|db-path: \"$HOME\/.sui\/db\"| ; s|genesis-file-location:.*|genesis-file-location: \"$HOME\/.sui\/genesis.blob\"| ; s|127.0.0.1|0.0.0.0|" $HOME/.sui/fullnode.yaml
```

7. Build SUI binaries.

```
cargo build --release --bin sui-node
mv ~/sui/target/release/sui-node /usr/local/bin/
sui-node -V
```
### YANG INI SALAH JGN DI COPY
```
cargo build --release
mv ~/sui/target/release/sui-node /usr/local/bin/
mv ~/sui/target/release/sui /usr/local/bin/
sui-node -V && sui -V
```

8. Create Service file for SUI Node.
```echo "[Unit]
echo "[Unit]
Description=Sui Node
After=network.target

[Service]
User=$USER
Type=simple
ExecStart=/usr/local/bin/sui-node --config-path $HOME/.sui/fullnode.yaml
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target" > $HOME/suid.service

mv $HOME/suid.service /etc/systemd/system/

sudo tee <<EOF >/dev/null /etc/systemd/journald.conf
Storage=persistent
EOF
```

9. Generate wallet
```
echo -e "y\n\n0\n" | sui client
```
> !Please backup your wallet key files located in `$HOME/.sui/sui_config/` directory!
## Top up your wallet
10. Get your wallet address:
```
sui client active-address
```

11. Navigate to [Sui Discord](https://discord.gg/sui) `#devnet-faucet` channel and top up your wallet
```
!faucet <YOUR_WALLET_ADDRESS>
```

![image](https://user-images.githubusercontent.com/50621007/180215182-cbb7fc6c-aba3-4834-ad05-f79e1c26b40c.png)

12. Wait until bot sends tokens to your wallet

![image](https://user-images.githubusercontent.com/50621007/180222321-1dc5323b-1174-41c8-b632-6ac2ce639ce1.png)

> OR YOU CAN TRY REQUEST FAUCET WITH CLI

### Request test tokens through cURL
Use the following cURL command to request tokens directly from the faucet server:
```
curl --location --request POST 'https://faucet.devnet.sui.io/gas' \
--header 'Content-Type: application/json' \
--data-raw '{
    "FixedAmountRequest": {
        "recipient": "<YOUR SUI ADDRESS>"
    }
}'
```

### Request test tokens through TypeScript SDK
You can also access the faucet through the TS-SDK.

```
import { JsonRpcProvider, Network } from '@mysten/sui.js';
// connect to Devnet
const provider = new JsonRpcProvider(Network.DEVNET);
// get tokens from the Devnet faucet server
await provider.requestSuiFromFaucet(
  '<YOUR SUI ADDRESS>'
);
```

13. Start SUI Full Node in Service.
```sudo systemctl restart systemd-journald
sudo systemctl daemon-reload
sudo systemctl enable suid
sudo systemctl restart suid
journalctl -u suid -f
```


### Get the five most recent transactions
```
curl --location --request POST 'http://127.0.0.1:9000/' --header 'Content-Type: application/json' \
--data-raw '{ "jsonrpc":"2.0", "id":1, "method":"sui_getRecentTransactions", "params":[5] }' | jq .
```

### Get details about a specific transaction
```
curl --location --request POST 'http://127.0.0.1:9000/' --header 'Content-Type: application/json' \
--data-raw '{ "jsonrpc":"2.0", "id":1, "method":"sui_getTransaction", "params":["<RECENT_TXN_FROM_ABOVE>"] }' | jq .
```

## Operations with objects
Now lets do some operations with objects

### Merge two objects into one
```
JSON=$(sui client gas --json | jq -r)
FIRST_OBJECT_ID=$(echo $JSON | jq -r .[0].id.id)
SECOND_OBJECT_ID=$(echo $JSON | jq -r .[1].id.id)
sui client merge-coin --primary-coin ${FIRST_OBJECT_ID} --coin-to-merge ${SECOND_OBJECT_ID} --gas-budget 1000
```

You should see output like this:
```
----- Certificate ----
Transaction Hash: t3BscscUH2tMnMRfzYyc4Nr9HZ65nXuaL87BicUwXVo=
Transaction Signature: OCIYOWRPLSwpLG0bAmDTMixvE3IcyJgcRM5TEXJAOWvDv1xDmPxm99qQEJJQb0iwCgEfDBl74Q3XI6yD+AK7BQ==@U6zbX7hNmQ0SeZMheEKgPQVGVmdE5ikRQZIeDKFXwt8=
Signed Authorities Bitmap: RoaringBitmap<[0, 2, 3]>
Transaction Kind : Call
Package ID : 0x2
Module : coin
Function : join
Arguments : ["0x530720be83c5e8dffde5f602d2f36e467a24f6de", "0xb66106ac8bc9bf8ec58a5949da934febc6d7837c"]
Type Arguments : ["0x2::sui::SUI"]
----- Merge Coin Results ----
Updated Coin : Coin { id: 0x530720be83c5e8dffde5f602d2f36e467a24f6de, value: 100000 }
Updated Gas : Coin { id: 0xc0a3fa96f8e52395fa659756a6821c209428b3d9, value: 49560 }
```

Lets yet again check list of objects
```
sui client gas
```

We can see that two first objects are now merged into one and gas has been payed by third object

![image](https://user-images.githubusercontent.com/50621007/180228094-10b563f4-ea6f-42cd-b560-6abeda47c2df.png)

>This is only one example of transactions that can be made at the moment. Other examples can be found at the [official website](https://docs.sui.io/build/wallet)



## Usefull commands for sui fullnode
Check sui node status
```
systemctl status suid
```

Check node logs
```
journalctl -fu suid -o cat
```

Check sui client version
```
sui --version
```

### Check Your Node Status Via CLI
```
curl -s -X POST http://127.0.0.1:9000 -H 'Content-Type: application/json' -d '{ "jsonrpc":"2.0", "method":"rpc.discover","id":1}' | jq .result.info
```

You should see something similar in the output:
```json
{
  "title": "Sui JSON-RPC",
  "description": "Sui JSON-RPC API for interaction with the Sui network gateway.",
  "contact": {
    "name": "Mysten Labs",
    "url": "https://mystenlabs.com",
    "email": "build@mystenlabs.com"
  },
  "license": {
    "name": "Apache-2.0",
    "url": "https://raw.githubusercontent.com/MystenLabs/sui/main/LICENSE"
  },
  "version": "0.1.0"
}
```

### Check Your Node Status via Website
https://www.scale3labs.com/check/sui


