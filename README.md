
# Part 1: run LND in Docker container
## Pull the container image from docker hub:
```bash
$ docker pull jnowakowski/neutrino-preloaded
```

## Run container in background:
```
$ docker run -tdi --name lnd jnowakowski/neutrino-preloaded
```

## Check container logs:
Anytime you can check the container logs. Stop it with ctrl+c (safe, it does not stop container). 
```
$ docker logs -f lnd
```

## Use LND CLI to check if synced with blockchain:
Wait a bit. Repeat command below. Observer parameter **"synced_to_chain"**, it will turn **true** when lnd is synced.
```
$ docker exec -it lnd lncli getinfo
{
    "identity_pubkey": "03498859d328b0b63a720e9a4263a9f388b2e85a421afe69960d985473cd604bb4",
    "alias": "03498859d328b0b63a72",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 1282220,
    "block_hash": "0000000000000b4852211aaffd94ff94723f566dc4aabe893fe6dd5e98ba219c",
    "synced_to_chain": false,
    "testnet": true,
    "chains": [
        "bitcoin"
    ],
    "uris": [
    ]
}
```
# Part II: open channel
## When synchronised, get a new address and open channel
Generate a new addres:
```
$ docker exec -it lnd lncli newaddress np2wkh
{
    "address": "2MycQn1S8pj7k8m9RzmJVksQyxAWp1Uzb7y"
}
```
## Visit the faucet to get free tentnet BTC:
http://faucet.lightning.community

Copy the faucet node address and open a connection:
```
docker exec -ti lnd lncli connect 02fa77e0f4ca666f7d158c4bb6675d1436e339903a9feeeaacbd6e55021b98e7ee@159.203.125.125
```

Go back to faucet website and open a channel by providing:
* Target node: your identity_pubkey from **lncli getinfo** command.
* Channel ammount: 100000
* Initial balance: 50000 
After 6 confirmations this transaction will be mined. Check it on 
https://testnet.smartbit.com.au/tx/{your_tx_id_from_faucet}

Meanwhile observer on command line:
```
$ docker exec -ti lnd lncli pendingchannels
```
After 6 confirmations channel shoud be visible with the command:
```
$ docker exec -ti lnd lncli listchannels
```
## After successfull channel opening your wallet should containd test BTC:
```
$ docker exec -ti lnd lncli walletbalance // on-chain balance (remains zero until closing the channel)
$ docker exec -ti lnd lncli channelbalance // channel balance (should be 50'000 satoshis now)
```

# Part III: send to others off-chain, via Lightning Channels
## Connect 
```
$ docker exec -ti lnd lncli connect <identity_pubkey>@<remote_ip_address>
```
## Check connection:
```
$ docker exec -ti lnd lncli listpeers
```
## Open channel:
```
$ docker exec -ti lnd lncli openchannel --node_key=<identity_pubkey> --local_amt=1000000
```
Now wait minimum 6 blocks until transaction is mined. Check it with commands:
```
$ docker exec -ti lnd lncli pendingchannels
$ docker exec -ti lnd lncli listchannels
```
Once funding transaction is mined, channel will move from pending to open.
## Receiving side: create invoice
```
$ docker exec -ti lnd lncli addinvoice --amt=10000
{
        "r_hash": "<random_hash>", 
        "pay_req": "<encoded_invoice>", 
}
**pay_req** is important - should be passed to other side.
```
## Paying side: send funds to the invoice
```
$ docker exec -ti lnd lncli sendpayment --pay_req=<encoded_invoice>
```
## Example transaction
```
### receiving side
orion@lightnode ~ $ docker exec -ti lnd lncli addinvoice 1024
{
        "r_hash": "78c13f43af70c6fa81cc5baaa210b435af2dc72c7430c06bbb34360330a1bf49",
        "pay_req": "lntb10240n1pdg87e5pp50rqn7sa0wrr04qwvtw42yy95xkhjm3evwscvq6amxsmqxv9phaysdqqcqzyswrj6ys3x3r9pgff7ncgamgh5kfe208t2tcrt4rlffvpzshzj3gthet3lvx52leqdprszhe2a0rklezzqh5v7hkfgwuykq5jt976mxysptlc76u"
}
### paying side
ubuntu@ip-172-31-34-239:~$ lncli --no-macaroons sendpayment --pay_req lntb10240n1pdg87e5pp50rqn7sa0wrr04qwvtw42yy95xkhjm3evwscvq6amxsmqxv9phaysdqqcqzyswrj6ys3x3r9pgff7ncgamgh5kfe208t2tcrt4rlffvpzshzj3gthet3lvx52leqdprszhe2a0rklezzqh5v7hkfgwuykq5jt976mxysptlc76u
{
        "payment_error": "",
        "payment_preimage": "1b86b9236a1fc1136ab0d629feb254e655965e5bf400910831a450041b53901c",
        "payment_route": {
                "total_time_lock": 1282894,
                "total_fees": 1,
                "total_amt": 1025,
                "hops": [
                        {
                                "chan_id": 1408016998345146368,
                                "chan_capacity": 100000,
                                "amt_to_forward": 1024,
                                "fee": 1,
                                "expiry": 1282750
                        },
                        {
                                "chan_id": 1410164344553472000,
                                "chan_capacity": 200000,
                                "amt_to_forward": 1024,
                                "expiry": 1282750
                        }
                ]
        }
}
```
## Wallet and channel balance
Can be checked as below:
```
$ docker exec -ti lnd lncli walletbalance
$ docker exec -ti lnd lncli channelbalance
```
# Part IV: Close channel and commit to blockchain
## Closing channel
This will commit fund to the blockchain. First of all get the channel details. Important parameter is **channel_point**:
```
$ docker exec -ti lnd lncli listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "0343bc80b914aebf8e50eb0b8e445fc79b9e6e8e5e018fa8c5f85c7d429c117b38",
       ---->"channel_point": "3511ae8a52c97d957eaf65f828504e68d0991f0276adff94c6ba91c7f6cd4275:0",
            "chan_id": "1337006139441152",
            "capacity": "1005000",
            "local_balance": "990000",
            "remote_balance": "10000",
            "commit_fee": "8688",
            "commit_weight": "724",
            "fee_per_kw": "12000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "10000",
            "total_satoshis_received": "0",
            "num_updates": "2",
            "pending_htlcs": [
            ],
            "csv_delay": 4
        }
    ]
}
```
Channel point consists of two numbers separated by a colon. The first one 
is **"funding_txid"** and the second one is **"output_index"**:
```
$ docker exec -ti lnd lncli closechannel --funding_txid=<funding_txid> --output_index=<output_index>
```
Wallet balance should be updated. This take some time, as on-chain transaction needs to be mined. 

#Part V: cleanup docker
```
$ docker stop lnd
$ docker rm lnd
```

# If faucet do not cooperate, reset the container:
```
$ docker exec -it lnd rm /root/.lnd/data/chain/bitcoin/testnet/wallet.db
$ docker restart lnd
```
Once lnd is synced, start over Part II.
