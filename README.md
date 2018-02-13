
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
# Part II: new address
## When synchronised, get a new address
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
$ docker exec -ti lnd lncli openchannels
```
