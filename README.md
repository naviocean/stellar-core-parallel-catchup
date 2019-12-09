# Parallel Stellar Core Catchup ⚡

## Background

### Goal

Sync a full Stellar validator node (including full history) as fast as possible.

### Problem

A full catchup takes weeks/months – even without publishing to an archive.

### Idea

 * Split the big ledger into small chunks of size `CHUNK_SIZE`.
 * Run a catchup for the chunks in parallel with `WORKERS` worker processes.
 * Stitch together the resulting database and history archive.

## Usage

```
./catchup.sh DOCKER_COMPOSE_FILE LEDGER_MIN LEDGER_MAX CHUNK_SIZE WORKERS
```

Arguments:

* `DOCKER_COMPOSE_FILE`: use `docker-compose.pubnet.yaml` for the public network (`docker-compose.testnet.yaml` for testnet).
* `LEDGER_MIN`: smallest ledger number you want. Use `1` for doing a full sync.
* `LEDGER_MAX`: largest ledger number you want, usually you'll want the latest one which is exposed as `core_latest_ledger` in any synced Horizon server, e.g. https://horizon.stellar.org/.
* `CHUNK_SIZE`: number of ledgers to work on in one worker.
* `WORKERS`: number of workers that should be spawned. For best performance this should not exceed the number of CPUs.

`Note: LEDGER_MAX % CHUNK_SIZE == 0`

## Hardware sizing and timing examples

* On Google Cloud a full sync took less than 24h on a `n1-standard-32` machine (32 CPUs, 120GB RAM, 1TB SSD) with a `CHUNK_SIZE` of `32768` and 32 workers (see below).
* ... add your achieved result here by submitting a PR.

## Example run on dedicated Google Cloud machine

```
sudo apt-get update
sudo apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg2 \
  software-properties-common \
  python-pip
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/debian \
  $(lsb_release -cs) \
  stable"
sudo apt-get update
sudo apt-get install -y docker-ce
sudo pip install docker-compose
echo '{"default-address-pools":[{"base":"172.80.0.0/16","size":29}]}' | sudo tee /etc/docker/daemon.json
sudo usermod -G docker andre
sudo reboot
# log in again and check whether docker works
docker ps
```

```
git clone https://github.com/naviocean/stellar-core-parallel-catchup.git
cd stellar-core-parallel-catchup
./catchup.sh docker-compose.pubnet.yaml 1 20971520 32768 32 2>&1 | tee catchup.log
```

You will get 3 important pieces of data for Stellar Core:

* SQL database: if you need to move the data to another container/machine you can dump the database by running the following:

    ```
    docker exec catchup-result_stellar-core-postgres_1 pg_dump -F d -f catchup-sqldump -j 10 -U postgres -d stellar-core
    ```

    Then copy the `catchup-sqldump` directory to the target container/machine 
    
    ```
    docker cp catchup-result_stellar-core-postgres_1:/catchup-sqldump ./
    ```
    
    and restore with `pg_restore`.
    
    ```
    docker cp catchup-sqldump hawking_stellar-core-postgres_1:/
    docker exec hawking_stellar-core-postgres_1 pg_restore -F d /catchup-sqldump -j 10 -U postgres -d stellar-core
    ```

* `data-result` directory: contains the `buckets` directory that Stellar Core needs for continuing with the current state in the SQL database.
    create a new container to mount 
    
    ```
    docker container create --name hawking-stellar-core -v hawking_core-data:/data hello-world
    docker cp ./data-result/buckets hawking-stellar-core:/data/
    docker cp ./history-result/ hawking-stellar-core:/data/history
    docker rm hawking-stellar-core
    ```
    
* `history-result` directory: contains the full history that can be published to help other validator nodes to catch up (e.g., S3, GCS, IPFS, or any other file storage).

Note: make sure you have a consistent state of the three pieces of data before starting Stellar Core in SCP mode (e.g., when moving data to another machine).

## Fix issue `Gap detected in stellar-core database (ledger=XXXX)`

 `"history_latest_ledger": 632219,` -> We'll call this value `ORIGINAL_CURRENT_LEDGER`
 
 ` "history_elder_ledger": 40010,` -> We'll call this value `ORIGINAL_ELDER_LEDGER`

#### Backfill history

1. Reingest Horizon
```
docker-compose -p hawking exec stellar-horizon horizon db backfill
```
2. Backfill
```
docker-compose -p hawking exec stellar-horizon horizon db backfill {ORIGINAL_ELDER_LEDGER - 1}
```
3. Stop your stellar-core instance
4. Connect to your core database, and run
```sql
SELECT ledgerseq FROM ledgerheaders ORDER BY ledgerseq ASC LIMIT 1;
```
this will return the _smallest_ ledger that your core instance knows about.

We'll call this value `ORIGINAL_LOW_LEDGER` for now on.

We'll call `LOW_LEDGER` the value that you want in history.

#### Reconstruction steps

Steps here are going to reconstruct history for the range `LOW_LEDGER .. ORIGINAL_LOW_LEDGER`

1. Prepare a fresh new core instance with a similar configuration file
    * use a different database, sqlite even
    * don't start it after running `newdb`
2. Perform a command line catchup to replay a range of ledger that includes the target range
```
stellar-core --conf X.conf catchup ORIGINAL_LOW_LEDGER/(ORIGINAL_LOW_LEDGER-LOW_LEDGER)
```

If this succeeds you can proceed to the next step

#### Merging steps

Repeat the following steps for the following sql tables:
* `ledgerheaders` (`ledgerseq`)
* `txhistory` (`ledgerseq`)
* `txfeehistory` (`ledgerseq`)
not imported from history as of this writing:
* `scphistory`
* `scpquorums`

We'll use `ledgerheaders` here to illustrate.

1. On the temporary node, export the data from `ledgerheaders` (save into a file `ledgerHeaders.sql`)
```sql
SELECT * from ledgerheaders WHERE ledgerseq >= LOW_LEDGER AND ledgerseq < ORIGINAL_LOW_LEDGER
```
2. (optional) verify that the hash of ledger `ORIGINAL_LOW_LEDGER-1` is the one stored in ledger `ORIGINAL_LOW_LEDGER`
```sql
SELECT ledgerhash FROM ledgerheaders WHERE ledgerseq = ORIGINAL_LOW_LEDGER-1
```
3. Login into the core database
```sql
SELECT prevhash FROM ledgerheaders WHERE ledgerseq = ORIGINAL_LOW_LEDGER
```
verify that this value is the same than the `ledgerhash` returned on the previous step
4.  and import the data exported in step 1
5. Verify that you do not have gaps
```sql
SELECT lh1.ledgerseq FROM ledgerheaders AS lh1 WHERE lh1.ledgerseq NOT IN ( SELECT lh2.ledgerseq-1 FROM ledgerheaders AS lh2 WHERE lh2.ledgerseq = lh1.ledgerseq + 1);
```

When this is done, you can start core, wait for it to catchup to the network.

After that you can reset Horizon and have it ingest all data.


## Reset

If you need to start from scratch again you can delete all docker-compose projects:

```
for PROJECT in $(docker ps --filter "label=com.docker.compose.project" -q | xargs docker inspect --format='{{index .Config.Labels "com.docker.compose.project"}}'| uniq | grep catchup-); do docker-compose -f docker-compose.pubnet.yaml -p $PROJECT down -v; done
docker volume prune
```
