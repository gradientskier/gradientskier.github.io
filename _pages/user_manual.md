---
layout: single
classes: wide
title: User manual
permalink: /user_manual/
toc: true
toc_icon: "cog"
---

## Procedures

### Perform a system upgrade

```bash
apt update
apt full-upgrade
reboot
```

### Change the keyboard layout

After changing the keyboard layout, it's important to regenerate the initramfs, otherwise the new keyboard layout won't work for the drive encryption password.

```bash
sudo mkinitramfs -o /boot/initramfs.gz
```

### Backup/restore: migrate blocks, chainstate and electrs db from another drive

Gather info:

```bash
fdisk -l
df -h
ll /mnt/sdb3/volumes/generated_bitcoin_datadir/_data/chainstate
ll /mnt/sdb3/volumes/generated_bitcoin_datadir/_data/blocks
```

Open and mount target device:

```bash
cryptsetup -v luksOpen /dev/sdc2 sdc2_crypt
mkdir -p /mnt/sdc2
mount /dev/mapper/sdc2_crypt /mnt/sdc2
cryptsetup -v luksOpen --key-file /mnt/sdc2/opt/rpi_vault/sda3_luks_keyfile /dev/sdc3 sdc3_crypt
mkdir -p /mnt/sdc3
mount /dev/mapper/sdc3_crypt /mnt/sdc3
```

Then perform block migration:

```bash
ll /mnt/sdc3/volumes/generated_bitcoin_datadir/_data/
rm -rf /mnt/sdc3/volumes/generated_bitcoin_datadir/_data/blocks
rm -rf /mnt/sdc3/volumes/generated_bitcoin_datadir/_data/chainstate
rm -rf /mnt/sdc3/volumes/generated_electrs_datadir/_data/db
rsync -av --progress /mnt/sdb3/volumes/generated_bitcoin_datadir/_data/blocks /mnt/sdc3/volumes/generated_bitcoin_datadir/_data/
rsync -av --progress /mnt/sdb3/volumes/generated_bitcoin_datadir/_data/chainstate /mnt/sdc3/volumes/generated_bitcoin_datadir/_data/
rsync -av --progress /mnt/sdb3/volumes/generated_electrs_datadir/_data/db /mnt/sdc3/volumes/generated_electrs_datadir/_data/
```

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

### Postgres container does not start after restoring an arm32 backup onto a arm64 image

Postgres docker container will fail with

```
2022-01-12 17:23:21.610 UTC [1] FATAL:  database files are incompatible with server
2022-01-12 17:23:21.610 UTC [1] DETAIL:  The database cluster was initialized without USE_FLOAT8_BYVAL but the server was compiled with USE_FLOAT8_BYVAL.
2022-01-12 17:23:21.610 UTC [1] HINT:  It looks like you need to recompile or initdb.
2022-01-12 17:23:21.611 UTC [1] LOG:  database system is shut down
```

You can restore the postgres database using the backup

```bash
# Stop all containers
docker stop $(docker ps -a -q)
# Remove the postgres datadir
rm -rf /var/lib/docker/volumes/generated_postgres_datadir/_data/*
# Start docker container
docker start generated_postgres_1
# Restore database using the dump
cat /var/lib/docker/opt/backups/<your_backup_name>.sql | docker exec -i generated_postgres_1 psql -U postgres
# Ensure databases have been created
docker exec -it generated_postgres_1 psql -U postgres
# In the postgres prompt
\l
exit
# Restart btcpayserver
service btcpayserver restart
```
