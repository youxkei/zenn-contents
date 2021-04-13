---
title: "夜になったらYouTubeやTwitterに繋がらなくして睡眠時間を確保する"
emoji: "💤"
type: "tech"
topics: ["RaspberryPi", "dnsmasq"]
published: true
---
YouTubeやニコニコ動画で永遠と動画を見続けてしまったりTwitterを永遠と見続けてしまったりして夜ふかししてしまうことがよくあると思います。

そこで、どこのご家庭にも1台は余っているRaspberry Piを使って、夜になったらYouTubeやTwitterに繋がらなくすることで睡眠時間を確保していきます。

繋がらなくすると言っても方法はいろいろとありますが、今回はRaspberry PiでDNSサーバを立てて、夜になったら設定したドメイン名に対する名前解決をできなくします。

さらに、Raspberry PiでDHCPサーバを立てていい感じに設定することで、PCだけではなく携帯端末からも繋がらなくします。

DNSサーバとDHCPサーバにはdnsmasqを使います。

# 今回使った機器
- Raspberry Pi
    - Raspberry Pi 4を使いました。
- 「DHCP機能」と「RAによるDNSサーバの通知機能」を切ることができるルータ
    - Raspberry PiをDHCPサーバとして動作させるため、ルータのDHCP機能を切る必要があります。
    - さらに、IPv6のRAによってDNSサーバのIPv6アドレスが通知されると困るので、この機能も切る必要があります。
        - RAによるIPv6アドレス割り当て自体はDHCPと競合しないので、機能を切る必要はありません。

# 手順

## Raspberry PiにOSを入れる
何はともあれまずはRaspberry PiにOSをインストールします。

今回はUbuntu Server 20.04.1 LTSを使ったので、以下の手順ではそれを前提とします。

インストールは、基本的には[How to install Ubuntu Server on your Raspberry Pi](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi)に書いてあるとおりにやれば大丈夫です。

大丈夫なんですが、Raspberry PiのIPアドレスを調べる手順で`arp`コマンドだとすぐにIPアドレスが出てきませんでした。

そこで、`arp-scan`コマンドを使ってRaspberry PiのIPアドレスを調べました。

```bash
sudo apt install arp-scan
sudo arp-scan -l
```

OSのインストールが終わったら、適宜アップグレードなどをしておきます。

## Raspberry PiのIPアドレスを固定する
Raspberry PiをDNSサーバにするので、IPアドレスが固定されている必要があります。

netplanの設定を書いて、IPアドレスを固定していきます。

```yaml:/etc/netplan/99-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.10.3/24
      gateway4: 192.168.10.1
      nameservers:
        addresses:
          - 192.168.10.1
```

我が家のLANではルータのIPアドレスは`192.168.10.1`でRaspberry PiのIPアドレスは`192.168.10.3`に設定しています。

ルータやRaspberry PiのIPアドレスは適宜読み替えてください。

## dnsmasqを導入する
dnsmasqをインストールして設定します。

```bash
sudo apt install dnsmasq
```

```md:/etc/dnsmasq.d/conf
listen-address=192.168.10.3
bind-interfaces
no-hosts
max-ttl=30

dhcp-range=192.168.10.128,192.168.10.254,24h
dhcp-option=option:netmask,255.255.255.0
dhcp-option=option:router,192.168.10.1
dhcp-option=option:dns-server,192.168.10.3
dhcp-leasefile=/tmp/dnsmasq.leases
```

dnsmasqの設定のそれぞれの項目については以下の通りです。

- `listen-adress`
    - Raspberry Piのアドレスを指定します。
        - 指定しないとdnsmasqは`0.0.0.0:53`をlistenしようとするので、systemd-resolvedと競合します。
- `bind-interfaces`
    - 非常に重要な設定です。これに気づかずに1時間くらいハマりました・・・
    - この設定を有効にしなかった場合、dnsmasqは`listen-address`に何を指定しても`0.0.0.0:53`でlistenしようとします。
        - つまり、systemd-resolvedとの競合を回避するために必須です。
- `no-hosts`
    - 名前解決の際に`/etc/hosts`を見ないようにします。
- `max-ttl`
    - dnsmasqが返すDNSレスポンスのTTLの最大値を設定します。
        - DNSによる名前解決の結果はこのTTLの期間だけPCや携帯端末にキャッシュされるので、それなりに短くする必要があります。
            - 今回は30秒としました。
- `dhcp-*`
    - DHCPの設定です。
        - デフォルトゲートウェイをルータのIPアドレスである`192.168.10.1`に、DNSサーバのIPアドレスを自身の`192.168.10.3`に設定しています。

## 名前解決ができないようにするドメインを設定する
```md:/etc/dnsmasq.hosts
address=/youtube.com/googlevideo.com/
address=/nicovideo.jp/dmc.nico/
address=/twitter.com/twimg.com/
```

`address`は`/<domain>[/<domain>...]/<ipaddr>`という構文で、`<domain>`を`<ipaddr>`に名前解決するように設定できます。

今回は名前解決して欲しくないので`<ipaddr>`を空文字列にしました。
`<ipaddr>`を空文字列にすると、`<domain>`に対してNXDOMAINを返すように設定できます。

`address`は1行で`/example.com/youtube.com/nicovideo.jp/`のように複数のドメインを指定できるので、ひとまず関連があるドメインをまとめました。


`address`は設定したドメインのサブドメインについても効果があります。
ある特定のサブドメインの名前解決はできるようにしたい場合、次のように`server`を設定します。

```
address=/nicovideo.jp/
server=/live.nicovideo.jp/#
```

さて、このドメインの設定は`/etc/dnsmasq.hosts`に置かれているので、通常であればこの設定は読み込まれません。
dnsmasqが設定ファイルを読みに行くディレクトリに`/etc/dnsmasq.hosts`へのシンボリックリンクを張ったり張らなかったりすることで、この設定を有効にしたり無効にしたりします。

## 夜になったらdnsmasqの設定を変えるようにする
```bash:/etc/cron.d/dnsmasq
0 23  *  *  * root      ln -s /etc/dnsmasq.hosts /etc/dnsmasq.d/hosts; systemctl restart dnsmasq
0  6  *  *  * root      rm                       /etc/dnsmasq.d/hosts; systemctl restart dnsmasq
```

毎日23時に`/etc/dnsmasq.hosts`のシンボリックリンクを`/etc/dnsmasq.d/hosts`に作成するようにします。
こうすることで、23時に`/etc/dnsmasq.hosts`の設定が有効になります。

毎日6時に`/etc/dnsmasq.d/hosts`を消すことで、`/etc/dnsmasq.hosts`の設定が無効になります。

## ルータの設定を変える
ルータの設定を以下のように変更します。

- DHCP機能をオフにする。
    - DHCPv4とDHCPv6の両方をオフにします。
- IPv6のステートレス自動設定で、DNSサーバのIPv6アドレスを通知しないようにする。
    - DNSサーバのIPv6アドレスさえ通知されなければいいので、ステートレス自動設定機能はオフにする必要はありません。

# まとめ
dnsmasqを使って、夜になったらYouTubeやニコニコ動画やTwitterなどの睡眠妨害サイトに繋がらなくすることで、睡眠時間を確保することができました。
