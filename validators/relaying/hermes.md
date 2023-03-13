---
description: Instructions for setting up the rust based relayer, Hermes
cover: >-
  ../../.gitbook/assets/Gitbook Banner large 6 (1) (1) (1) (1) (1) (1) (1) (1)
  (1) (1) (1) (9).png
coverY: 0
---

# Hermes

## Assumptions

This guide is and example of configuration for Juno<->Osmosis relaying.
We assume that you already have access to Juno and Osmosis nodes. These can be either local nodes, or you can access them over the network. However, for networked version, you will need to adjust the systemd configuration not to depend on the chains that are run on other servers. And naturally the hermes configuration needs to adjust the addressing of each chain as well.

In these instructions, Hermes is installed under /srv/hermes, adjust the paths according to your setup.

These instructions are based on installation on Ubuntu 20.04 LTS, but should work the same on Ubuntu 22.04 or recent Debian.

You will need **rust**, **build-essential** and **git** installed to follow these instructions.

## Prerequisites

Install Prerequisites: Rust and Cargo
```bash
sudo apt update
sudo apt install build-essential
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## Building Hermes

For preparation, we will create a dedicated user to run Hermes. Following command will also create home directory for the new user.

```bash
sudo useradd -m -d /srv/hermes hermes
sudo sudo -u hermes -s
```

Now it's time to install Hermes. Hermes is packaged in the ibc-relayer-cli Rust crate. To install the last release, run this command

```bash
cargo install ibc-relayer-cli --bin hermes --locked
```

This will download and build the crate ibc-relayer-cli, and install the hermes binary in /srv/hermes/.cargo/bin/hermes and update the PATH

You can check the hermes version:

```bash
hermes version
hermes 1.3.0
```

## Configuring Hermes with auto-config

{% hint style="info" %}
Thanks to the new functionality of auto-configuration implemented through chain-registry, the initial setup will be a lot easyer than with older releases.
{% endhint %}

As first step, we create the .hermes folder, where we would save the config.toml file of hermes configuration, useful in future steps for manual tweaking (like changing RPC and gRPC nodes).

```bash
mkdir $HOME/.hermes
```

Then, we rely on the auto-config of hermes to create the config.toml file. Auto-config will create for us the standard hermes configuration, the chains specific configurations and the channels configurations, all taken from chain-registry (https://github.com/cosmos/chain-registry).

As stated on the beginning, we want to create a Juno<->Osmosis relayer. With 2 chains we can use this command:

```bash
hermes config auto --output $HOME/.hermes/config.toml --chains juno:juno-relayer --chains osmosis:osmo-relayer
```

With more chains, just append to the command others ```--chains chain:wallet-name``` parameters.

With that command you had configured Hermes for Juno (that in chain-registry will be resolved in juno-1) with a wallet called juno-relayer (not imported yet, this is only a label that must be the same of the future imported wallet), and with Osmosis (osmosis-1) with a wallet called osmo-relayer.

Doing so the hermes binary will populate for us the $HOME/.hermes/config.toml file with something like this (at the time of writing):

```bash
[global]
log_level = 'info'
[mode.clients]
enabled = true
refresh = true
misbehaviour = false

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
tx_confirmation = false
auto_register_counterparty_payee = false

[rest]
enabled = false
host = '127.0.0.1'
port = 3000

[telemetry]
enabled = false
host = '127.0.0.1'
port = 3001

[[chains]]
id = 'juno-1'
type = 'CosmosSdk'
rpc_addr = 'https://rpc-juno-ia.cosmosia.notional.ventures/'
websocket_addr = 'wss://rpc-juno-ia.cosmosia.notional.ventures/websocket'
grpc_addr = 'https://grpc-juno-ia.cosmosia.notional.ventures/'
rpc_timeout = '10s'
account_prefix = 'juno'
key_name = 'juno-relayer'
key_store_type = 'Test'
store_prefix = 'ibc'
default_gas = 100000
max_gas = 400000
gas_multiplier = 1.1
max_msg_num = 30
max_tx_size = 180000
clock_drift = '5s'
max_block_time = '30s'
memo_prefix = ''
sequential_batch_tx = false

[chains.trust_threshold]
numerator = '1'
denominator = '3'

[chains.gas_price]
price = 0.1
denom = 'ujuno'

[chains.packet_filter]
policy = 'allow'
list = [
    [
    'transfer',
    'channel-0',
],
    [
    'wasm.juno1v4887y83d6g28puzvt8cl0f3cdhd3y6y9mpysnsp3k8krdm7l6jqgm0rkn',
    'channel-47',
],
]

[chains.address_type]
derivation = 'cosmos'

