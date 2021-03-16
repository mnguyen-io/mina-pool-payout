# mina-pool-payout

_Inspired by [minaexplorer](https://github.com/garethtdavies/mina-payout-script)_

This script will calculate the required payouts for accounts delegating to a given account. The script will output JSON containing a list of public keys and their payout amounts, e.g.:

```json
[
  {
    "publicKey": "B62qkbdgRRJJfqcyVd23s9tgCkNYuGMCmZHKijnJGqYgs9N3UdjcRtR",
    "total": 4644003513
  },
  {
    "publicKey": "B62qqMo7X8i2NnMxrtKf3PAWfbEADk4V1ojWhGeH6Gvye9hrMjBiZjM",
    "total": 22577193721
  }
]
```

## Getting started

### Setting up your environment

Copy `sample.env` to `.env` and make the following changes within the `.env`:

1. Set `COMMISSION_RATE` to the commission your pool charges. Default is assumed to be the Mina Foundation maximum rate of .05 if a value is not provided.

2. Set `POOL_PUBLIC_KEY` to the public key of the pool account being tracked for payouts. This should be the block producer public key.

3. Set `SEND_TRANSACTION_FEE` to the transaction fee for payout transactions. It is specified in the .env file in MINA, but will be translated to NANOMINA for the actual payment transactions. Double check that this is in Mina!

Keys can either be generated with each run and used as a one-time payout key, or can be provided in the .env config file.

If generating a key for each run, remember that the account will need to be created and there will be an account creation fee - currently the network default is 1 mina.

If providing keys, the 58-char private key should be specified. This can be retrieved from a pk file by running the mina advanced dump-keypair command, e.g.

```bash
mina advanced dump-keypair --privkey-path keys/my-payout-wallet
```

4. Set `SEND_PRIVATE_KEY` to the sender private key. It can be left blank "" if ephemeral keys will be used.

5. Set `SEND_PUBLIC_KEY` to the sender public key. It can also be blank if generating ephemeral keys.

6. Set `SEND_EPHEMERAL_KEY` to true or false. If true, the SEND_PRIVATE_KEY will be ignored and a new keypair will be generated for the payout transactions.

   The consensus parameters will be used to determine finality and the max height if `MAX_HEIGHT` is not provided.

7. Set `GLOBAL_SLOT_START=0` - expect to deprecate

8. Set `SLOTS_PER_EPOCH=7140` - expeect to deprecate

9. Set `MIN_CONFIRMATIONS` to whatever number of confirmed blocks you require before paying out. Default to 290 or "k" to use the assumed network finality. 

    The process will include blocks at a height up to the **lower of** `MAX_HEIGHT` and the current tip minus `MIN_CONFIRMATIONS`. 

    To clarify - `MAX_HEIGHT` only applies _below_ the minimum confirmation window. (i.e. Given a  current block height of 1,500; `MAX_HEIGHT` of 5,000; and `MIN_CONFIRMATIONS` of 290, the process will consider blocks up to height 1210 (1500-290). If `MAX_HEIGHT` were set to 1,000, then the process would consider blocks up to height 1000.)

10. Populate `DATABASE_URL` with the connection string for your archive node Postgresql instance. This will typically look something like:

    ```
    DATABASE_URL=postgresql://USER:PASSWORD@HOST:PORT/DATABASENAME
    ```

11. Export the relevant staking ledger and place in src/data/ledger directory. You can export the current staking ledger with: 

    ```
    coda ledger export staking-epoch-ledger > staking-epoch-ledger.json
    ```

### Running the script

1. Run `npm install` to install the project dependencies.
2. Run `npm run payout -- -m={MIN_BLOCK}` to run the script as a dry run, where `{MIN_BLOCK}` is the lowest blockheight to process. This will not transmit any actual payments and will output a hash of the payment details.
3. Run `npm run payout -- -m={MIN_BLOCK} -h={PAYOUT_HASH}` where `{PAYOUT_HASH}` is the hash produced during the dry run in the prior step. If this run produces the same hash (i.e. nothing has changed since the dry run), then the signed payment(s) will be transmitted.

### Seeing Results ###

The process will output summary informaiton to the console, and will generate several files under the src/data directory. Files will include:

```bash
./src/data/payout_details_[datetime_minblock_maxblock].json - contains the detailed calculations for each delegator key at each block.
./src/data/payout_transactions_[datetime_minblock_maxblock].json - contains the list of payout transactions that should be sent.
./src/data/[nonce].json - contains a signed transaction for each payment that should be sent. These should be broadcast to the network.
```
