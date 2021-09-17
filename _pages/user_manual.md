---
layout: single
classes: wide
title: User manual
permalink: /user_manual/
toc: true
toc_icon: "cog"
---

## Known issues

### 1GB lnd channel.db on 32 bits architectures

The node is based on a 32 bits architecture; read https://twitter.com/c_otto83/status/1431589736540475395?s=20

## Troubleshooting

### Docker containers do not stop correctly

The following command will force a stop to all docker containers

```bash
docker stop $(docker ps -a -q) && docker rm $(docker ps -a -q)
```

### Bitcoind container do not start correctly

This is likely due to a chainstate corruption happened during an unsafe shutdown. 
Make sure this is the case by reading bitcoind container logs:

```bash
docker logs btcpayserver_bitcoind -n 20 -f
```

Example output in case of chainstate corruption:

```
2021-09-02T17:43:16Z init message: Verifying blocks...
2021-09-02T17:43:16Z Verifying last 6 blocks at level 3
2021-09-02T17:43:16Z [0%]...[16%]...[33%]...[50%]...[66%]...LevelDB read failure: Corruption: block checksum mismatch: /home/bitcoin/.bitcoin/chainstate/186071.ldb
2021-09-02T17:43:27Z Fatal LevelDB error: Corruption: block checksum mismatch: /home/bitcoin/.bitcoin/chainstate/186071.ldb
2021-09-02T17:43:27Z You can use -debug=leveldb to get more complete diagnostic messages
2021-09-02T17:43:27Z Fatal LevelDB error: Corruption: block checksum mismatch: /home/bitcoin/.bitcoin/chainstate/186071.ldb
2021-09-02T17:43:27Z : Error opening block database.
Please restart with -reindex or -reindex-chainstate to recover.
: Error opening block database.
Please restart with -reindex or -reindex-chainstate to recover.
2021-09-02T17:43:27Z Aborted block database rebuild. Exiting.
2021-09-02T17:43:27Z Shutdown: In progress...
```

The issue can be solved by removing the chainstate, but requires a few days to rebuild it.

```
rm -rf /var/lib/docker/volumes/generated_bitcoin_datadir/_data/chainstate/
```
