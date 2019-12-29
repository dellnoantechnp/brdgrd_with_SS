### 简介
利用 [brdgrd](https://github.com/NullHypothesis/brdgrd) 可在用户空间随机调整 TCP window size 的特性，保证 Shadowsocks 及其分支工具能够更健壮的进行数据交互，减少被不明高墙主动探测和 block 的风险。

### 测试环境
* CentOS 7.x
* ssr

### 安装步骤
```bash
git clone https://github.com/NullHypothesis/brdgrd
cd brdgrd
make -j 4
iptables -A OUTPUT -p tcp --tcp-flags SYN,ACK SYN,ACK --sport $SSPORT -j NFQUEUE --queue-num 0
./brdgrd
```

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
`libnetfilter_queue` 用于将数据包从内核转移至用户空间。
`brdgrd` 只是通过 libnetfilter_queue 获取TCP SYN,ACK SYN,SCK 这三种TCP握手数据包的 `Window size value: **`，并重写用于握手建立的 window size，所以对于性能开销很小。

经过 brdgrd 过滤过后的流量，能极大限度的逃避***的主动监测，在铭感十七能够最大限度的保护服务端安全运行。

请确保 `iptables.service` | `firewalld.service` 始终运行。

### service 
```bash
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