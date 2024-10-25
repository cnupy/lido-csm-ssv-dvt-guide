# Comprehensive guide to setting up a distributed Lido CSM validator cluster with SSV Network

The [Community Staking Module](https://operatorportal.lido.fi/modules/community-staking-module) (CSM) is the first permissionless module in the [Lido protocol](https://lido.fi), allowing anyone to start running validators on the Ethereum blockchain with much greater capital efficiency compared to running a regular ("vanilla") Ethereum validator.

A distributed validator ([DVT](https://ethereum.org/en/staking/dvt/)) is an Ethereum validator that runs on more than one node. [SSV Network](https://ssv.network/) is a set of tools providing permissionless access to running distributed validators.

This tutorial uses the Holesky testnet for demonstration purposes, but the same steps can be applied to the mainnet.

# Hardware & system requirements

 - CPU: Quad-core
 - RAM: 16GB 
 - Storage: 512GB NVME SSD (For mainnet at least 2TB)

A full guide to setting up your operating system can be found [here](https://dvt-homestaker.stakesaurus.com/) or [here](https://docs.ethstaker.cc/ethstaker-knowledge-base). For this tutorial, I'm assuming that all cluster members are running Linux with Git and Docker installed, and have properly secured their servers.

# Getting started

Creating a trust-minimised distributed Valdator cluster requires a multi-sig wallet for management, a split contract to distribute rewards to operators, and the client software needed to run the SSV DVT. Operators also need a full Ethereum node with an execution and consensus client of their choice, and the MEV-boost client configured with at least one of the Lido approved relays.

## The SSV client

`ssvnode` is the software that allows validators to be run on a group of independent nodes - a cluster. A complete multi-container `Docker` setup including execution client, consensus client, MEV-Boost and the `ssvnode` client can be found in this repository https://github.com/cnupy/ssv-validator-node and the first step is to clone it:

```
git clone https://github.com/cnupy/ssv-validator-node.git
```

Make sure your user has the `docker` role. If not you can use this command to add it:

```
sudo usermod -a -G docker $USER
```

You will then need to exit the ssh session and log in again.

Finally, you will need to create the `.env` configuration file:

```
cd ssv-validator-node
cp .env.sample .env
```

Edit the `.env` in your favourite editor and set the variable `NETWORK=holesky`.

## Generating Operator Keys

All cluster members will need an encrypted key pair to run their node. To create the operator key pair the operator first generates an encryption password:

```
mkdir secrets
tr -dc 'A-Za-z0-9' </dev/urandom | head -c 16 >> secrets/password
```

And then execute this code to generate the key:

```
touch secrets/encrypted_private_key.json
docker compose run --rm ssv-node \
/go/bin/ssvnode \
generate-operator-keys \
-p password
```

![image](https://hackmd.io/_uploads/HJNmMFVpA.png)

To view the public key use this command:

```
cat secrets/encrypted_private_key.json | jq .pubKey
```

![image](https://hackmd.io/_uploads/HJWsJoppA.png)

**At this point, each operator must make a backup of the `encrypted_private_key.json` and the `password` files!**

## Creating the DV cluster wallet

Detailed instructions on how to create a Safe Wallet can be found [here](https://help.safe.global/en/articles/40868-creating-a-safe-on-a-web-browser). 
The Holesky Testnet Safe deployment can be found at this address: https://holesky-safe.protofire.io

One of the cluster members should obtain the signer addresses from all the cluster members, then connect his signer wallet and choose to create a new Safe. 

![chrome_ofxRcHQItb](https://hackmd.io/_uploads/HJImiiVh0.png)

After giving the Safe a name and selecting the Holesky network, he continues by clicking the `Next` button.

![chrome_0n8nPU5G5q](https://hackmd.io/_uploads/SkrQijV2R.png )

Then he adds all the signer addresses of the cluster members and proceeds to the final step by clicking the `Next` button.

![chrome_qvPajGtE0N](https://hackmd.io/_uploads/S18mijE3A.png)

Finally, he sends the transaction to create the Safe by clicking on the `Create` button.

![chrome_BfjGLxjtYM](https://hackmd.io/_uploads/SkrmjoV3C.png)

## Creating the reward split contract

One of the cluster members should obtain the reward addresses from all the cluster members. Then he should open https://app.splits.org and select to create a `new contract`. Then he should select `Holesky` for the network.

![image](https://hackmd.io/_uploads/B1VDt6S2A.png)

Select `Split` for the contract type.

![image](https://hackmd.io/_uploads/BknFFprh0.png)

Add the reward addresses of all cluster members. Then he can choose whether the contract is immutable (recommended option), whether he wants to sponsor the maintainers of [splits.org](https://splits.org), and whether there is a distribution bounty so that third parties can distribute the rewards in exchange for a small fee.

![image](https://hackmd.io/_uploads/H1q0KaS20.png)

Finally, click the `Create Split` button, execute the transaction and share the created split contract with all cluster members for review.

## Registering the operator

The SSV operator registry is located at this address: https://app.ssv.network
To register an operator each cluster member must connect their wallet, change the Network to `Holesky` in the upper right corner and click the `Join as Operator` button. 

![image](https://hackmd.io/_uploads/ry0mzfcpR.png)

Click the `Register Operator` button.

![image](https://hackmd.io/_uploads/SkGPMG5aC.png)

Fill in the public key generated earlier, select the `Private` toggle and continue to the next step.

![image](https://hackmd.io/_uploads/Sk4AzMqaA.png)

Select `0` for the operator fee and click the `Next` button. 

![image](https://hackmd.io/_uploads/HkYgQfcp0.png)

Finally register the operator by clicking the `Register Operator` button and sign the transaction.

![image](https://hackmd.io/_uploads/BkZQXGcp0.png)

After the transaction is executed the operator is ready for use.

![image](https://hackmd.io/_uploads/rkC67z96A.png)

Take note of the operator ID and click on the `Manage Operator` button.

![image](https://hackmd.io/_uploads/SkeYMEzq6A.png)

In the upper right corner select the three dots button and select the `Permission Settings` option.

![image](https://hackmd.io/_uploads/HJ1i4fq6A.png)

Click on the `Authorized Addresses` option.

![image](https://hackmd.io/_uploads/rkllBzqpA.png)

Enter the address of the `Safe` wallet, click on the `Add Addresses` button and sign the transaction.

![image](https://hackmd.io/_uploads/SJneLMq6A.png)

## Running the Distributed Key Generation (DKG) service endpoint.

To run DKG each operator must enable the `ssv-dkg` service for their node. To do so each cluster member should first set their operator ID in the `.env` file.

```SSV_OPERATOR_ID=1234```

Finally, start the service using this command:

```
docker compose up -d ssv-dkg
```

### Sharing the DKG service - Option 1

If the node has a static public IP one easy option would be to use it as a service endpoint: `https://xxx.xxx.xxx.xxx:3030`. Depending on the operating system and the network equipment there might be additional steps needed to open the port in the firewall and make it publicly accessible.

### Sharing the DKG service - Option 2

If there is no public static IP available or the node is behind CGNAT the operator can use a free VPN service called `tailscale` to share the DKG endpoint. First, create an account at https://login.tailscale.com.

![image](https://hackmd.io/_uploads/BJ0MsSiTC.png)

To install `tailscale` on the node run this command:

```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Copy the authentication link and paste it into the browser to authenticate the node.

![image](https://hackmd.io/_uploads/S136If5TR.png)

Finally, click on the `Connect` button.

![image](https://hackmd.io/_uploads/SJvrPGqpC.png)

After successful authentication run: 

```
sudo tailscale funnel --bg https+insecure://localhost:3030
```

The public endpoint URL https://xxxxx.xxxxx.ts.net is shown on the screen.

![image](https://hackmd.io/_uploads/rkcX_McpC.png)

### Testing the DKG endpoint

The operator can test the DKG endpoint by running this command:

```
docker compose run --rm ssv-dkg ping --ip https://xxx.xxx.xxx.xxxx
```
or

```
docker compose run --rm ssv-dkg ping --ip https://xxxxx.xxxxxx.ts.net
```

If successfully configured a message similar to this one should appear.

![image](https://hackmd.io/_uploads/S1SwkLja0.png)

### Publishing the DKG endpoint to the operator registry

In the upper right corner of the `Operator details` screen select the three dots button and select the `Edit Details` option.

![image](https://hackmd.io/_uploads/HJ1i4fq6A.png)

At the bottom of the screen fill in the `DKG Endpoint` field, click on the `Update` button and confirm the change with your signature.

*Note: setting the endpoint URL requires a port. If using `tailscale` the port is 443.*

![image](https://hackmd.io/_uploads/SJ74b8saA.png)

## Creating the DV cluster

The official SSV [documentation](https://docs.ssv.network/validator-user-guides/validator-management) contains detailed instructions on setting up a distributed cluster.

#### Creating the cluster configuration

After all cluster members have registered their operators one of the cluster members opens the [SSV](https://app.ssv.network/) web app and connects the cluster `Safe` wallet.

![image](https://hackmd.io/_uploads/SyCKB8o6R.png)

Then click on the `Distribute Validator` button on the next page.

![image](https://hackmd.io/_uploads/ryvKUw6pR.png)

And then on the `Generate new key shares` button.

![image](https://hackmd.io/_uploads/HJ4sIP6pR.png)

Select the cluster size, select the cluster operators from the list and click on the `Next` button.

![image](https://hackmd.io/_uploads/BJ6T2P66C.png)

Click on the `Offline` button.

![image](https://hackmd.io/_uploads/H1qSTP6aC.png)

On the next screen select the number of validators the cluster will be running, set the `Withdrawal Address` to Lido's `Withdrawal Vault` - `0xF0179dEC45a37423EAD4FaD5fCb136197872EAd9` as per Lido [documentation](https://docs.lido.fi/deployed-contracts/holesky/) and click on the `Confirm` button next to it.

*Note: The mainnet `Withdrawal Vault`  addresses is `0xB9D7934878B5FB9610B3fE8A5e441e8fad7E293f` ([source](https://docs.lido.fi/deployed-contracts))*

![image](https://hackmd.io/_uploads/HJjhgd6pC.png)

Once confirmed, select the operating system and copy the `DKG ceremony` command. The command can be executed on any machine with `Docker` installed. 

![image](https://hackmd.io/_uploads/r110ZOppA.png)

After executing the command a screen similar to the one below can be observed indicating the successful completion of the DKG ceremony.

![image](https://hackmd.io/_uploads/BypxNdppC.png)

Several files have been generated and placed in a directory named `ceremony-yyyy-MM-dd--hh-mm-ss.mmm` with multiple sub-folders for each validator key `[nonce]-[validator_pubkey]`:

`deposit-[validator_pubkey].json` - this file contains the deposit data necessary to activate the validator

`keyshares-[validator_pubkey].json` - this file contains the keyshares necessary to register the validator on the `SSV` network

`proofs.json` - this file contains the signatures indicating that the ceremony was conducted by the cluster operators and is crucial for resharing the validators with a different set of operators in the future. 

Please ensure to back up all the files securely.

![image](https://hackmd.io/_uploads/H1qw0dTTR.png)

Return to the web browser and acknowledge the statement about validator deposits (although, for the moment, that's not been done yet), and then click on the `Register Validator` button.

![image](https://hackmd.io/_uploads/Hy0I6_paR.png)

Select the validators you are about to register and click on the `Next` button.

*Note: there is a limit of 20 validators you can register in a single transaction. This is due to limitations of the `Safe` Wallet. For this reason, if you need to register more than 20 validators from a single `keyshares.json` file, you will need to repeat the procedure above, without actually executing the DKG ceremony, and upload the same `keyshares.json` file multiple times. The `SSV` web app will recognize previously registered validators "automagically", and skip them.*

![image](https://hackmd.io/_uploads/r1zaZKaa0.png)

On the funding screen select for how long the cluster will be running and click the `Next` button.

*Note: To obtain SSV tokens on the Holesky network the official SSV [faucet](https://faucet.ssv.network/) can be used.*

![image](https://hackmd.io/_uploads/Hy_-EFTpA.png)

Acknowledge and accept the warning regarding cluster balances and fees and click on the `Next` button.

![image](https://hackmd.io/_uploads/HJqJBKa6C.png)

Acknowledge and accept the warning regarding slashing risks and click on the `Next` button.

![image](https://hackmd.io/_uploads/BkZBHFaaC.png)

Approve the `SSV` spending.

![upload_97f6b7fd3454f6a30513b3c7299769ec](https://hackmd.io/_uploads/ryxeQ0t66A.png)

Sign the transaction in the `Safe` and share it with the rest of the cluster members.

![image](https://hackmd.io/_uploads/B1x9PYT6C.png)

Once the signature threshold has been reached and the transaction has been executed, click on the `Register Validators` button.

![image](https://hackmd.io/_uploads/HJ_lpFap0.png)

Again sign and share the transaction with other cluster members.

![image](https://hackmd.io/_uploads/HkuTCYapA.png)

The last step would be to set the cluster fee recipient to Lido's `Execution Layer Rewards Vault` - `0xE73a3602b99f1f913e72F8bdcBC235e206794Ac8`. To do so open the `My Account` tab and click on the `Fee Address` button.

![image](https://hackmd.io/_uploads/BkKHxcT6C.png)

Set the `Fee Recipient Address` field to `0xE73a3602b99f1f913e72F8bdcBC235e206794Ac8` as per Lido [documentation](https://docs.lido.fi/deployed-contracts/holesky/) and click the `Update` button.

*Note: The mainnet `Fee Recipient` address is `0x388C818CA8B9251b393131C08a736A67ccB19297` ([source](https://docs.lido.fi/deployed-contracts))*

![image](https://hackmd.io/_uploads/BJcGWcTTC.png)

Once again sign the transaction in `Safe` and share it with other cluster members for approval.

![image](https://hackmd.io/_uploads/rk1kz5pa0.png)

After this, the validators are ready to be registered with `Lido CSM`.

## MEV Boost

### What is MEV

MEV stands for Maximal Extractable Value. This is the additional value that can be captured by the block proposer by optimising the selection and order of the transactions included in the proposed block. Such an optimisation often requires the use of sophisticated algorithms and access to resources not available to the regular node operator. The parties capable of doing this are called `Searchers`. They find the most profitable transactions, bundle them and provide the bundles to the `Block Builders` who assemble the bundles into a complete block. At the beginning of each epoch, node operators register the validators they control with a `Block Builder` (or `Relay`) of their choice and if they are selected to propose a block they can choose to propose the one provided by the `Relay` in exchange for an additional tip. If the operator wishes to connect to multiple `Relays` a software called `MEV-Boost` is required. Using `MEV-Boost` allows the operator to select the most profitable block from all the connected `Relays`, creating a kind of `Block Marketplace`. In the context of Lido CSM, it is worth noting that running `MEV-Boost` is a requirement. Although there are currently no penalties for proposing self-built blocks, this may change in the future.

### Configuring the MEV-boost client

To configure MEV-boost each cluster memeber needs to edit the `.env` file and set the `BUILDER_API_ENABLED=true` and `MEVBOOST_RELAYS=` to the URL of at least one of Lido's approved MEV relays [here](https://enchanted-direction-844.notion.site/6d369eb33f664487800b0dedfe32171e?v=985cb7e521de43d78c67b7ad29adec84). Multiple relays must be separated by a comma. **The use of unapproved relays is strictly forbidden! All cluster members must use identical configurations (same relays) to avoid missing block proposals due to a lack of consensus!**

## Starting the Node

Each cluster member should start the node with the following command:

```
docker compose up -d
```

At this point, execution and consensus clients should start syncing, and the `SSV` client should start waiting for the validators to be activated. 

## Verify Cluster Registration (optional step)

Before deploying the keys to the Lido CSM module each cluster member can verify the keys registered to SSV are the same that they created via DKG. To do so they must the `Safe` Wallet app, identify the transaction that registered validators to SSV and copy the transaction hash by clicking on the `Copy` icon.

![image](https://hackmd.io/_uploads/r1kfIcTp0.png)

Then run the following in the cluster directory:

```
docker run --rm\
-u root:root \
-v $(pwd)/dkg-output:/dkg \
-v $(pwd)/merge-output:/output \
raekwonthethird/ssv-automate:v0.0.4-holesky \
merge-deposit /dkg -t <TX_HASH_FROM_SAFE> -o /output
```

The expected result would be similar to the following screen:

![image](https://hackmd.io/_uploads/Skh4Yc6TR.png)

The tool will verify the validity of the registered validators (i.e. the correctness of keyshares, user nonce, and operator keys involved), using SSV API.

For additional security, cluster participants could visually check the public keys in their aggregate `deposit-data.json` file in the `merge-output` folder to verify that they match those registered to SSV.

To do so log in on the [SSV](https://app.ssv.network/) web app with the cluster's multi-sig account. Under `My Account` page, verify that there is only one cluster and that the operators in it are the expected cluster participants. Copy the public key of (at least) one of the validators from the cluster by clicking on the copy icon.

![image](https://hackmd.io/_uploads/ryT5c56TC.png)

Then, `grep` the `deposit-data.json` file in the `merge-output` folder to verify that there is a matching public key in it.

```
cat merge-output/deposit_data-yyyy-MM-ddThh-mm-ssZ.json \
| grep --color -E <validator_pub_key>
```

![image](https://hackmd.io/_uploads/ryQLacT6C.png)

## Deploy the keys to Lido CSM

One of the cluster members opens the Lido CSM widget using this address https://csm.testnet.fi/?mode=extended. Note the `mode=extended` parameter. This allows the Lido CSM reward address to be set to the split contract created earlier. He connects the cluster Safe to the widget using `WalletConnect`.

![image](https://hackmd.io/_uploads/HkNhlaHhA.png)

Copies the connection link...

![image](https://hackmd.io/_uploads/SyTJ-6r2R.png)

And pastes it into the Safe `WalletConnect` screen.

![image](https://hackmd.io/_uploads/HJxuZpH3R.png)

He clicks on the `Create Node Operator` button...

![image](https://hackmd.io/_uploads/H1g_Q0dlyl.png)

Pastes the contents of the `deposit-data.json` file into the `Upload deposit data` field. There should be enough ETH/stETH/wstETH deposited in the cluster Safe to cover the bond.

![image](https://hackmd.io/_uploads/rkRYzRuxkg.png)

Expand the `Specify custom addresses` section...

![image](https://hackmd.io/_uploads/Syj0MCOeyg.png)

Set the `Reward Address` field to the `Split` contract address and the `Manager Address` field to the `Safe` wallet address. **Make sure you select the `Extended` option before creating the operator, otherwise the reward address will have ultimate control over the node operator, and since this is a simple splitter contract, you won't be able to make any changes to the operator, as this contract has no signing capabilities.** Check that the correct addresses are set and click the `Create Node Operator` button.

![image](https://hackmd.io/_uploads/rk_kRauekg.png)

Sign the transaction in the safe and share it with the rest of the cluster members.

![image](https://hackmd.io/_uploads/r154TpSnC.png)

Before signing the transaction, the remaining members should check that the transaction details contain the correct manager address (the address of the Safe) and reward address (the address of the split contract).

![image](https://hackmd.io/_uploads/HkliC6Hn0.png)

Once the signature threshold has been reached and the transaction has been executed, the cluster is ready for deposit from Lido CSM.

![image](https://hackmd.io/_uploads/HJwzJRShA.png)

## Monitoring the CSM operator

You can follow [this](https://dvt-homestaker.stakesaurus.com/bonded-validators-setup/lido-csm/monitoring-and-address-management) guide for the steps required to monitor your CSM operator.

## Exiting Validators

You can use the [SSV](https://app.ssv.network/) web app to exit the validators.

![image](https://hackmd.io/_uploads/B13GR56p0.png)
