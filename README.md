# ethernet-joy-kun-connection

X680x0 + イーサネットじょい君 + Raspberry Pi + Wi-Fi LAN + Internet connection setup guide

---

## はじめに

Raspberry Piを簡易DHCPサーバ兼無線LANルーターとして構成し、X680x0実機とイーサネットじょい君から接続するための覚書です。

### なぜ Raspberry Pi か

yunk氏のイーサネットじょい君により、X680x0実機をネットワークに参加させることが容易になりました。

ただ、我が家の場合はX680x0実機の近くに有線ポートを持つWi-Fiルーターが無く、物理的な接続が難しいという課題があります。

このため、X680x0実機の近くで常時稼働している Raspberry Pi の一つをDHCPサーバ兼Wi-Fiルータとして構成して活用してしまおうというのが趣旨です。

また、この構成を取ることでRaspberry Piとftpなどでファイルのやり取りも可能となります。

<img src='images/EthernetJoyKun1.png'/>

---

## 用意するもの

- X680x0実機
- イーサネットじょい君 (https://yunkya2.booth.pm/items/8063175)
- joynetd (https://github.com/yunkya2/joynetd)
- Raspberry Pi Zero2 W (または4B/3B+) *家庭内Wi-Fi LANに接続済みであること
- microUSB Ethernetアダプター (4B/3B+の場合は不要)
- イーサネットストレートケーブル (クロスケーブルでも構いません)

---

## X680x0側設定 (共通)

### CONFIG.SYS

Nereidの推奨設定に従うが、ether_ne.sys の代わりに etherL12.sys を使う。

        FILES     = 50
        BUFFERS   = 99 4096
        LASTDRIVE = Z:
        PROCESS   = 32 10 50
        DEVICE    = \USR\SYS\etherL12.sys

### \etc\protocols
        ip    0     IP
        icmp  1     ICMP
        tcp   6     TCP
        udp   17    UDP

### \etc\services
        ftp-data  20/tcp
        ftp       21/tcp
        telnet    23/tcp
        domain    53/tcp    nameserver
        domain    53/udp    nameserver
        finger    79/tcp    finger

---

### ネットワーク構成

* DNS(Wi-FiルータLAN側アドレス) ... 192.168.11.1
* デフォルトゲートウェイ(Wi-FiルータLAN側アドレス) ... 192.168.11.1
* サブネット(WLAN) ... 192.168.11.0/255.255.255.0
* サブネット(有線Ethernet) ... 192.168.21.0/255.255.255.0
* Raspberry Pi IPアドレス(WLAN) ... 192.168.11.x (DHCP自動取得)
* Raspberry Pi IPアドレス(有線Ethernet) ... 192.168.21.101
* X680x0 Nereid IPアドレス ... 192.168.21.68

### X680x0側設定 (AUTOEXEC.BAT)

設定1,2とは異なるので注意

        SET SYSROOT=C:\
        xip -n2
        ifconfig lp0 up
        ifconfig en0 192.168.21.68 netmask 255.255.255.0 up
        inetdconf +dns 192.168.11.1 +router 192.168.21.1
        
### X680x0側設定 (\etc\hosts)

設定1,2とは異なるので注意

        127.0.0.1       localhost   localhost.local
        192.168.21.68   x68000xvi   x68000xvi.local
        192.168.21.101  raspi       raspi.local

### X680x0側設定 (\etc\network)

設定1,2とは異なるので注意

        127   loopback
        192.168.21  private-net

### Raspberry Pi設定 (/etc/dhcpcd.conf)

        # Example static IP configuration:
        interface eth0
        static ip_address=192.168.21.101/24

### Raspberry Pi設定 (IPルーティングの有効化 と IPv6の無効化)

これ以降の設定を行わない場合は X680x0 - Raspberry Pi 間の peer-to-peer 通信のみとなります。

        sudo vi /etc/sysctl.conf

コメントアウトされている行を有効化

        net.ipv4.ip_forward=1 

以下の行を追加

        net.ipv6.conf.all.disable_ipv6=1 

再起動

        sudo reboot

ipv6の行が出力されないことを確認

        ifconfig

### Raspberry Pi設定 (iptables)

        $ sudo apt install iptables-persistent
        $ sudo iptables –-table nat –-append POSTROUTING --out-interface wlan0 -j MASQUERADE
        $ sudo iptables -t nat -L -v -n
        $ sudo netfilter-persistent save

もし上記設定だけだとルーティングされない場合は以下追加

        $ sudo iptables –-append FORWARD –-in-interface eth0 -j ACCEPT
