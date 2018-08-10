# Setting up a lightning payment channels
## Introduction
This topic is the tutorial on lightning payment channels. A way how to set up a local cluster of nodes Alice and Bob, have them talk to each other, set up channels, and route payments between one another is described. A baseline description of the different components that must work together as part of developing on `lnd` is also shown.
This tutorial assumes you have `lnd` and `lncli` executables already on your machine.
The schema will be as shown below. Keep in mind that you can easily extend this network to include additional nodes, in order to create multihop payments etc. by simply running more local `lnd` instances.
```  
    (1)                        (1)                      
 + ----- +                   + --- +         
 | Alice | <--- channel ---> | Bob | 
 + ----- +                   + --- +                
     |                          |                           
     |                          |                           
     + - - - -  - - - - - - - - + 
                  |
         + --------------- +
           | XSN network | <--- (2)
         + --------------- +
```
## Understanding the components
## LND

  `lnd` is the main component that we will interact with. `lnd` stands for Lightning Network Daemon, and handles channel opening/closing, routing and sending payments, and managing all the Lightning Network state that is separate from the underlying XSN network itself.
  Running an `lnd` node means that it is listening for payments, watching the blockchain, etc. By default it is awaiting user input.
  `lncli` is the command line client used to interact with your `lnd` nodes. Typically, each `lnd` node will be running in its own terminal window, so that you can see its log outputs. `lncli` commands are thus run from a different terminal window.

## Setting up our environment

Developing on `lnd` can be quite complex since there are many more moving pieces, so to simplify that process, we will walk through a recommended workflow.

## Running xsnd

Let’s start by running `xsnd`, if you don’t have it up already. Open up a new terminal window, ensure you have your $GOPATH set, and run:

`./xsnd –testnet --rpcuser=rpcuser –rpcpass=rpcpassword –txindex --zmqpubrawblock=tcp://127.0.0.1:28332`

Breaking down the components:

* `--txindex` is required so that the `lnd` client is able to query historical transactions from xsnd.

* `--testnet` specifies that we are using the testnet network.

* `--rpcuser` and `rpcpass` sets a default password for authenticating to the `xsnd` instance.

