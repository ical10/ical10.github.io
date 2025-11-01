---
title: Substrate-based Relay Chain Deployment Setup
description: "meta description"
date: 2025-02-28
image: "/images/posts/substrate-illustration.png"
categories: ["substrate"]
authors: ["Husni"]
tags: ["substrate", "relay chain"]
draft: false
---

## Run A Validator Node
### Initial Set-up
#### Requirements

The most common way for a beginner to run a validator is on a cloud server running Linux. You may choose whatever VPS provider that you prefer. As OS it is best to use a recent Debian Linux. For this guide we will be using Ubuntu 22.04, but the instructions should be similar for other platforms.

#### Reference Hardware

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
* Network
    * The minimum symmetric networking speed is set to 500 Mbit/s (= 62.5 MB/s). This is required to support a large number of parachains and allow for proper congestion control in busy network situations.

The specs listed above are not a hard requirement to run a validator, but are considered best practice. Running a validator is a responsible task; using professional hardware is a must in any way.

#### Install and Configure Network Time Protocol Client

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


### Running the Substrate Relay Chain Binaries
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
You can build the validator binaries from the fork of polkadot-sdk. We can use [polkadot-stable2407](https://github.com/paritytech/polkadot-sdk/tree/polkadot-stable2407) branch as an example. This branch should be the most stable version of polkadot-sdk.

First, ensure that `wasm32-unknown-unknown` is added as target in `rustup`.

In x86_64:
```
rustup target add wasm32-unknown-unknown --toolchain stable-x86_64-unknown-linux-gnu
rustup component add rust-src --toolchain stable-x86_64-unknown-linux-gnu
```

In arm64:
```
rustup target add wasm32-unknown-unknown --toolchain stable-aarch64-unknown-linux-gnu
rustup component add rust-src --toolchain stable-aarch64-unknown-linux-gnu

```

Now you can build the binaries. Go to the root of the repo, and run:
`cargo build --release`

> <font color="#F7A004">warning</font>
> <font color="#F7A004">Run the command above only once! Always check first whether the binaries are already available or not in `/target/release`. If they are available, you don't need to build them again.</font>

After compiling the binaries, you can verify the installation by going to `target/release` and running:
```
./polkadot --version
./polkadot-prepare-worker --version
./polkadot-execute-worker --version
```

It should return something like this (the version doesn't matter but all three versions have to match):

```
polkadot 1.15.1-16b0fd09d9e
polkadot 1.15.1
polkadot 1.15.1
```

#### Generate a Sudo Key

You will need to also create a Sudo key, which will have Root privileges on the relay chain. This will also avoid using predefined accounts and minimizing attacks on the relay network.

Use `key` command to generate the `Sudo` key:
```
./target/release/polkadot key generate --scheme Sr25519 
```

The command displays output similar to the following:
```
Secret phrase:       fresh gauge diet release catch egg method such process ancient erupt account
  Network ID:        substrate
  Secret seed:       0xc8d50f6db37f82ed6b0b607e9bf0d8d3dbce139b388e3407cd27c090613d1d04
  Public key (hex):  0x30d44a91f42a34ebc7c10b214a7d3bd54c99275826ea31491c94eea579a3a35e
  Account ID:        0x30d44a91f42a34ebc7c10b214a7d3bd54c99275826ea31491c94eea579a3a35e
  Public key (SS58): 5DAjDg22ErYgwqSdHtpL8cLTf4Bm7hVqsfXqwzS11geN48dT
  SS58 Address:      5DAjDg22ErYgwqSdHtpL8cLTf4Bm7hVqsfXqwzS11geN48dT
```

Store the `Secret phrase` and `secret seed` in a secure place. Take note of the ss58 public key as you will need it later.

#### Generate a Network Key

We will create a network key for `Alice`. The same steps should work for other validators. Assuming that the base path for `Alice` is at `/tmp/alice`, we can simply generate a network key by running:

```
 ./target/release/polkadot key generate-node-key --base-path /tmp/alice
```

The output will be similar as follows:
```
Generating key in "/tmp/alice/chains/polkadot/network/secret_ed25519"
12D3KooWCMbjHYtXxMJG82uZ9iPkohwDQuvGEbnjMRJViAZvaDVc
```

Remember the path above as we will need it later.

#### Create a Custom Chain Specification 

After you generate the keys to use with your blockchain, you are ready to create a custom chain specification using those key pairs then share your custom chain specification with other validators.

To enable others to participate in your blockchain network, ensure that they generate their own keys. After you collect the keys for network participants, you can create a custom chain specification to replace the local chain specification.

For simplicity, the custom chain specification you create in this tutorial is a modified version of the `rococo-local` chain specification that illustrates how to create a two-node network. You can follow the same steps to add more nodes to the network if you have the required keys. 

Example files has already been provided in `plain-relay-chainspec.json` and `raw-relay-chainspec.json` in the current branch. We are going to go through the steps on how to reproduce these chainspec files.

##### Modify the Local Chain Specification

To create a new chain specification based on the local specification:

1. Make sure you're still in the root directory.
2. Export the `rococo-local` chain specification to a file named `plain-relay-chainspec.json` by running the following command:
`./target/release/polkadot build-spec --disable-default-bootnode --chain rococo-local > plain-relay-chainspec.json`
3. Open the `plain-relay-chainspec.json` file in a text editor.
4. Modify the `name` field to identify this chain specification as a custom chain specification.
`"name": "Substrate Relay"`
5. Modify the `id` field.
`"id": "relay_testnet"`
6. Modify the `protocolId` field.
`"protocolId": "relay_testnet"`
7. Modify the `sudo` field. Make sure that there is only one key value there.
```json
 "sudo": {
          "key": "5DAjDg22ErYgwqSdHtpL8cLTf4Bm7hVqsfXqwzS11geN48dT"
},
```
8. Save your changes and close the file.

#### Convert the Chain Specification to Raw Format

After you prepare a chain specification with the validator information, you must convert it into a raw specification format before it can be used. The raw chain specification includes the same information as the unconverted specification. However, the raw chain specification also contains encoded storage keys that the node uses to reference the data in its local storage. Distributing a raw chain specification ensures that each node stores the data using the proper storage keys.

To convert a chain specification to use the raw format:
1. Open a terminal shell on your computer.
2. Make sure you're still at the root directory.
3. Convert the `plain-relay-chainspec.json` chain specification to the raw format with the file name `raw-relay-chainspec.json` by running the following command:

```bash
./target/release/polkadot build-spec --chain plain-relay-chainspec.json --raw --disable-default-bootnode > raw-relay-chainspec.json
```

#### Prepare to launch the private network

After you distribute the custom chain specification to all network participants, you're ready to launch the Relay network. If you follow the steps in this tutorial, you'll be able to add multiple computers to your network.

To continue, verify the following:

- You have generated or collected the account keys for at least two authority accounts.
- You have generated network keys for all validators in the network.
- You have updated your custom chain specification to include the keys for block production (`aura`) and block finalization (`grandpa`).
- You have converted your custom chain specification to raw format and distributed the raw chain specification to the nodes participating in the private network.

If you have completed these steps, you are ready to start the first node in the relay chain.


#### Start the First Node

You are responsible for starting the first node, called the **bootnode**. For now, you are going to run the node using `--alice` account. New updates will be released on this handbook on how to use custom keys when running nodes.

To start the first node:
1. Make sure you're at the root of the project.
2. Start the first node using the custom chain specification by running a command similar to the following:

```bash
./target/release/polkadot \
    --base-path /tmp/alice \
    --chain raw-relay-chainspec.json \
    --alice \
    --port 30333 \
    --rpc-port 9945 \
    --prometheus-port 9615 \
    --prometheus-external \
    --force-authoring \
    --node-key 8eca754559aba9b2ae7167725bbeaac3f0f86fe8c37394a286946c0be91c9949 \
    --validator
```
3. We should provide value for `--node-key` option. That value is the private key of corresponding network key, which can be found in `/tmp/alice/chains/polkadot/network/secret_ed25519`. <font color="#f00">Beware! Do not share this private key as this could compromise the security of your node.</font>


Note the following command-line options you are using to start the node:

* The `--base-path` option specifies the directory for storing all of the data related to this chain.
* The `--chain command-line` option specifies the custom chain specification.
* The `--validator` command-line option indicates that this node is an authority for the chain.
* The `--node-key` option specifies the ed25519 private key of your node and is important so that your node can be discovered by other authorities on the network.

<!-- This command also starts the node using your own keys instead of a predefined account. Because you aren't using a predefined account with known keys, you'll need to add your keys to the keystore in a separate step. -->

#### View information about node operations

After you start the local node, information about the operations performed is displayed in the terminal shell. In that terminal, verify that you see output similar to the following:

```
2025-01-20 10:44:06 Parity Polkadot
2025-01-20 10:44:06 ✌️  version 1.16.0-87971b3e927
2025-01-20 10:44:06 ❤️  by Parity Technologies <admin@parity.io>, 2017-2025
2025-01-20 10:44:06 📋 Chain specification: Substrate Relay
2025-01-20 10:44:06 🏷  Node name: Alice
2025-01-20 10:44:06 👤 Role: AUTHORITY
2025-01-20 10:44:06 💾 Database: RocksDb at /tmp/alice/chains/relay_testnet/db/full
2025-01-20 10:44:09 🚀 Using prepare-worker binary at: "/Users/path-to/common-polkadot-sdk/target/release/polkadot-prepare-worker"
2025-01-20 10:44:09 🚀 Using execute-worker binary at: "/Users/path-to/common-polkadot-sdk/target/release/polkadot-execute-worker"
2025-01-20 10:44:09 🏷  Local node identity is: 12D3KooWCMbjHYtXxMJG82uZ9iPkohwDQuvGEbnjMRJViAZvaDVc
2025-01-20 10:44:09 Running libp2p network backend
2025-01-20 10:44:09 💻 Operating system: macos
2025-01-20 10:44:09 💻 CPU architecture: aarch64
2025-01-20 10:44:09 📦 Highest known block at #58
2025-01-20 10:44:09 〽️ Prometheus exporter started at 0.0.0.0:9615
2025-01-20 10:44:09 Running JSON-RPC server: addr=127.0.0.1:9945,[::1]:9945
2025-01-20 10:44:09 🏁 CPU single core score: 994.40 MiBs, parallelism score: 808.11 MiBs with expected cores: 8
2025-01-20 10:44:09 🏁 Memory score: 39.44 GiBs
2025-01-20 10:44:09 🏁 Disk score (seq. writes): 2.54 GiBs
2025-01-20 10:44:09 🏁 Disk score (rand. writes): 407.96 MiBs
2025-01-20 10:44:09 ⚠️  Starting January 2025 the hardware will fail the minimal physical CPU cores requirements Failed checks: BLAKE2-256(expected: 1000.00 MiBs, found: 994.40 MiBs), BLAKE2-256-Parallel-8(expected: 1000.00 MiBs, found: 808.11 MiBs), Rnd Write(expected: 420.00 MiBs, found: 407.96 MiBs),  for role 'Authority',
find out more when this will become mandatory at:
https://wiki.polkadot.network/docs/maintain-guides-how-to-validate-polkadot#reference-hardware
2025-01-20 10:44:09 ⚠️  The hardware does not meet the minimal requirements Failed checks: BLAKE2-256(expected: 1000.00 MiBs, found: 994.40 MiBs), BLAKE2-256-Parallel-8(expected: 1000.00 MiBs, found: 808.11 MiBs), Rnd Write(expected: 420.00 MiBs, found: 407.96 MiBs),  for role 'Authority' find out more at:
https://wiki.polkadot.network/docs/maintain-guides-how-to-validate-polkadot#reference-hardware
2025-01-20 10:22:42 Starting with an empty approval vote DB.
2025-01-20 10:22:42 👶 Starting BABE Authorship worker
2025-01-20 10:22:42 🥩 BEEFY gadget waiting for BEEFY pallet to become available...
2025-01-20 10:22:42 🚨 Some security issues have been detected.
Running validation of malicious PVF code has a higher risk of compromising this machine.
Secure mode is enabled only for Linux
and a full secure mode is enabled only for Linux x86-64.
2025-01-20 10:22:47 💤 Idle (0 peers), best: #0 (0x0dda…e35f), finalized #0 (0x0dda…e35f), ⬇ 0 ⬆ 0
2025-01-20 10:22:48 🙌 Starting consensus session on top of parent 0x0dda6b48aeff586562322b5a7cdda2bde525cedd36d0c642d40ba1659094e35f (#0)
2025-01-20 10:22:50 ParentBlockRandomness did not provide entropy
2025-01-20 10:22:50 ParentBlockRandomness did not provide entropy
2025-01-20 10:22:50 🎁 Prepared block for proposing at 1 (11 ms) [hash: 0x098007fa0554d55d4307a85fa42f1ca04318d06781ee7f9c5cbc5b4f392124a1; parent_hash: 0x0dda…e35f; extrinsics (2): [0xc8c1…1690, 0x659d…9732]
2025-01-20 10:22:50 🔖 Pre-sealed block for proposal at 1. Hash now 0xd8c7f9c3bd76ea7588070ae4c8115c7c1b3b5b9850a0bb8ccc44386880b6e51e, previously 0x098007fa0554d55d4307a85fa42f1ca04318d06781ee7f9c5cbc5b4f392124a1.
2025-01-20 10:22:50 👶 New epoch 0 launching at block 0xd8c7…e51e (block slot 289560828 >= start slot 289560828).
2025-01-20 10:22:50 👶 Next epoch starts at slot 289560838
2025-01-20 10:22:50 🏆 Imported #1 (0x0dda…e35f → 0xd8c7…e51e)
2025-01-20 10:22:52 💤 Idle (0 peers), best: #1 (0xd8c7…e51e), finalized #0 (0x0dda…e35f), ⬇ 0 ⬆ 0
2025-01-20 10:22:54 🙌 Starting consensus session on top of parent 0xd8c7f9c3bd76ea7588070ae4c8115c7c1b3b5b9850a0bb8ccc44386880b6e51e (#1)
2025-01-20 10:22:54 🎁 Prepared block for proposing at 2 (3 ms) [hash: 0xabfd15b1da6c353344fc76c79825686acd5c9d750ca0f5d547565055f20808e9; parent_hash: 0xd8c7…e51e; extrinsics (2): [0x1352…a888, 0x63a4…9d60]
```

Take note of the following information:

* The output indicates that the chain specification being used is the custom chain specification you created and specified using the `--chain` command-line option.
* The output indicates that the node is an authority because you started the node using the `--validator` command-line option.
* The output shows the genesis block being initialized with the block hash `🙌 Starting consensus session on top of parent 0x0dda6b48aeff586562322b5a7cdda2bde525cedd36d0c642d40ba1659094e35f (#0)`
* The output specifies the Local node identity for your node. In this example, the node identity is `12D3KooWCMbjHYtXxMJG82uZ9iPkohwDQuvGEbnjMRJViAZvaDVc`.
* The output specifies the IP address used for the node is the local host `127.0.0.1`.

These values are for this specific tutorial example. The values in your output will be specific to your node and you must provide the values for your node to other network participants to connect to the bootnode.

Now that you have successfully started a validator node using your own keys and taken note of the node identity, you can continue to the next step. Before you add your keys to the keystore, however, stop the node by pressing `Control-c`.

#### Enable other participants to join

You can now allow other validators to join the network using the `--bootnodes` and `--validator` command-line options. You will be using `--bob` account for this node. Remember, you will also need to generate a network key for `Bob`.

To add a second validator to the relay network:

1. Make sure you're still at the root directory.
2. Start a second blockchain node by running a command similar to the following:

```
./target/release/polkadot \
--base-path /tmp/bob \
--chain raw-relay-chainspec.json \
--bob \
--port 30444 \
--rpc-port 9956 \
--force-authoring \
--node-key a8a8ac174efe098f0e5ce7c7485ec934e1f2e39da1b06bd0752b8499b4f33afc \
--bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWCMbjHYtXxMJG82uZ9iPkohwDQuvGEbnjMRJViAZvaDVc \
--validator
```
    
This command uses the `base-path`, `name` and `validator` command-line options to identify this node as a second validator for the relay network. 
    
The `--chain` command-line option specifies the chain specification file to use. This file must be identical for all validators in the network. Be sure to set the correct information for the `--bootnodes` command-line option. In particular, be sure you have specified the local node identifier from the first node in the network. If you don't set the correct bootnode identifier, you see errors like this:

```
The bootnode you want to connect to at ... provided a different peer ID than the one you expect: ...
```
    
<!-- 3. Add the `aura` secret key generated from the `key` subcommand by running a command similar to the following:

```
./target/release/substrate-node key insert --base-path /tmp/niskala-node-2 \
  --chain customSpecRaw.json \
  --scheme Sr25519 \
  --suri <second-participant-secret-seed> \
  --password-interactive \
  --key-type aura
```
    
Replace <second-participant-secret-seed> with the secret phrase or secret seed that you generated in [Generate a Second Set of Keys](#Generate-a-Second-Set-of-Keys). The `aura` key type is required to enable block production.

4. Type the password you used to generate the keys.
5. Add the `grandpa` secret key generated from the `key` subcommand to the local keystore by running a command similar to the following:

```
./target/release/substrate-node key insert --base-path /tmp/niskala-node-2 \
  --chain customSpecRaw.json \
  --scheme Ed25519 --suri <second-participant-secret-seed> \
  --password-interactive \
  --key-type gran
```

Replace <second-participant-secret-seed> with the secret phrase or secret seed that you generated in Generate a second key pair. The gran key type is required to enable block finalization.

Block finalization requires at least two-thirds of the validators to add their keys to their respective keystores. Because this network is configured with two validators in the chain specification, block finalization can only start after the second node has added its keys.

6. Type the password you used to generate the keys.
7. Verify that your keys are in the keystore for `niskala-node-2` by running the following command:

```
ls /tmp/niskala-node-2/chains/local_testnet/keystore
```

The command displays output similar to the following:

```
617572610a6cadb3d6f55a121de4c89754dd835e634ae83249734dfad01c2fae7e9ac102
6772616e5f273f61a4910897cec969b598a70a832fb7894ad7c741e2a559617898426f20
```
    
Substrate nodes require a restart after inserting a `grandpa` key, so you must shut down and restart nodes before you see blocks being finalized.

8. Shut down the node by pressing Control-c.
10. Restart the second blockchain node by running the following command:

```
./target/release/substrate-node \
  --base-path /tmp/niskala-node-2 \
  --chain ./customSpecRaw.json \
  --port 30334 \
  --rpc-port 9946 \
  --validator \
  --rpc-methods Unsafe \
  --prometheus-port 9615 \
  --prometheus-external \ 
  --name Niskala-Node-2 \
  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWLmrYDLoNTyTYtRdDyZLWDe1paxzxTw5RgjmHLfzW96SX \
  --force-authoring \
  --password-interactive
```
    
After both nodes have added their keys to their respective keystores—located under `/tmp/niskala-node-1` and `/tmp/niskala-node-2` -and been restarted, you should see the same genesis block and state root hashes. -->

You should see that each node has one peer (`1 peers`), and they have produced a block proposal (`best: #2 (0xe111…c084)`). After a few seconds, you should see new blocks being finalized on both nodes. This is because the network requires **at least 2 validators to finalize blocks**.

#### Adding More Nodes with Custom Keys
You can add another validator node with its own keys. This process can be done even when the relay chain is still running and validating blocks.

First, generate a key using `key` subcommand. Simply run:
```
./target/release/polkadot key generate
```

will generate something similar like this:
```
Secret phrase:       snake decrease demand issue excuse crop wagon seven border fortune cement pass
  Network ID:        substrate
  Secret seed:       0xcf02d16d988b233d1267b059c885d0d347f74625b9d50272000aa819e7fa08e3
  Public key (hex):  0x2c2de1c38897e050f9e773925fe90a09d815d96ea127b561d2a7111f0b00b231
  Account ID:        0x2c2de1c38897e050f9e773925fe90a09d815d96ea127b561d2a7111f0b00b231
  Public key (SS58): 5D4dbPz1uvk7ExLGwac37UJcDLuqYbaLXKaekTPtPFZGqt4M
  SS58 Address:      5D4dbPz1uvk7ExLGwac37UJcDLuqYbaLXKaekTPtPFZGqt4M
```

Now you must insert this key into the keystore for `aura`, `grandpa`, and `beefy`. By assuming the base path is `/tmp/validator03`, for inserting `aura` key simply run:

```
./target/release/polkadot key insert \
    --base-path /tmp/validator03 \
    --chain raw-relay-chainspec.json \
    --scheme Sr25519 \
    --suri "snake decrease demand issue excuse crop wagon seven border fortune cement pass" \
    --key-type aura
```

for `grandpa`:
```
./target/release/polkadot key insert \
    --base-path /tmp/validator03 \
    --chain raw-relay-chainspec.json \
    --scheme Ed25519 \
    --suri "snake decrease demand issue excuse crop wagon seven border fortune cement pass" \
    --key-type gran
```

and for `beefy`:
```
./target/release/polkadot key insert \
    --base-path /tmp/validator03 \
    --chain raw-relay-chainspec.json \
    --scheme ecdsa \
    --suri "snake decrease demand issue excuse crop wagon seven border fortune cement pass" \
    --key-type beef
```

You also need to store network key and insert it into the keystore as well. Run:
```
 ./target/release/polkadot key generate-node-key --base-path /tmp/validator03 --chain raw-relay-chainspec.json
```

Run the node by using a similar command as the previous nodes:

```
./target/release/polkadot \
--base-path /tmp/validator03 \
--chain raw-relay-chainspec.json \
--name "validator-03" \
--port 30555 \
--rpc-port 9967 \
--force-authoring \
--bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWCMbjHYtXxMJG82uZ9iPkohwDQuvGEbnjMRJViAZvaDVc \
--validator
```

Now, you should add some balances to the new key. Simply go to `polkadot.js` apps and send some funds (500k tokens is enough) from an account (e.g. `Alice`) to the validator address (`5D4dbPz1uvk7ExLGwac37UJcDLuqYbaLXKaekTPtPFZGqt4M`). To make things simple for the steps, make sure that you are opening the apps with the correct RPC port (`9967`).

![Screenshot 2025-01-24 at 11.26.59](https://hackmd.io/_uploads/S1edjyZuyx.png)

Then you must now rotate session keys. 

- Simply go to `Developer` -> `RPC Calls`. 
- Then select `author` -> `rotateKeys`. 
- Click `Submit RPC call`. 
- Copy the generated public key.

![Screenshot 2025-01-24 at 11.42.03](https://hackmd.io/_uploads/ryx50JbOye.png)

Now you must load the secret seed (`"snake decrease demand issue excuse crop wagon seven border fortune cement pass"`) into a compatible wallet extension. 

- Then, go to `Developer` -> `Extrinsic` -> `Submission` tab.
- Select the new validator account in `using selected account` field.
- Select `session` -> `setKeys(keys, proof)`. 
- Insert the copied public key in `keys` field. Insert `0x00` in `proof` field.
- Submit transaction and sign it.

![Screenshot 2025-01-24 at 11.34.41](https://hackmd.io/_uploads/SyzOAJbdkl.png)


Then you must register the new validator using Sudo account.

- Go to `Developer` -> `Sudo`.
- Then select `validatorManager` -> `registerValidators(validators)`
- Select the new validator account in the field just below `validators`.
- Submit transaction and sign it.

![Screenshot 2025-01-24 at 11.38.50](https://hackmd.io/_uploads/ByWiRy-OJe.png)

After waiting for the next session to start, new validator will start producing blocks.

#### Running an Archive Node

When running as a simple sync node (above), only the state of the past 256 blocks will be kept. To support the full state, just add the `--pruning` flag:
```
./target/release/idchain-parachain --name "My node's name" --pruning archive
// No need to add --validator because it only store full state, not producing / validating blocks
...another commands needed to run the binary
```
It is possible to almost quadruple synchronization speed by using an additional flag: `--wasm-execution Compiled`. Note that this uses much more CPU and RAM, so it should be turned off after the node syncs.

    
#### Using systemd for a Validator Node
You can run your validator as a systemd process so that it will automatically restart on server reboots or crashes. This is important to maintain liveness of the network This is important to maintain liveness of the network.

Before following this guide you should have already set up your validator and make sure that the network is running as expected (validators producing blocks, every validators are sync'ed to each others, finalising blocks, etc.)

First create a new unit file called `relay-validator.service` in `/etc/systemd/system/`.

```
touch /etc/systemd/system/relay-validator.service
```

In this unit file you will write the commands that you want to run on server boot / restart.

```
[Unit]
Description=Substrate Relay Validator

[Service]
ExecStart=PATH_TO_RELAY_BIN --validator --name SHOW_ON_TELEMETRY
Restart=always
RestartSec=120

[Install]
WantedBy=multi-user.target
```

> <font color="#F7A004">warning</font>
> <font color="#F7A004"> It is recommended to delay the restart of a node with RestartSec in the case of node crashes. It's possible that when a node crashes, consensus votes in GRANDPA aren't persisted to disk. In this case, there is potential to equivocate when immediately restarting. What can happen is the node will not recognize votes that didn't make it to disk, and will then cast conflicting votes. Delaying the restart will allow the network to progress past potentially conflicting votes, at which point other nodes will not accept them. </font>

To enable this to autostart on bootup run:

```
systemctl enable relay-validator.service
```

Start it manually with:

```
systemctl start relay-validator.service
```
    
You can check that it's working with:

```
systemctl status relay-validator.service
```
    
You can tail the logs with `journalctl` like so:

```
journalctl -f -u relay-validator
```

### Monitor Your Node
This guide will walk you through how to set up Prometheus with Grafana to monitor your node using Ubuntu 18.04 or 20.04.

A Substrate-based chain exposes data such as the height of the chain, the number of connected peers to your node, CPU, memory usage of your machine, and more. To monitor this data, Prometheus is used to collect metrics and Grafana allows for displaying them on the dashboard.

#### Preparation
First, create a user for Prometheus by adding the `--no-create-home` flag to disallow `prometheus` from logging in.

```
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
```

Create the directories required to store the configuration and executable files.

```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

Change the ownership of these directories to prometheus so that only `prometheus` can access them.

```
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

#### Installing and Configuring Prometheus

After setting up the environment, update your OS, and install the latest Prometheus. You can check the latest release by going to their GitHub repository under the [releases](https://github.com/prometheus/prometheus/releases/) page.

```
sudo apt-get update && apt-get upgrade
wget https://github.com/prometheus/prometheus/releases/download/v2.26.0/prometheus-2.26.0.linux-amd64.tar.gz
tar xfz prometheus-*.tar.gz
cd prometheus-2.26.0.linux-amd64
```


The following two binaries are in the directory:

- prometheus - Prometheus main binary file
- promtool

The following two directories (which contain the web interface, configuration files examples and the license) are in the directory:

- consoles
- console_libraries

Copy the executable files to the `/usr/local/bin/` directory.

```
sudo cp ./prometheus /usr/local/bin/
sudo cp ./promtool /usr/local/bin/
```

Change the ownership of these files to the `prometheus` user.

```
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

Copy the consoles and console_libraries directories to `/etc/prometheus`

```
sudo cp -r ./consoles /etc/prometheus
sudo cp -r ./console_libraries /etc/prometheus
```

Change the ownership of these directories to the `prometheus` user.

```
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

Once everything is done, run this command to remove `prometheus` directory.

```
cd .. && rm -rf prometheus*
```

Before using Prometheus, it needs some configuration. Create a YAML configuration file named `prometheus.yml` by running the command below.

```
sudo nano /etc/prometheus/prometheus.yml
```

The configuration file is divided into three parts which are `global`, `rule_files`, and `scrape_configs`.

- `scrape_interval` defines how often Prometheus scrapes targets, while `evaluation_interval` controls how often the software will evaluate rules.

- `rule_files` block contains information of the location of any rules we want the Prometheus server to load.

- `scrape_configs` contains the information which resources Prometheus monitors.

The configuration file should look like this below:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "substrate_node"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9615"]
```

With the above configuration file, the first exporter is the one that Prometheus exports to monitor itself. As we want to have more precise information about the state of the Prometheus server we reduced the scrape_interval to 5 seconds for this job. The parameters static_configs and targets determine where the exporters are running. The second exporter is capturing the data from your node, and the port by default is 9615.

You can check the validity of this configuration file by running 

```
promtool check config /etc/prometheus/prometheus.yml
```

Save the configuration file and change the ownership of the file to `prometheus` user.

```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

#### Starting Prometheus

To test that Prometheus is set up properly, execute the following command to start it as the `prometheus` user.

```
sudo -u prometheus /usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries
```

The following messages indicate the status of the server. If you see the following messages, your server is set up properly.

```
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:380 msg="No time or size retention was set so using the default time retention" duration=15d
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:418 msg="Starting Prometheus" version="(version=2.26.0, branch=HEAD, revision=3cafc58827d1ebd1a67749f88be4218f0bab3d8d)"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:423 build_context="(go=go1.16.2, user=root@a67cafebe6d0, date=20210331-11:56:23)"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:424 host_details="(Linux 5.4.0-42-generic #46-Ubuntu SMP Fri Jul 10 00:24:02 UTC 2020 x86_64 ubuntu2004 (none))"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:425 fd_limits="(soft=1024, hard=1048576)"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:426 vm_limits="(soft=unlimited, hard=unlimited)"
level=info ts=2021-04-16T19:02:20.169Z caller=web.go:540 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2021-04-16T19:02:20.170Z caller=main.go:795 msg="Starting TSDB ..."
level=info ts=2021-04-16T19:02:20.171Z caller=tls_config.go:191 component=web msg="TLS is disabled." http2=false
level=info ts=2021-04-16T19:02:20.174Z caller=head.go:696 component=tsdb msg="Replaying on-disk memory mappable chunks if any"
level=info ts=2021-04-16T19:02:20.175Z caller=head.go:710 component=tsdb msg="On-disk memory mappable chunks replay completed" duration=1.391446ms
level=info ts=2021-04-16T19:02:20.175Z caller=head.go:716 component=tsdb msg="Replaying WAL, this may take a while"
level=info ts=2021-04-16T19:02:20.178Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=0 maxSegment=4
level=info ts=2021-04-16T19:02:20.193Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=1 maxSegment=4
level=info ts=2021-04-16T19:02:20.221Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=2 maxSegment=4
level=info ts=2021-04-16T19:02:20.224Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=3 maxSegment=4
level=info ts=2021-04-16T19:02:20.229Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=4 maxSegment=4
level=info ts=2021-04-16T19:02:20.229Z caller=head.go:773 component=tsdb msg="WAL replay completed" checkpoint_replay_duration=43.716µs wal_replay_duration=53.973285ms total_replay_duration=55.445308ms
level=info ts=2021-04-16T19:02:20.233Z caller=main.go:815 fs_type=EXT4_SUPER_MAGIC
level=info ts=2021-04-16T19:02:20.233Z caller=main.go:818 msg="TSDB started"
level=info ts=2021-04-16T19:02:20.233Z caller=main.go:944 msg="Loading configuration file" filename=/etc/prometheus/prometheus.yml
level=info ts=2021-04-16T19:02:20.234Z caller=main.go:975 msg="Completed loading of configuration file" filename=/etc/prometheus/prometheus.yml totalDuration=824.115µs remote_storage=3.131µs web_handler=401ns query_engine=1.056µs scrape=236.454µs scrape_sd=45.432µs notify=723ns notify_sd=2.61µs rules=956ns
level=info ts=2021-04-16T19:02:20.234Z caller=main.go:767 msg="Server is ready to receive web requests."

```

Go to `http://SERVER_IP_ADDRESS:9090/graph` to check whether you are able to access the Prometheus interface or not. If it is working, exit the process by pressing on `CTRL + C`.

Next, we would like to automatically start the server during the boot process, so we have to create a new `systemd` configuration file with the following config.

```
sudo nano /etc/systemd/system/prometheus.service
```

```
[Unit]
  Description=Prometheus Monitoring
  Wants=network-online.target
  After=network-online.target

[Service]
  User=prometheus
  Group=prometheus
  Type=simple
  ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
  ExecReload=/bin/kill -HUP $MAINPID

[Install]
  WantedBy=multi-user.target
```

Once the file is saved, execute the command below to reload `systemd` and enable the service so that it will be loaded automatically during the operating system's startup.

```
sudo systemctl daemon-reload && systemctl enable prometheus && systemctl start prometheus
```

Prometheus should be running now, and you should be able to access its front again end by re-visiting `IP_ADDRESS:9090/`.

#### Installing Grafana

In order to visualize your node metrics, you can use Grafana to query the Prometheus server. Run the following commands to install it first.

```
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_7.5.4_amd64.deb
sudo dpkg -i grafana_7.5.4_amd64.deb
```

If everything is fine, configure Grafana to auto-start on boot and then start the service.

```
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

You can now access it by going to the `http://SERVER_IP_ADDRESS:3000/login`. The default user and password is `admin/admin`.

:::info
**Note**
If you want to change the port on which Grafana runs (3000 is a popular port), edit the file `/usr/share/grafana/conf/defaults.ini` with a command like `sudo vim /usr/share/grafana/conf/defaults.ini` and change the `http_port` value to something else. Then restart grafana with `sudo systemctl restart grafana-server`.
:::

![1-grafana-login-c1c6fbd7d08509b83393b50c01bb0616](https://hackmd.io/_uploads/rkXStIRDyx.png)


In order to visualize the node metrics, click settings to configure the `Data Sources` first.

![2-add-data-source-d761a4186c463aad357c6130b2881789](https://hackmd.io/_uploads/rkzUtICwkl.png)

Click `Add data source` to choose where the data is coming from.

![2-add-data-source-2-1a307a18d157b5a6dcfc5ff9affa9998](https://hackmd.io/_uploads/B1uPtU0v1g.png)

Select `Prometheus`.

![3-select-prometheus-0791dd096d2ca64c0146121e58f9c3e3](https://hackmd.io/_uploads/rk8uKUAvJx.png)

The only thing you need to input is the URL that is `https://localhost:9090` and then click `Save & Test`. If you see `Data source is working`, your connection is configured correctly.

![4-configure-data-source-7b1620ce4fc9ab2de90283415cea7df9](https://hackmd.io/_uploads/BJ6FYLCw1e.png)

Next, import the dashboard that lets you visualize your node data. Go to the menu bar on the left and mouse hover "+" then select `Import`.

`Import via grafana.com` - It allows you to use a dashboard that someone else has created and made public. You can check what other dashboards are available via `https://grafana.com/grafana/dashboards`. In this guide, we use `"Substrate Node Metrics"`, so input `"21715"` under the id field and click `Load`.

![5-import-dashboard-4a6f27887cfd081b9385dfd897787cbd](https://hackmd.io/_uploads/rkK7580Pkx.png)

Once it has been loaded, make sure to select "Prometheus" in the Prometheus dropdown list. Then click `Import`.

![5-import-dashboard-2-b6118a68ef2f8d78c555735471678f22](https://hackmd.io/_uploads/H1x_qLCP1g.png)


In the meantime, start your Relay node by running `./polkadot`. If everything is done correctly, you should be able to monitor your node's performance such as the current block height, network traffic, running tasks, etc. on the Grafana dashboard.

![6-dashboard-metric-52044f98ca5a45715a8731a4cc96ed1b](https://hackmd.io/_uploads/BkO59LRPkg.png)


#### What's Next?

You can now create, build, and connect your own Substrate-based parachain to the relay chain. See [Substrate-based Parachain Deployment Setup](/post-3) for more details.

#### References
- Initial setup: https://wiki.polkadot.network/docs/maintain-guides-how-to-validate-polkadot#initial-set-up
- Using systemd for a validator node: https://wiki.polkadot.network/docs/maintain-guides-how-to-systemd
- Secure Validator Mode: https://wiki.polkadot.network/docs/maintain-guides-secure-validator
- Monitor Your Node: https://wiki.polkadot.network/docs/maintain-guides-how-to-monitor-your-node
