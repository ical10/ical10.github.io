---
title: Substrate-based EVM Parachain Deployment Setup
description: "a guide on how to deploy an EVM-based parachain using Substrate"
date: 2025-02-28
image: "/images/posts/evm-parachain-illustration.png"
categories: ["substrate"]
authors: ["Husni"]
tags: ["substrate", "parachain", "evm"]
draft: false
---

## Run A Collator Node
### Initial Set-up​
#### Requirements​

The most common way for a beginner to run a validator is on a cloud server running Linux. You may choose whatever VPS provider that you prefer. As OS it is best to use a recent Debian Linux. For this guide we will be using Ubuntu 22.04, but the instructions should be similar for other platforms.

#### Reference Hardware​

The transaction weights in Polkadot are benchmarked on reference hardware. We ran the benchmark on VM instances of two major cloud providers: Google Cloud Platform (GCP) and Amazon Web Services (AWS). To be specific, we used n2-standard-8 VM instance on GCP and c6i.4xlarge on AWS. It is recommended that the hardware used to run the validators at least matches the specs of the reference hardware in order to ensure they are able to process all blocks in time. If you use subpar hardware you will possibly run into performance issues, get less era points, and potentially even get slashed.
* CPU
    * x86-64 compatible;
    * Intel Ice Lake, or newer (Xeon or Core series); AMD Zen3, or newer (EPYC or Ryzen);
    * 8 physical cores @ 3.4GHz; starting with January 2025, the recommendation is to use a hardware with at least 8 physical cores, see referenda for more details about the rationale;
    * Simultaneous multithreading disabled (Hyper-Threading on Intel, SMT on AMD);
    * Prefer single-threaded performance over higher cores count. A comparison of single-threaded performance can be found here.
* Storage
    * An NVMe SSD of 1 TB (As it should be reasonably sized to deal with blockchain growth). An estimation of current chain snapshot sizes can be found here. In general, the latency is more important than the throughput.
* Memory
    * 32 GB DDR4 ECC.
* System
    * Linux Kernel 5.16 or newer.
    * Ubuntu 22.04 or newer (Recommended).
* Network
    * The minimum symmetric networking speed is set to 500 Mbit/s (= 62.5 MB/s). This is required to support a large number of parachains and allow for proper congestion control in busy network situations.

The specs listed above are not a hard requirement to run a validator, but are considered best practice. Running a validator is a responsible task; using professional hardware is a must in any way.

#### Install and Configure Network Time Protocol Client​

NTP is a networking protocol designed to synchronize the clocks of computers over a network. NTP allows you to synchronize the clocks of all the systems within the network. Currently it is required that validators' local clocks stay reasonably in sync, so you should be running NTP or a similar service. You can check whether you have the NTP client by running:
```
timedatectl
``` 

> <font color="#f00">Caution</font>
> <font color="#f00">If you are using Ubuntu 18.04 or a newer version, NTP Client should be installed by default.</font>

If NTP is installed and running, you should see System clock synchronized: yes (or a similar message). If you do not see it, you can install it by executing:

```
sudo apt-get install ntp
```

ntpd will be started automatically after install. You can query ntpd for status information to verify that everything is working:
```
sudo ntpq -p
```
> <font color="#F7A004">Warning</font>
> <font color="#F7A004">Skipping this can result in the validator node missing block authorship opportunities. If the clock is out of sync (even by a small amount), the blocks the validator produces may not get accepted by the network.
</font>

#### Make Sure Landlock is Enabled​
>Landlock is a Linux security feature used in Polkadot:
> Landlock empowers any process, including unprivileged ones, to securely restrict themselves.

To make use of landlock, make sure you are on the reference kernel version or newer. Most Linux distributions should already have landlock enabled, but you can check by running the following as root:
`dmesg | grep landlock || journalctl -kg landlock`

If it is not enabled, please see the official docs ("Kernel support") if you would like to build Linux with landlock enabled.

#### Secure-Validator Mode
The binary has a Secure-Validator Mode, enabling several protections for keeping keys secure. The protections include highly strict filesystem, networking, and process sandboxing on top of the existing wasmtime sandbox.

This mode is activated by default if the machine meets the following requirements. If not, there is an error message with instructions on disabling Secure-Validator Mode, though this is not recommended due to the security risks involved.

**Requirements**
- **Linux on x86-64 family** (usually Intel or AMD).
- **seccomp enabled**. You can check that this is the case by running the following command:
```
cat /boot/config-`uname -r` | grep CONFIG_SECCOMP=
```

The expected output, if enabled, is:

```
CONFIG_SECCOMP=y
```

- **OPTIONAL: Linux 5.13**. Provides access to even more strict filesystem protections.

