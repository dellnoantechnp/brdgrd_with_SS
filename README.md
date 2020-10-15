### 简介
利用 [brdgrd](https://github.com/NullHypothesis/brdgrd) 可在用户空间随机调整 TCP window size 的特性，保证 Sha￥dow￥sock￥s 及其分支工具能够更健壮的进行数据交互，减少被不明高墙主动探测和 block 的风险。

### 测试环境
* CentOS 7.x
* ssr

### 安装步骤
```bash
sudo yum install -y make gcc glibc  # redhat/centos
sudo apt install make               # ubuntu
sudo apt install gcc                # ubuntu
sudo apt update                     # ubuntu
sudo apt install build-essentials   # ubuntu
# you need have check compile tools

git clone https://github.com/NullHypothesis/brdgrd
cd brdgrd
make -j 4
iptables -A OUTPUT -p tcp --tcp-flags SYN,ACK SYN,ACK --sport $SSPORT -j NFQUEUE --queue-num 0 [--queue-bypass]
./brdgrd
```
> **--queue-bypass** : 当user space程序 brdgrd 没有运行的时候，iptables rule 继续往下匹配，brdgrd 程序退出的时候，不进行流量丢弃操作。

### 注意事项
编译 `brdgrd` 报错，`libnetfilter_queue.h` 头文件缺失：
```
error: libnetfilter_queue/libnetfilter_queue.h: No such file or directory
```
需编译安装如下三个包（`libnfnetlink`、`libmnl`、`libnetfilter_queue`）：
```
wget http://www.netfilter.org/projects/libnfnetlink/files/libnfnetlink-1.0.1.tar.bz2
wget http://www.netfilter.org/projects/libmnl/files/libmnl-1.0.3.tar.bz2
wget http://www.netfilter.org/projects/libnetfilter_queue/files/libnetfilter_queue-1.0.2.tar.bz2

(tar -xf libnfnetlink-1.0.1.tar.bz2 && cd libnfnetlink-1.0.1 && ./configure && make && make install)
(tar -xf libmnl-1.0.3.tar.bz2 && cd libmnl-1.0.3 && ./configure && make && make install)
(tar -xf libnetfilter_queue-1.0.2.tar.bz2 && cd libnetfilter_queue-1.0.2 && ./configure && make && make install)
```
以上三个依赖包安装结束后，就可以正常编译 brdgrd 了。

### 说明
`libnetfilter_queue` 用于将数据包从kernel into user space。
`brdgrd` 只是通过 libnetfilter_queue 获取TCP的 *SYN,ACK* **SYN,ACK** 这两种TCP flag数据包的 `Window size value: **`，并重写用于连接通信确认的 window size，所以对于性能开销很小（对于ss来说，仅匹配 ACK，所以开销非常小）。

经过 brdgrd 过滤过后的流量，能极大限度的逃避 *** 的主动监测，在*铭感十七* 能够最大限度的保护服务端安全运行。

请确保 `iptables.service` | `firewalld.service` 始终运行(enabled)。

### service 
```bash
# RedHat7.x | CentOS 7.x
cat > /usr/lib/systemd/system/brdgrd.service << EOF
[Unit]
Description=brdgrd daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/brdgrd
KillMode=process
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
systemctl enable brdgrd
systemctl start brdgrd
```