* `--zmqpubrawblock` , since `lnd` uses `ZeroMQ` to interface with `xsnd`, your `xsnd` installation must be compiled with ZMQ.  Note that if you installed `xsnd` from source and ZMQ was not present, then ZMQ support will be disabled,      and `lnd` will quit on a connection refused error. Configure the `xsnd` instance for ZMQ with `--zmqpubrawblock`. This options must use unique address in order to provide a reliable delivery of notifications (e.g.  --zmqpubrawblock=tcp://127.0.0.1:28332).

## Starting lnd (Alice’s node)
Now, let’s set up the two `lnd` nodes. To keep things as clean and separate as possible, open up a new terminal window, ensure you have $GOPATH set and $GOPATH/bin in your PATH, and create a new directory under $GOPATH called dev that will represent our development space. We will create separate folders to store the state for alice, and bob, and run all of our `lnd` nodes on different localhostports.
```
# Create our development space
cd $GOPATH
mkdir dev
cd dev

# Create folders for each of our nodes
mkdir alice bob
```

Start up the Alice node from within the alice directory:
```
cd $GOPATH/dev/alice

alice$ ./lnd --nobootstrap --xsncoin.active --xsncoin.testnet --debuglevel=debug --xsncoin.node=xsnd --xsnd.rpcuser=rpcuser --xsnd.rpcpass=rpcpassword --xsnd.zmqpath=tcp://127.0.0.1:28332
```
The Alice node should now be running and displaying output ending with a line beginning with “Waiting for wallet encryption password.”


Breaking down the components:
* `--debuglevel`: The logging level for all subsystems. Can be set to trace, debug, info, warn, error, critical.
* `--xsncoin.testnet`: Specifies `lnd` to use `xsnd` testnet
* `--xsncoin.active`: Specifies that xsn is active. Can also include --litecoin.active to activate Litecoin, --bitcoind.active for bitcoin.
* `--xsncoin.node=xsnd`: Use the `xsnd` full node to interface with the blockchain. 
* `--xsnd.rpcuser` and `--xsnd.rpcpass`: The username and password for the xnsd instance.
* `--xsnd.zmqpath`: The path to the ZMQ socket providing raw blocks.
* `--nobootstrap`: disable automatic network bootstrapping.

## Starting second(Bob’s) node
Just as we did with Alice, start up the Bob node from within the bob directory. Note: if you’re doing this on same machine, you need to configure this node setting up another datadir and logdir to be in separate locations so that there is never a conflict, and listen on different rpc ports.

## Run Bob:
```
# In a new terminal window

cd $GOPATH/dev/bob

# running from same machine:

bob$ ./lnd --rpclisten=localhost:10002 --listen=localhost:10012 --restlisten=localhost:8002 --datadir=data --logdir=log --nobootstrap --xsncoin.active --xsncoin.testnet --debuglevel=debug --xsncoin.node=xsnd --xsnd.rpcuser=rpcuser --xsnd.rpcpass=rpcpassword –xsnd.zmqpath=tcp://127.0.0.1:28332

# running from another machine (just the same as with first node):

bob$ ./lnd --nobootstrap --xsncoin.active --xsncoin.testnet --debuglevel=debug --xsncoin.node=xsnd --xsnd.rpcuser=rpcuser --xsnd.rpcpass=rpcpassword –xsnd.zmqpath=tcp://127.0.0.1:28332
```

Breaking down additional options:
* `--rpclisten`: The host:port to listen for the RPC server. This is the primary way an application will communicate with `lnd`
* `--listen`: The host:port to listen on for incoming P2P connections. This is at the networking level, and is distinct from the Lightning channel networks and Bitcoin/Litcoin network itself.
* `--restlisten`: The host:port exposing a REST api for interacting with `lnd` over HTTP. For example, you can get Alice’s channel balance by making a GET request to localhost:8001/v1/channels. This is not needed for this tutorial, but you can see some examples here.
* `--datadir`: The directory that lnd’s data will be stored inside
* `--logdir`: The directory to log output.

## Working with lncli and authentication
Now that we have our `lnd` nodes up and running, let’s interact with them! To control `lnd` we will need to use `lncli`, the command line interface.

`lnd` uses macaroons for authentication to the rpc server. `lncli` typically looks for `admin.macaroon` file in the `lnd` home directory, but since we changed the location of our application data, we have to set `--macaroonpath` in the following command. To disable macaroons, pass the `--no-macaroons` flag into both `lncli` and `lnd`.

`lnd` allows you to encrypt your wallet with a passphrase and optionally encrypt your cipher seed passphrase as well. This can be turned off by passing --noencryptwallet into `lnd` or `lnd.conf`. We recommend going through this process at least once to familiarize yourself with the security and authentication features around `lnd`.
We will test our rpc connection to the Alice node. Let’s create Alice’s wallet and set her passphrase:
```
cd $GOPATH/dev/alice

alice$ ./lncli  --macaroonpath=~/.lnd/admin.macaroon create
```
You’ll be asked to input and confirm a wallet password for Alice, which must be longer than 8 characters. You also have the option to add a passphrase to your cipher seed. For now, just skip this step by entering “n” when prompted about whether you have an existing mnemonic, and pressing enter to proceed without the passphrase.

You can now request some basic information as follows:
```
alice$ ./lncli --macaroonpath=~/.lnd/admin.macaroon getinfo
```

`lncli` just made an RPC call to the Alice `lnd` node. This is a good way to test if your nodes are up and running and `lncli` is functioning properly. Note that in future sessions you may need to call `lncli` unlock to unlock the node with the password you just set.

Switch to another machine/terminal to do the same for Bob’s node.
```
# from another machine

cd $GOPATH/dev/bob

bob$ ./lncli –macaroonpath=~/.lnd/admin.macaroon create

# Note that if we’re running on the same machine we specify the --rpcserver here, which corresponds to --rpcport=10002 that we set when started the Bob’s lndnode.

bob$ ./lncli –rpcserver=localhost:10002 –macaroonpath=~/.lnd/admin.macaroon create
```

##### lncli options

To see all the commands available for `lncli`, simply type `lncli --help` or `lncli -h`.

## Setting up XSN addresses
Let’s create a new address for Alice. This will be the address that stores Alice’s on-chain balance. `np2wkh` specifes the type of address and stands for Pay to Nested Witness Key Hash.
```
Alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon newaddress np2wkh
{
    "address": <ALICE_ADDRESS>
}
```
And for Bob’s node:
```
bob$ ./lncli –macaroonpath=~/.lnd/admin.macaroon newaddress np2wkh
{
    "address": <BOB_ADDRESS>
}
```

## Funding wallets
That’s a lot of configuration! At this point, we’ve generated onchain addresses for Alice and Bob. Now, we will send some coins to our wallets in order to be able to create some contracts in future.

From xsn-cli send some coins to previousl generated addresses:
```
./xsn-cli --testnet --rpcuser=rpcsuer –rpcpassword=rpcapssword sendtoaddress <ALICE_ADDRESS> 10
```
Now we should wait for 6 blocks to be generated in order to see our coins using `lnd` wallet, this can take some time

After that check wallet balance.
```
alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon walletbalance
```

It’s no fun if only Alice has money. Let’s give some to Bob as well.
```
./xsn-cli --testnet --rpcuser=rpcsuer –rpcpassword=rpcapssword sendtoaddress <BOB_ADDRESS> 10
# Check Bob’s balance
bob$ ./lncli –macaroonpath=~/.lnd/admin.macaroon walletbalance
```

## Creating the P2P Network

Now that Alice and Bob have some testnet coins, let’s start connecting them together.
Connect Alice to Bob:
```
# Get Bob's identity pubkey:
bob$ ./lncli –macaroonpath=~/.lnd/admin.macaroon getinfo
{
    "identity_pubkey": ←--- BOB_IDENTITY_PUBKEY "02d3b335ee98935cd1c071189b2ffe6c30c107e6d9c1aaf1e5020b090de5a1219a",
    "alias": "02d3b335ee98935cd1c0",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 15162,
    "block_hash": "9446d629705a5e8db48c2b894b028f1cf7349613633d20c7f17b4099b68bba5b",
    "synced_to_chain": false,
    "testnet": true,
    "chains": [
        "xsncoin"
    ],
    "uris": [
    ],
    "best_header_timestamp": "1533882410",
    "version": "0.4.2-beta commit=bb310a3d0d0f0cab1d07cb99c3f03072fa6118bf"
}

# Connect Alice and Bob together

alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon connect <BOB_IDENTITY_PUBKEY>@<bob’s_machine_ip:9735
{

}

# Note 9735 is a default ip that lnd listen’s, if you’re connected with –rpclisten=<some_port> you should use this port instead of 9735.
```
Let’s check that Alice and Bob are now aware of each other.
```
# Check that Alice has added Bob as a peer:

alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon listpeers
{
    "peers": [
        {
            "pub_key": "032b91f551ccdf9a013fdb4621d68b29471851abfa6e36e136408b87cbc4219ed4",
            "address": "107.21.133.151:9735",
            "bytes_sent": "402",
            "bytes_recv": "979",
            "sat_sent": "10000",
            "sat_recv": "0",
            "inbound": false,
            "ping_time": "0"
        }
    ]
}

# Check that Bob has added Alice as a peer:

bob$ ./lncli –macaroonpath=~/.lnd/admin.macaroon listpeers
{
    "peers": [
        {
            "pub_key": "02d3b335ee98935cd1c071189b2ffe6c30c107e6d9c1aaf1e5020b090de5a1219a",
            "address": "93.77.152.213:47808",
            "bytes_sent": "979",
            "bytes_recv": "402",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": true,
            "ping_time": "0"
        }
    ]
}
```

## Setting up Lightning Network
Before we can send payment, we will need to set up payment channels from Alice to Bob.

Let’s open the Alice<–>Bob channel.
```
alice$ ./lncli --macaroonpath=~/.lnd/admin.macaroon openchannel <BOB_PUBKEY> --local_amt=1000000
```
* `--local_amt` specifies the amount of money that Alice will commit to the channel. To see the full list of options, you can try `lncli openchannel --help`.

We now need to wait for six new blocks for the channel to be considered as valid.

Check that Alice<–>Bob channel was created:
```
alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "032b91f551ccdf9a013fdb4621d68b29471851abfa6e36e136408b87cbc4219ed4",
            "channel_point": "3334e320f5ae9ca0f8ebe43fd782fc2865ad1635a6652af07047193e8b5f7893:1",
            "chan_id": "13849448463597569",
            "capacity": "100000000",
            "local_balance": "99989817",
            "remote_balance": "10000",
            "commit_fee": "10183",
            "commit_weight": "600",
            "fee_per_kw": "253",
            "unsettled_balance": "0",
            "total_satoshis_sent": "10000",
            "total_satoshis_received": "0",
            "num_updates": "2",
            "pending_htlcs": [
            ],
            "csv_delay": 801,
            "private": false
        }
    ]
}
```
## Sending single hop payments
Finally, to the exciting part - sending payments! Let’s send a payment from Alice to Bob.
First, Bob will need to generate an invoice:
```
bob$ ./lncli –macaroonpath=~/.lnd/admin.macaroon addinvoice --amt=10000
{
        "r_hash": "<a_random_rhash_value>",
        "pay_req": "<encoded_invoice>",
}
```

Send the payment from Alice to Bob:
```
alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon sendpayment --pay_req=<encoded_invoice>
{ 
	"payment_error": "", 
	"payment_preimage": "1483fc568676ade417780ba90d034d3501005657932f7374e72ad1ef647ec84b", 
	"payment_route": { 
		"total_time_lock": 13177, 
		"total_amt": 10000, 
		"hops": [ 
			   { 
				"chan_id": 13849448463597569, 
				"chan_capacity": 98999817, 
				"amt_to_forward": 10000, 
				"expiry": 13177, 
				"amt_to_forward_msat": 10000000 
			   } 
		], 
	"total_amt_msat": 10000000 
	} 
}
```

```
# Check that Alice's channel balance was decremented accordingly:
alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon listchannels

# Check that Bob's channel was credited with the payment amount:
bob$ ./lncli –macaroonpath=~/.lnd/admin.macaroon listchannels
```

## Closing channels
Let’s try closing a channel.
```
alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "032b91f551ccdf9a013fdb4621d68b29471851abfa6e36e136408b87cbc4219ed4",
            "channel_point": "3334e320f5ae9ca0f8ebe43fd782fc2865ad1635a6652af07047193e8b5f7893:1",
            "chan_id": "13849448463597569",
            "capacity": "100000000",
            "local_balance": "99989817",
            "remote_balance": "10000",
            "commit_fee": "10183",
            "commit_weight": "600",
            "fee_per_kw": "253",
            "unsettled_balance": "0",
            "total_satoshis_sent": "10000",
            "total_satoshis_received": "0",
            "num_updates": "2",
            "pending_htlcs": [
            ],
            "csv_delay": 801,
            "private": false
        }
    ]
}
```
The Channel point consists of two numbers separated by a colon, which uniquely identifies the channel. The first number is `funding_txid` and the second number is `output_index`.

```
# Close the Alice<-->Bob channel from Alice's side.
alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon closechannel --funding_txid=<funding_txid> --output_index=<output_index>

#  Wait for 1 block to be generate including the channel close transaction to close the channel:

# Check that Bob's on-chain balance was credited by his settled amount in the channel.
alice$ ./lncli –macaroonpath=~/.lnd/admin.macaroon walletbalance
{
    "total_balance": "100022154",
    "confirmed_balance": "100022154",
    "unconfirmed_balance": "0"
}
```