#### Linux Best Practices
- Never use the root user.
- Always update the security patches for your OS.
- Enable and set up a firewall.
- Never allow password-based SSH, only use key-based access.
- Disable non-essential SSH subsystems (banner, motd, scp, X11 forwarding) and harden your SSH configuration (reasonable guide to begin with).
- Back up your storage regularly.

### Running the Substrate Parachain Binaries
#### Prerequisites: Install Rust and Dependencies​
If you have never installed Rust, you should do this first.

```
sudo apt install --assume-yes git clang curl libssl-dev protobuf-compiler
```

Then, run this command to fetch the latest version of Rust and install it.

```
curl https://sh.rustup.rs -sSf | sh -s -- -y
```

If you have already installed Rust, run the following command to make sure you are using the latest version.

```
rustup update
```

If not, this command will fetch the latest version of Rust and install it.


> <font color="#1936C9">note</font>
> <font color="#1936C9">If you do not have "curl" installed, run:</font>
> `sudo apt install curl`

It will also be valuable to have "websocat" (Netcat, curl and socat for WebSockets) installed for RPC interactions. Installation instructions for various operating systems can be found [here](https://github.com/vi/websocat#installation).

To configure your shell, run the following command.
`source $HOME/.cargo/env`

Verify your installation.
`rustc --version`

Finally, run this command to install the necessary dependencies for compiling and running the Polkadot node software.
`sudo apt install make clang pkg-config libssl-dev build-essential`

> <font color="#1936C9">note</font>
> <font color="#1936C9">If you are using OSX and you have Homebrew installed, you can issue the following equivalent command INSTEAD of the previous one:</font>
`brew install cmake pkg-config openssl git llvm`

#### Building the Binaries
You can build the collator binaries from Substrate EVM Template made by OpenZeppelin. For this example, we will use [Polkadot Runtime Templates](https://github.com/OpenZeppelin/polkadot-runtime-templates/tree/v1.0.0) to build the parachain binary.

Run:
```
cargo build --release --features="async-backing"
```

> <font color="#F7A004">warning</font>
> <font color="#F7A004">Run the command above only once! Always check first whether the binaries are already available or not in `/target/release`. If they are available, you don't need to build them again.</font>

After compiling the binaries, you can verify the installation by going to `target/release` and running:
```
./evm-template-node --version
```

It should return something like this:

```
evm-template-node 3.0.0-591b4584db4
```

#### Reserve a ParaId

:::warning
In order to reserve a ParaId for an EVM-based parachain, you will need to coordinate with the Sudo key holder on Substrate Relay. The steps below can only be done by the holder.
:::
    
1. Connect to one of the validator node via [Polkadot.js Apps](https://polkadot.js.org/apps/). Check the ws endpoint on the relay chain side.
2. Go to `Network` > `Parachains`
3. Go to `Parathreads` tab
4. Click the `+ ParaId` button
5. Take note of the value in `parachain id` field (normally it is `2000` if it is your first ParaId)
6. Click `Submit` and `Sign and Submit` (use `Niskala Sudo Account` to sign)
7. Go to `Network` > `Explorer`
8. You will see a `registrar.Reserved` event on `recent events` tab on the right. This means you have successfully reserved a ParaId.

#### Create a Sudo Key Using Metamask

Since we are deploying an EVM-based chain, we need to create a sudo key that is compatible with such chain. To create this key, we are going to use Metamask. 

Simply follow the guide here: https://support.metamask.io/configure/accounts/how-to-add-accounts-in-your-wallet/

Once done, note the public address as we will need it later on.

#### Preparing Necessary Files to Become a Parachain

First, generate a plain chainspec from parachain binary:
```
./target/release/evm-template-node build-spec --disable-default-bootnode --dev > plain-parachain-chainspec.json
```

Edit the chainspec:
1. Update `name` to `EVM Parachain`,
2. `id` to `evm_parachain`,
3. `chainType` to `Live`,
4. `protocolId` to `evm-parachain`;
5. `forkId` to `evm-parachain`;
6. Make sure that `telemetryEndpoints` is an empty array;
7. `tokenSymbol` to `UNIT`;
8. Change `para_id` and `parachainInfo.parachainId` from `1000` to the previously saved `parachain id` (probably `2000` if it is your first time).
9. Edit `sudo.key` with the Sudo key you have created earlier.
10. Edit `evmChainId.chainId` to same value as in no.5

Generate a raw chainspec:

```
./target/release/evm-template-node build-spec --chain plain-parachain-chainspec.json --disable-default-bootnode --raw > raw-parachain-chainspec.json
```

Generate the genesis state:

```
./target/release/evm-template-node export-genesis-state --chain raw-parachain-chainspec.json > genesis-state
```

Generate the validation code:

```
./target/release/evm-template-node export-genesis-wasm --chain raw-parachain-chainspec.json > genesis-wasm
```

#### Registering the parachain
You need access again to the Sudo key on the Substrate Relay to register the parachain. The steps below can only be done using this key. 

1. Go back to polkadot.js.org/apps. Go to `Developer` -> `Sudo`.
2. Select `parasSudoWrapper` -> `sudoScheduleParaInitialize(id, genesis)`.
3. Enter the `reserved id (2000)` to `id` field.
4. Enable file upload for both `genesisHead` and `validationCode`, because you will upload files for these.
5. Select `Yes` for `paraKind`. This means we register our parachain as a parachain, not as parathread.
6. Click and upload your `genesis-state` file generated earlier into `genesisHead` field.
7. Click and upload your `genesis-wasm` file generated earlier into `validationCode` field.
8. It should look like below:
![register-parachain](https://hackmd.io/_uploads/Bks5g48Oyl.png)
9. Submit Sudo

Once the transaction is submitted, the parachain will start the onboarding process. You can see it on Polkadot.js apps under `Network` -> `Parachains` -> `Parathreads` tab. The onboarding process should only take several minutes. While waiting the parachain to be onboarded, you can continue with the next steps.

#### Starting Collator Node

Copy the raw chainspec file from Substrate Relay. This file should be named `raw-relay-chainspec.json` or something similar on another handbook. It should be the exact same file used by validators to run their nodes.

With all of the necessary files in place, you can start running a collator node.

:::danger
be sure to clear your storage if you were running a node before
```
./target/release/evm-template-node purge-chain --chain raw-parachain-chainspec.json --base-path {insert-your-base-path-here}
```
:::

```
./target/release/evm-template-node \
    --alice \
    --collator \
    --force-authoring \
    --chain raw-parachain-chainspec.json \
    --base-path /tmp/collator01 \
    --port 40333 \
    --rpc-port 8844 \
    -- \
    --execution wasm \
    --chain raw-relay-chainspec.json \
    --port 30343 \
    --rpc-port 9977 \
    --sync warp
```

Your node should be running without any problem. From the logs, you can see that it should peer with Substrate Relay validators, and no pair on the parachain side (because it is only a single collator running). Note as well the Node ID (something similar like `12D3KooWBKH2m8Lycq7CSKoWWzwBrKk7eJ9DfKvFyP4Sm26QQNj1`).

If you encounter an error similar to this:
```
Error: NetworkKeyNotFound("/tmp/collator01/chains/evm-parachain/network/secret_ed25519")
```

Then you will need to create a network key first for the collator.

Simply run:
```
./target/release/evm-template-node key generate-node-key --chain raw-parachain-chainspec.json --base-path /tmp/collator01
```

It will output something like this:
```
Generating key in "/tmp/collator01/chains/evm-parachain/network/secret_ed25519"
12D3KooWFizyfMcXZvpCamxo9GJ262U8PQrHD2Sck96Uv374kvaN
```
which means the node key is successfully generated.

Then rerun the command to run the collator.

Your collator should not start producing blocks yet. You need **at least two collators to start producing blocks and coming into consensus**, so you will need to deploy another collator and sync it to the network.

#### Connecting Another Collator Node to the Network
The commands to run another collator node is basically similar to the first one, with a slight modification.

You can easily run:
```
./target/release/evm-template-node \
    --bob \
    --collator \
    --force-authoring \
    --chain raw-parachain-chainspec.json \
    --base-path /tmp/collator02 \
    --port 40444 \
    --rpc-port 8845 --bootnodes /ip4/127.0.0.1/tcp/40333/p2p/12D3KooWBKH2m8Lycq7CSKoWWzwBrKk7eJ9DfKvFyP4Sm26QQNj1 \
    -- \
    --execution wasm \
    --chain raw-relay-chainspec.json \
    --port 30343 \
    --rpc-port 9977 \
    --sync warp
```

Once it starts running, you should see the new collator peering with other nodes within the network, and you should see block production in your node terminal. Block finalisation should follow several moments later.

#### Setting Parachain into a Permanent Slot

Once the parachain is registered, the onboarding process will commence. Before the onboarding process is done, we need to assign the `paraId` into a permanent slot. To do that, simply do:
- `Developer` -> `Sudo`
- `assignedSlots` -> `assignedPermParachainSlot(id)`
- Enter the `paraId`, `Submit` and sign with Sudo key
- We will see that once the parachain is onboarded, the slot is valid for 365 days. Check it in: `assignedSlots` -> `permanentSlots`. Enter `paraId` in `Option<u32>` field. The output will something similar like this:
```
[
  6
  365
]
```

where `6` is the start of lease period, and `365` is the duration of the slot.
