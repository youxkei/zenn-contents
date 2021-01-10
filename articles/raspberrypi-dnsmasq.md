---
title: "夜になったらYouTubeやTwitterに繋がらなくする"
emoji: "💤"
type: "tech"
topics: ["RaspberryPi", "dnsmasq"]
published: false
---
YouTubeやニコニコ動画で永遠と動画を見続けたりTwitterを永遠と見続けてしまって夜ふかししてしまうことがよくあると思います。

そこで、どこのご家庭にも1台は余っているRaspberry Piを使って、夜になったらYouTubeやTwitterに繋がらなくしていきます。

## 方針
繋がらなくする（つまり、ブロッキングする）方法はいろいろとあります。

今回は、Raspberry PiでDNSサーバを立てて、夜になったら設定したドメイン名に対する名前解決をできなくします。
さらに、Raspberry PiでDHCPサーバを立てて設定することで、携帯端末などが無線LANに接続されたときにDNSサーバがRaspberry Piに設定されるようにします。

## 必要なもの
- Raspberry Pi
    - バージョンは何でもいいですが、自分はRaspberry Pi 4を使いました。
- DHCP機能を切れるルータ
    - Raspberry PiをDHCPサーバとして動作させるため、ルータのDHCP機能を切れる必要があります。