[[chains]]
id = 'osmosis-1'
type = 'CosmosSdk'
rpc_addr = 'https://rpc-osmosis-ia.cosmosia.notional.ventures/'
websocket_addr = 'wss://rpc-osmosis-ia.cosmosia.notional.ventures/websocket'
grpc_addr = 'https://grpc-osmosis-ia.cosmosia.notional.ventures/'
rpc_timeout = '10s'
account_prefix = 'osmo'
key_name = 'osmo-relayer'
key_store_type = 'Test'
store_prefix = 'ibc'
default_gas = 100000
max_gas = 400000
gas_multiplier = 1.1
max_msg_num = 30
max_tx_size = 180000
clock_drift = '5s'
max_block_time = '30s'
memo_prefix = ''
sequential_batch_tx = false

[chains.trust_threshold]
numerator = '1'
denominator = '3'

[chains.gas_price]
price = 0.1
denom = 'uosmo'

[chains.packet_filter]
policy = 'allow'
list = [
    [
    'transfer',
    'channel-42',
],
    [
    'transfer',
    'channel-169',
],
]

[chains.address_type]
derivation = 'cosmos'
```

As you can see, hermes retrieved the chains configurations, and it has populated also the channels between these 2 chains.

{% hint style="warning" %}
BEWARE: Every time you run the ```hermes config auto``` command it will WIPE the config file in input and rewrite it with chain-registry data. This mean that you will lose any changes, like REST or Telemetry non standard configuration, RPCs and gRPCs configuration (if you are using your owns or trust some specific public RPCs/gRPCs) and every other parameter changed on the chains.
The positive things is that it will update also the channels configuration, so if there had been channels changing, you will get the new active channels (if someone had updated chain-registry).
Except for the initial config, where you have nothing to wipe, use it carefully. 
{% endhint %}

## Setting up wallets

We setup the wallets by creating key configuration files that will be imported to hermes. Here we go through Juno key setting, other chains are similar.

```bash
{
  "name":"juno-relayer",
  "type":"local",
  "address":"junoxxx",
  "pubkey":"{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"xxx\"}",
  "mnemonic": "24 words seed"
}
```

You can manually populate the values once the wallet had been created from Keplr for example (and token received for public key to be visible in explorers). 
If you have your own CLI configured for the relative chain, you can use this command to have the json output you need (for example with juno):

```bash
junod keys add juno-relayer --output json
```

Assuming that you have saved the juno-relayer wallet information in juno.json file, you can run this command to add it to hermes

```bash
hermes keys add --key-name juno-relayer --chain juno-1 --key-file ./juno.json
```

If you want to make sure the keys got imported, you can check them with following command

```bash
bin/hermes keys list juno-1
```

If all is fine then shred it

```bash
shred -u ./juno.json
```

You must import a wallet for every chain you relay.

## Validate the config

You can validate the configuration with following:

```bash
hermes -c $HOME/.hermes/config.toml  config validate
Success: "validation passed successfully"
```

Then you can also run a health-check

```bash
hermes health-check

INFO ThreadId(01) using default configuration from '/srv/hermes/.hermes/config.toml'
INFO ThreadId(01) running Hermes v1.3.0
INFO ThreadId(01) health_check{chain=juno-1}: performing health check...
WARN ThreadId(05) health_check{chain=juno-1}: Chain 'juno-1' has no minimum gas price value configured for denomination 'ujuno'. This is usually a sign of misconfiguration, please check your config.toml
INFO ThreadId(01) health_check{chain=juno-1}: chain is healthy
INFO ThreadId(01) health_check{chain=osmosis-1}: performing health check...
WARN ThreadId(11) health_check{chain=osmosis-1}: Chain 'osmosis-1' has no minimum gas price value configured for denomination 'uosmo'. This is usually a sign of misconfiguration, please check your config.toml
INFO ThreadId(01) health_check{chain=osmosis-1}: chain is healthy
SUCCESS performed health check for all chains in the config
```

## Testing the setup

Let's do a quick test to see things work properly.

```bash
hermes start
```

Once we see things load up correctly and there are no fatal errors, we can break out of hermes with **ctrl-c**.

## Configuring systemd

Now we will setup hermes to be run by systemd, and to start automatically on reboots.

Create the following configuration to **/etc/systemd/system/hermes.service**

```
[Unit]
Description=Hermes IBC relayer
ConditionPathExists=/srv/hermes/hermes
#enable the line below if you are using LOCAL nodes
#After=network.target juno.service cosmos.service osmo.service

[Service]
Type=simple
User=hermes
WorkingDirectory=/srv/hermes
ExecStart=$(which hermes) start
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Then we will start hermes with the newly created service and enable it. Note that this step is done from your normal user account that has sudo privileges, so no longer as hermes.

```bash
sudo systemctl start hermes.service
sudo systemctl enable hermes.service
```
