# Update already installed delta testnet validator node on Ubuntu 20.04

> **Note**  
> Do not execute all the commands below as root. sudo is included where it is required. 

## Update software

### Stop the node if it is running

```
sudo systemctl stop casper-node
```

### Back-up your old config

```
sudo mv /etc/casper/config.toml /etc/casper/config.toml.old
```

### Remove previously installed node software

```
sudo apt remove -y casper-client casper-node
```

### Install new node software

```
cd ~

curl -JLO https://bintray.com/casperlabs/debian/download_file?file_path=casper-client_0.3.1-3423_amd64.deb
curl -JLO https://bintray.com/casperlabs/debian/download_file?file_path=casper-node_0.3.1-3423_amd64.deb
sudo apt install -y ./casper-node_0.3.1-3423_amd64.deb ./casper-client_0.3.1-3423_amd64.deb 
```

## Re-build smart contracts that are required to bond to the network 

### Build smart contracts

#### Go to the directory with casper-node sources

```
cd ~/casper-node
```

#### Pull the latest changes

```
git pull
```

#### Checkout the release branch:

```
git checkout release-0.3.1
```

#### Remove previous builds

```
make clean
```

#### Build the contracts

```
make setup-rs
make build-client-contracts -j
```

## Fund your account

To fund an account visit the [Faucet](https://clarity.casperlabs.io/#/faucet) page. Select the account you want to fund and hit "Request Tokens". Wait until the request transaction succeeds.

## Clean up after the previous run

### Rotate logs

```
sudo logrotate -f /etc/logrotate.d/casper-node
```

### Delete the database from the previous run

```
sudo /etc/casper/delete_local_db.sh
```

## Run the node

### Set trusted hash

Get the trusted hash value from an already bonded validator

```
KNOWN_ADDRESSES=$(cat /etc/casper/config.toml | grep known_addresses)
KNOWN_VALIDATOR_IP=$(grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' <<< "$KNOWN_ADDRESSES")

curl -s http://$KNOWN_VALIDATOR_IP:7777/status | jq .last_added_block_info.hash
```

If the hash is not null update ```trusted_hash``` property at the top of the config file. If it is null make sure it is commented in the file.

```
sudo nano /etc/casper/config.toml
```

### Start the node
sudo systemctl start casper-node

### Monitor the node status

#### Check the node log

```
tail -n100 -f /var/log/casper/casper-node.log
```

#### Check if a known validator sees your node among peers

```
curl -s http://$KNOWN_VALIDATOR_IP:7777/status | jq .peers
```

You should see your IP address on the list

#### Check the node status

```
curl -s http://127.0.0.1:7777/status
```

## Bond to the network

Once you ensure that your node is running correctly and is visible by other proceed to bonding.

### Check your balance

Check your balance to ensure you have funds to bond:

To get the balance we need to perform the following three query commands:

1. Get the state root hash (has to be performed for each balance check because the hash changes with time): 

    ```
    casper-client get-state-root-hash --node-address http://127.0.0.1:7777 | jq -r
    ```
2. Get the main purse associated with your account:

    ```
    casper-client query-state --node-address http://127.0.0.1:7777 --key <PUBLIC_KEY_HEX> --state-root-hash <STATE_ROOT_HASH> | jq -r
    ```

3. Get the main purse balance:

    ```
    casper-client get-balance --node-address http://127.0.0.1:7777 --purse-uref <PURSE_UREF> --state-root-hash <STATE_ROOT_HASH> | jq -r
    ```

If you followed the installation steps from this document you can run the following script to check the balance:

```
PUBLIC_KEY_HEX=$(cat /etc/casper/validator_keys/public_key_hex)
STATE_ROOT_HASH=$(casper-client get-state-root-hash --node-address http://127.0.0.1:7777 | jq -r '.result | .state_root_hash')
PURSE_UREF=$(casper-client query-state --node-address http://127.0.0.1:7777 --key "$PUBLIC_KEY_HEX" --state-root-hash "$STATE_ROOT_HASH" | jq -r '.result | .stored_value | .Account | .main_purse')
casper-client get-balance --node-address http://127.0.0.1:7777 --purse-uref "$PURSE_UREF" --state-root-hash "$STATE_ROOT_HASH" | jq -r '.result | .balance_value'
```

### Make bonding request

To bond to the network as a validator you need to submit your bid using ```casper-client```:

```
casper-client put-deploy \
        --chain-name "casper-delta-5" \
        --node-address "http://127.0.0.1:7777/" \
        --secret-key "/etc/casper/validator_keys/secret_key.pem" \
        --session-path "$HOME/casper-node/target/wasm32-unknown-unknown/release/add_bid.wasm" \
        --payment-amount 1000000000 \
        --session-arg=public_key:"public_key='<PUBLIC_KEY_HEX>'" \
        --session-arg=amount:"u512='9000000000000000'" \
        --session-arg=delegation_rate:"u64='10'"
```

Where:
- ```amount``` - This is the amount that is being bid. If the bid wins, this will be the validator’s initial bond amount. Recommended bid in amount is 90% of your faucet balance.  This is 900,000 CSPR  or 9000000000000000 motes as an argument to the add_bid contract deploy. 
- ```delegation_rate``` - The percentage of rewards that the validator retains from delegators that delegate their tokens to the node.

Replace ```<PUBLIC_KEY_HEX>``` with the hex representation of you public key. 

Remember the ```deploy_hash``` returned in the response to query its status later.

If you followed the installation steps from this document you can run the following script to bond. It substitutes the public key hex value for you and sends recommended argument values:

```
KNOWN_ADDRESSES=$(cat /etc/casper/config.toml | grep known_addresses)
KNOWN_VALIDATOR_IP=$(grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' <<< "$KNOWN_ADDRESSES")
PUBLIC_KEY_HEX=$(cat /etc/casper/validator_keys/public_key_hex)
CHAIN_NAME=$(curl -s http://$KNOWN_VALIDATOR_IP:8888/status | jq -r '.chainspec_name')

casper-client put-deploy \
    --chain-name "$CHAIN_NAME" \
    --node-address "http://$KNOWN_VALIDATOR_IP:7777/" \
    --secret-key "/etc/casper/validator_keys/secret_key.pem" \
    --session-path "$HOME/casper-node/target/wasm32-unknown-unknown/release/add_bid.wasm" \
    --payment-amount 1000000000 \
    --session-arg=public_key:"public_key='$PUBLIC_KEY_HEX'" \
    --session-arg=amount:"u512='9000000000000000'" \
    --session-arg=delegation_rate:"u64='10'"
```

### Check that you bonding request worked

Sending a transaction to the network does not mean that the transaction processed successfully. It’s important to check to see that the contract executed properly:

```
casper-client get-deploy --node-address http://<KNOWN_VALIDATOR_IP>:7777 <DEPLOY_HASH> | jq .result.execution_results
```

Replace ```<DEPLOY_HASH>``` with the deploy hash of the transaction you want to check.

If you followed the installation steps from this document you can run the following script to get a known validator IP:

```
KNOWN_ADDRESSES=$(cat /etc/casper/config.toml | grep known_addresses)
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' <<< "$KNOWN_ADDRESSES"
```

### Query the auction info and look for your bid

To determine if the bid was accepted, execute the following command:

```
casper-client get-auction-info --node-address http://<KNOWN_VALIDATOR_IP>:7777
```

The bid should appear among the returned ```bids```. If the public key associated with a bid appears in the ```validator_weights``` structure for an era, then the account is bonded in that era.

If you followed the installation steps from this document you can run the following command:

```
casper-client get-auction-info --node-address http://$KNOWN_VALIDATOR_IP:7777
```