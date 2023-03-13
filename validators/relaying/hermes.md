---
description: Instructions for setting up the rust based relayer, Hermes
cover: >-
  ../../.gitbook/assets/Gitbook Banner large 6 (1) (1) (1) (1) (1) (1) (1) (1)
  (1) (1) (1) (9).png
coverY: 0
---

# Hermes

## Assumptions

We assume that you already have access to Juno, Osmosis and Cosmos nodes. These can be either local nodes, or you can access them over the network. However, for networked version, you will need to adjust the systemd configuration not to depend on the chains that are run on other servers. And naturally the hermes configuration needs to adjust the addressing of each chain as well.

The given example has all relayed chains run locally, Juno is on standard ports, other chains are configured as follows:

* Osmosis: 36657 and 39090
* Cosmos: 46657 and 49090
* Sifchain: 56657 and 59090

In these instructions, Hermes is installed under /srv/hermes, adjust the paths according to your setup.

These instructions are based on installation on Debian 10, but should work the same on Debian 11 or recent Ubuntu.

You will need **rust**, **build-essential** and **git** installed to follow these instructions.

## Building Hermes

Install Prerequisites: Rust and Cargo
```bash
sudo apt update
sudo apt install build-essential
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

For preparation, we will create a dedicated user to run Hermes. Following command will also create home directory for the new user.

```bash
sudo useradd -m -d /srv/hermes hermes
sudo sudo -u hermes -s
```

Now is time install Hermes. Hermes is packaged in the ibc-relayer-cli Rust crate. To install the last release, run this command

```bash
cargo install ibc-relayer-cli --bin hermes --locked
```
This will download and build the crate ibc-relayer-cli, and install the hermes binary in /srv/hermes/.cargo/bin/hermes

You can check the hermes version:

```bash
hermes version
hermes 1.3.0
```

## Configuring Hermes with auto-config

{% hint style="info" %}
Thanks to the new functionalities of auto-configuration implemented through chain-registry, we start setting up the config file for the wallets.
{% endhint %}

As first step, we create the .hermes folder, where we would save the config.toml file of hermes configuration, useful in future steps for manual tweaking (like changing RPC and gRPC nodes).

```bash
mkdir .hermes
```

Then, we rely on the auto-config of hermes to create the config.toml file. Auto-config will create for us the standard hermes configuration, the chain specific configuration and the channel configuration, all taken from chain-registry (https://github.com/cosmos/chain-registry).

Let's consider that we want to create a JUNO<->OSMOSIS relayer. With 2 chains we can use this command:

```bash
 hermes config auto --output .hermes/config.toml --chains juno:juno-relayer --chains osmosis:osmo-relayer
```

With that configuration you are going to tell Hermes that for chain JUNO (that in chain-registry will be resolved in juno-1) you are using a wallet called juno-relayer (not imported yet), and with OSMOSIS (osmosis-1) you are using a wallet called osmo-relayer.


















## Setting up wallets


We setup the wallets by creating key configuration files that are imported to hermes. Here we go trhough Juno key setting, other chains are similar.

```bash
{
  "name":"juno-relayer",
  "type":"local",
  "address":"junoxxx",
  "pubkey":"{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"xxx\"}",
  "mnemonic": "24 words seed"
}
```


## Configuring Hermes

Choose your favourite editor and edit the following configuration template to mach your setup. There are features like telemetry and rest API that you can enable, but they are not necessary, so they are left out from this tutorial.

```
[global]
strategy = 'packets'
filter = true
log_level = 'info'
clear_packets_interval = 100

#
# Chain configuration Juno
#

[[chains]]
id = 'juno-1'
rpc_addr = 'http://127.0.0.1:26657'
grpc_addr = 'http://127.0.0.1:29090'
websocket_addr = 'ws://127.0.0.1:26657/websocket'

rpc_timeout = '20s'
account_prefix = 'juno'
key_name = 'juno-relayer'
store_prefix = 'ibc'
max_msg_num=15
max_gas = 1000000
gas_price = { price = 0.001, denom = 'ujuno' }
clock_drift = '5s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3'}

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-0'],
  ['transfer', 'channel-1'],
  ['transfer', 'channel-5'],
]


#
# Chain configureation Osmosis
#

[[chains]]
id = 'osmosis-1'

# API access to Osmosis node with indexing
rpc_addr = 'http://127.0.0.1:36657'
grpc_addr = 'http://127.0.0.1:39090'
websocket_addr = 'ws://127.0.0.1:36657/websocket'

rpc_timeout = '20s'
account_prefix = 'osmo'
key_name = 'osmo-relayer'
store_prefix = 'ibc'
max_gas =  1000000
gas_price = { price = 0.000, denom = 'uosmo' }
clock_drift = '5s'
trusting_period = '7days'
trust_threshold = { numerator = '1', denominator = '3' }

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-42'],
]

#
# Chain configuration Cosmos
#

[[chains]]
id = 'cosmoshub-4'

# API access to Cosmos node with indexing
rpc_addr = 'http://127.0.0.1:46657'
grpc_addr = 'http://127.0.0.1:49090'
websocket_addr = 'ws://127.0.0.1:46657/websocket'

rpc_timeout = '20s'
account_prefix = 'cosmos'
key_name = 'cosmos-relayer'
store_prefix = 'ibc'
max_msg_num=15
max_gas = 1000000
gas_price = { price = 0.0001, denom = 'uatom' }
clock_drift = '5s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-207'],
]

#
# Chain configuration Sifchain
#

[[chains]]
id = 'sifchain-1'

# API access to Cosmos node with indexing
rpc_addr = 'http://127.0.0.1:56657'
grpc_addr = 'http://127.0.0.1:59090'
websocket_addr = 'ws://127.0.0.1:56657/websocket'

rpc_timeout = '20s'
account_prefix = 'sif'
key_name = 'sif-relayer'
store_prefix = 'ibc'
max_msg_num=15
max_gas = 10000000
gas_price = { price = 0.001, denom = 'rowan' }
clock_drift = '5s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-14'],
]
```

You can validate the configuration with following:

```bash
hermes@Demo:~$ bin/hermes -c .hermes/config.toml  config validate
Success: "validation passed successfully"
```

Next we will import this key configuration to hermes and shred the used json file. (Using chain\_id **juno-1**.)

```bash
bin/hermes keys add juno-1 -f ./seed-juno.json
shred -u ./seed-juno.json
```

If you want to make sure the keys got imported, you can check them with following command (smart thing to run it before shredding the json file):

```bash
bin/hermes keys list juno-1
```

## Testing the setup

Let's do a quick test to see things work properly.

```bash
bin/hermes start
```

Once we see things load up correctly and there are no fatal errors, we can break out of hermes with **ctrl-c**.

## Configuring systemd

Now we will setup hermes to be run by systemd, and to start automatically on reboots.

Create the following configuration to **/etc/systemd/system/hermes.service**

```
[Unit]
Description=Hermes IBC relayer
ConditionPathExists=/srv/hermes/hermes
After=network.target juno.service cosmos.service osmo.service

[Service]
Type=simple
User=hermes
WorkingDirectory=/srv/hermes
ExecStart=/srv/hermes/hermes start
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Then we well start hermes with the newly created service and enable it. Note that this step is done from your normal user account that has sudo privileges, so no longer as hermes.

```bash
sudo systemctl start hermes.service
sudo systemctl enable hermes.service
```
