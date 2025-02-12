# solana 节点搭建

## 记录一下solana节点搭建的步骤和可能遇到的问题， 以便以后查阅和帮助需要搭建节点的小伙伴

## 版本
- solana-cli 2.0.18
- yellowstone-grpc v3.1.0
- ubuntu 22.04

## 1. 准备

### 1.1. 硬件
- CPU solana构建索引的时候非常吃cpu, 我搭建的时候cpu是AMD EPYC 9554P 64-Core
- RAM 稳定运行的内存大概是400G左右, 所以安全一点推荐配置512G及其以上的内存
- 网络 1Gbps+
- 磁盘 两个nvme 2T以上的容量(重要, 一定要分开挂载到ledger和accounts上)


### 1.2. 软件

- solana-cli 2.0.18
- yellowstone-grpc v3.1.0 (grpc, 可以不装)
- ubuntu 22.04

### 2. 搭建

#### 2.1. 创建文件夹
```bash

mkdir /root/sol
mkdir /root/sol/accounts
mkdir /root/sol/ledger
mkdir /root/sol/bin

```
#### 2.2. 准备硬盘挂载

```bash

# n + e
fdisk /dev/nvme0n1
fdisk /dev/nvme1n1

mkfs -t ext4 /dev/nvme0n1
mkfs -t ext4 /dev/nvme1n1

# 将格式化好的硬盘挂载到对应目录上去
mount /dev/nvme0n1 /root/sol/ledger
mount /dev/nvme1n1 /root/sol/accounts

vim /etc/fstab

# 自动挂载
/dev/nvme0n1 /root/sol/ledger ext4 defaults 0 0
/dev/nvme1n1 /root/sol/accounts ext4 defaults 0 0

```

#### 2.3. 设置cpu为性能模式
Solana节点对cpu主频要求较高，推荐使用高频cpu并将性能设置为performance模式。


```bash
apt install linux-tools-common linux-tools-$(uname -r)

cpupower frequency-info

cpupower frequency-set --governor performance

watch "grep 'cpu MHz' /proc/cpuinfo"

```

#### 2.4. 安装solana-cli

```shell
# 版本可以根据情况替换
sh -c "$(curl -sSfL https://release.anza.xyz/v2.0.18/install)"
vim /root/.bashrc
export PATH="/root/.local/share/solana/install/active_release/bin:$PATH"
source /root/.bashrc

solana --version

```


#### 2.5. 创建验证者私钥

```shell
solana-keygen new -o validator-keypair.json
```

#### 2.6. 系统调优

1. 修改/etc/sysctl.conf

```shell
vim /etc/sysctl.conf
```
添加如下
```shell
# Increase UDP buffer sizes
net.core.rmem_default = 134217728
net.core.rmem_max = 134217728
net.core.wmem_default = 134217728
net.core.wmem_max = 134217728

# Increase memory mapped files limit
vm.max_map_count = 1000000

# Increase number of allowed open file descriptors
fs.nr_open = 1000000
```

```shell
sysctl -p
```

2. 修改/etc/systemd/system.conf

```shell
vim /etc/systemd/system.conf
```
添加如下

```shell
DefaultLimitNOFILE=1000000
```

```shell
systemctl daemon-reload
```

3. 修改/etc/security/limits.conf
```shell
vim /etc/security/limits.conf
```
添加如下

```shell
# Increase process file descriptor count limit
* - nofile 1000000
```

```shell
ulimit -n 1000000 # 手动设置一下，不然需要重启机器
```

### 2.7. 开启防火墙

```shell
sudo ufw allow 22
sudo ufw allow 8000:8020/tcp
sudo ufw allow 8000:8020/udp
sudo ufw allow 8899 # http 端口
sudo ufw allow 8900 # websocket 端口
sudo ufw allow 11111 # geyser grpc 端口
sudo ufw allow 11112 # geyser prometheus 端口

sudo ufw enable
sudo ufw status

```

### 2.8 创建启动脚本和服务
```shell
vim /root/sol/bin/validator.sh
```
添加如下

```shell
#!/bin/bash

export RUST_LOG=warn

exec agave-validator \
    --identity /root/validator-keypair.json\
    --ledger /root/sol/ledger \
    --accounts /root/sol/accounts \
    --known-validator 7Np41oeYqPefeNQEHSv1UDhYrehxin3NStELsSKCT4K2 \
    --known-validator GdnSyH3YtwcxFvQrVVJMm1JhTS4QVX7MFsX56uJLUfiZ \
    --known-validator DE1bawNcRJB9rVm3buyMVfr8mBEoyyu73NBovf2oXJsJ \
    --known-validator CakcnaRDHka2gXyfbEd2d3xsvkJkqsLw2akB3zsN1D2S \
    --entrypoint entrypoint.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint2.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint3.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint4.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint5.mainnet-beta.solana.com:8001 \
    --expected-genesis-hash 5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d \
    --full-rpc-api \
    --private-rpc \
    --rpc-bind-address 0.0.0.0 \
    --no-voting \
    --rpc-port 8899 \
    --gossip-port 8001 \
    --dynamic-port-range 8000-8020 \
    --wal-recovery-mode skip_any_corrupted_record \
    --limit-ledger-size \
    --account-index program-id \
    --account-index spl-token-mint \
    --account-index spl-token-owner \
    --enable-rpc-transaction-history \
    --init-complete-file /root/init-completed \
    --log /root/solana-rpc.log \
    --log-messages-bytes-limit 1073741824 \
    --geyser-plugin-config /root/yellowstone-config.json
    
    # --geyser-plugin-config 黄石插件的配置文件路径, 如果不需要grpc则去掉
    # --log-messages-bytes-limit 日志文件大小限制 (1G = 1073741824)
    # --rpc-bind-address 0.0.0.0 所有ip都可以访问
    # --private-rpc 不在网络中暴露rpc
    
    
    # 以下参数按需选择添加
	# 务必了解每个参数的功能
	# --rpc-bind-address 0.0.0.0 \
	# --tpu-enable-udp \
	# --only-known-rpc \
	# --rpc-send-default-max-retries 0 \
	# --rpc-send-service-max-retries 0 \
	# --rpc-send-retry-ms 2000 \
	# --minimal-snapshot-download-speed 1073741824 \
	# --maximum-snapshot-download-abort 3 \
	# --rpc-send-leader-count 1500 \
	# --accounts-index-memory-limit-mb 1024000 \
	# --limit-ledger-size 50000000 \
	# --minimal-snapshot-download-speed 1073741824 \
```

