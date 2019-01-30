# Quick guide for server deploy

## Setting up your development environment

Install ZeroMQ:
```
apt-get update
apt-get install libzmq3-dev
```
Install RocksDB:
```
apt-get install -y build-essential git wget pkg-config libzmq3-dev libgflags-dev libsnappy-dev zlib1g-dev libbz2-dev liblz4-dev
git clone https://github.com/facebook/rocksdb.git
cd rocksdb
CFLAGS=-fPIC CXXFLAGS=-fPIC make release
cd ..
```
Install Docker-CE:

You can see how to install Docker [here](https://docs.docker.com/install/linux/docker-ce/ubuntu/).

For example: for ubuntu:
```
apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get install docker-ce
```

## Build

Prepare:
```
cd ~/
git clone https://github.com/trezor/blockbook.git
cd blockbook
vi build/docker/bin/Dockerfile
```
Find `FROM debian:9`, then modify according to the OS of the server.

For example, `FROM ubuntu:18.04`, `FROM ubuntu:16.04`, etc.

Build:
```
make deb-blockbook-fujicoin
```

## Deploy

Find your fujicoin core from [here](https://download.fujicoin.org/).

```
cd ~/
mkdir .fujicoin
cd .fujicoin
vi fujicoin.conf
```
For example:
```
server=1
listen=1
daemon=1
rpcuser=username123
rpcpassword=password456
rpcallowip=127.0.0.1
port=3777
rpcport=3776
addresstype=legacy
changetype=legacy
txindex=1
zmqpubhashtx=tcp://127.0.0.1:38348
zmqpubhashblock=tcp://127.0.0.1:38348
rpcworkqueue=1100
maxmempool=2000
dbcache=1000
```
Get latest raw blockchain DB:
```
wget https://download.fujicoin.org/Raw_Blockchain_DB/Fujicoin-3.0-DB-txi1-190130.zip  # for example
unzip Fujicoin-3.0-DB-txi1-190130.zip
rm Fujicoin-3.0-DB-txi1-190130.zip
```
Run fujicoind:
```
cd ~/
mkdir fujicoin
wget https://download.fujicoin.org/fujicoin-v0.16.3.1/fujicoin-0.16.3-x86_64-linux-gnu.tar.gz  # for example
tar -xzvf fujicoin-0.16.3-x86_64-linux-gnu.tar.gz
./fujicoind
```
Confirm synchronization:
```
tail -F ~/.fujicoin/debug.log
```
Prepare for blockbook:
```
cd ~/blockbook
contrib/scripts/build-blockchaincfg.sh fujicoin
mv server/testcert.crt  server/testcert.key .
mv build/blockbook build/blockchaincfg.json .
vi blockchaincfg.json
```
Fix "rpc_url" "rpc_user" "rpc_pass". For example:
```
{
    "coin_name": "Fujicoin",
    "coin_shortcut": "FJC",
    "coin_label": "Fujicoin",
    "rpc_url": "http://127.0.0.1:3776",
    "rpc_user": "username in fujicoin.conf",
    "rpc_pass": "password in fujicoin.conf",
    "rpc_timeout": 25,
    "parse": true,
    "message_queue_binding": "tcp://127.0.0.1:38348",
    "subversion": "",
    "address_format": "",
    "mempool_workers": 8,
    "mempool_sub_workers": 2,
    "block_addresses_to_keep": 300
}
```
Run blockbook:
```
./blockbook -sync -blockchaincfg=blockchaincfg.json -internal=:9048 -public=:9148 -certfile=testcert -logtostderr
```

## Configure apache2

Setting of wss is necessary.

```
vi /etc/apache2/sites-available/default-ssl.conf
```
Add below, for example:
```
        <VirtualHost *:443>
                ServerAdmin root@localhost
                ServerName explorer.fujicoin.org

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                ProxyRequests Off
                SSLEngine on
                SSLProxyEngine on

                SSLCertificateFile    /etc/letsencrypt/live/explorer.fujicoin.org/fullchain.pem
                SSLCertificateKeyFile /etc/letsencrypt/live/explorer.fujicoin.org/privkey.pem

                <Proxy *>
                        Order deny,allow
                        Allow from all
                </Proxy>
                        ProxyPass /socket.io/ wss://localhost:9148/socket.io/
                        ProxyPassReverse /socket.io/ wss://localhost:9148/socket.io/

                        ProxyPass / https://localhost:9148/
                        ProxyPassReverse / https://localhost:9148/

        </VirtualHost>
```
Restart apache2:
```
a2enmod proxy_http
a2enmod proxy_wstunnel
service apache2 restart
```

## Note

- In case of the old Ubuntu 16.04 it is necessary to upgrade to the latest version(>=16.04.5).