```shell
chmod +x /root/sol/bin/validator.sh
```

```shell
vim /etc/systemd/system/sol.service
```

添加如下

```shell

[Unit]
Description=Solana Validator
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
Environment="PATH=/bin:/usr/bin:/root/.local/share/solana/install/active_release/bin"
ExecStart=/root/sol/bin/validator.sh

[Install]
WantedBy=multi-user.target

```

### 2.9. 下载geyser插件
```shell
# 需要与solana-cli版本对照, 否则可能会报错, 参考github: https://github.com/rpcpool/yellowstone-grpc/releases
wget https://github.com/rpcpool/yellowstone-grpc/releases/download/v3.1.0%2Bsolana.2.0.18/yellowstone-grpc-geyser-release-x86_64-unknown-linux-gnu.tar.bz2

apt-get install bzip2

tar -xvjf yellowstone-grpc-geyser-release-x86_64-unknown-linux-gnu.tar.bz2 
```

### 2.10. 配置geyser插件

```shell
vim yellowstone-config.json
```

添加如下

```shell
# 以下端口
# libpath 下载的geyser插件路径
# log 日志等级
# grpc 配置
{
    "libpath": "/root/yellowstone-grpc-geyser-release/lib/libyellowstone_grpc_geyser.so",
    "log": {
        "level": "info"
    },
    "grpc": {
        "address": "0.0.0.0:11111",
        "snapshot_plugin_channel_capacity": null,
        "snapshot_client_channel_capacity": "500_000_000",
        "channel_capacity": "5_000_000",
        "unary_concurrency_limit": 5000,
        "unary_disabled": false,
        "max_decoding_message_size": "16_777_216"
    },
    "prometheus": {
        "address": "0.0.0.0:11112"
    }
}

```

### 2.11. 启动服务

```shell

systemctl start sol
systemctl status sol

```


### 其他


```shell

# 查看服务是否正常运行
tail -f /root/solana-rpc.log
journalctl -u sol -f --no-hostname -o cat
ps aux | grep solana-validator

```
```shell
# 查看同步进度
solana-keygen pubkey /root/validator-keypair.json
solana gossip | grep {pubkey}
solana catchup {pubkey}

# 如果更改了端口
solana catchup --our-localhost {rpc port}

# 如果还是无法查看同步进度
tail -f /root/solana-rpc.log | awk '
/highest-confirmed-slot=/ {
    split($0, a, "highest-confirmed-slot=");
    split(a[2], b, "i");
    confirmed=b[1];
}
/optimistic_slot slot=/ {
    split($0, c, "slot=");
    split(c[2], d, "i");
    latest=d[1];
    if(confirmed != "" && latest != "") {
        printf "已确认slot: %d, 最新slot: %d, 差值: %d\n", 
        confirmed, latest, latest-confirmed;
        fflush();
    }
}'

```

### 快照

如果嫌下载快照数据太慢, 可以提前先下载快照数据, 这里使用的是docker. 参考: https://github.com/c29r3/solana-snapshot-finder
```shell

# -v /root/sol/ledger 是自己的ledger路径

sudo docker run -it --rm \
-v /root/sol/ledger:/solana/snapshot \
--user $(id -u):$(id -g) \
c29r3/solana-snapshot-finder:latest \
--snapshot_path /solana/snapshot \
--min_download_speed 20 \
--threads_count 50 \
--max_snapshot_age 500 \
--measurement_time 60

```

### 遇到的一些问题

#### 追不上slot

1. ledger, accounts 一定要分开挂载到不同的盘, 否则大概率追不上slot, 就算追上了也可能过一段时间就掉下去
2. 保持自己的节点版本与主流版本一致, 否则也可能出现slot追不上的情况, 如果运行一段时间后, 发现slot时常落后, 可以考虑升级到最新的主流版本(记得geyser版本要和新版本保持一致)

#### 装了geyser插件一直重启, 不能成功启动

1. coredumpctl list -r 看看 应该是动态库版本等问题导致进程coredump了
2. 下载geyser的检查插件, 看看配置文件是否有问题
```shell
wget https://github.com/rpcpool/yellowstone-grpc/releases/download/v3.1.0%2Bsolana.2.0.18/config-check-ubuntu-22.04

chmod +x config-check-ubuntu-22.04

./config-check-ubuntu-22.04 --config yellowstone-config.json

```


# 感谢
搭建过程参考了一些资料, 以及solana社区大佬们的热心帮助, 希望本记录能帮助到更多需要搭建solana节点的小伙伴.

# 参考资料
https://chainbuff.gitbook.io/tutorials/solana/da-jian-ji-chu-she-shi
