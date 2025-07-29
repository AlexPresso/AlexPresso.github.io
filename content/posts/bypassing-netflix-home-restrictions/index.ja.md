---
title: "Netflixの同一世帯制限の解除方法"
date: 2025-07-21T22:30:49+02:00
draft: false
---

> "Love is sharing a password." — [Netflix、Twitter投稿 - 2017年3月10日](https://x.com/netflix/status/840276073040371712)

{{< alert >}}
**免責事項:** この記事は教育目的のみです。Netflixの制限を回避することは利用規約に違反します（これで法的な部分は終わりです）。
{{< /alert >}}

2023年、Netflixはパスワード共有の取り締まりを開始しました。
その一環として**Netflixの世帯**という概念が導入されました。
Netflixのような大規模なストリーミングプラットフォームの運営が安価でないことは理解していますが、彼らが選んだ実施方法には全く同意できません。

現代の家族は必ずしも同じ家に住んでいるわけではありません。
子供たちは大学進学で家を離れます。親は異なる都市に住んでいることもあります。パートナーは出張することもあります。まさに私がそのような状況に直面し、以下がその解決方法です。

## Netflixの世帯

Netflixは異なる技術やチェックを使用して世帯を定義しています：
- デバイスフィンガープリント：IPが変わっても、異なるネットワークに接続されていても、デバイスを識別するために使用されます。
- IP + IPQS（IPQualityScore）：実際に世帯のネットワークに接続されているかどうか、プロキシ/VPN/データセンター内のIP範囲ではないか、そしてあなたのIPがIPQSによって不正/位置偽装としてフラグが立てられていないかをチェックします。
- 位置情報 + アクティビティ追跡：デバイスの位置が世帯IPの位置と（おおよそ）同じであるか、または位置情報が世界中で非常に速く変化しているかをチェックします。

デバイスフィンガープリントはそれほど重視する必要はありませんが、「ほぼ」検出されない方法で位置を適切に偽装するには、単なるVPNセットアップ以上のものが必要です。

## ネットワークアーキテクチャ

私の自宅ネットワークスタックは、Raspberry PI上の[K3S](https://k3s.io/)（軽量Kubernetes）クラスターで完全にデプロイおよびオーケストレーションされています：
- [PiHole](https://pi-hole.net/)と[Cloudflared](https://github.com/cloudflare/cloudflared)サイドカー：DHCP、DNSシンク（ドメインブラックリスト/広告ブロッカー）、DoH（DNS-over-HTTPs）用。
- [Wireguard](https://www.wireguard.com/)：スプリットトンネリングモードで設定されたVPNサーバー。モバイルデバイスからPiHole DNSシンクを使用したり、自宅のNASにアクセスしたりするためのものです。

*あなたもKubernetesを使用していて、同じスタックをデプロイしたい場合は、私のhelmチャートを[ここ](https://github.com/AlexPresso/helm.alexpresso.me/tree/main/charts)で見つけることができます（helmリポジトリの追加に関する説明はReadmeにあります）*。

{{< figure src="network-architecture.png" alt="自宅のネットワークアーキテクチャ" title="ネットワークアーキテクチャ" >}}

## 解決策

位置情報を偽装するための最初のステップはVPNを使用することです。
VPNサーバーはNetflix世帯IPとして定義されている自宅ネットワークでホストされる必要があります。
ここから、他の多くの投稿では以下を提案しています：
- 「すべてのクライアントトラフィックをVPNを通してトンネリングするだけ」：これは機能しますが、私が好むやり方ではありません。私はスプリットトンネリング（必要なトラフィックのみをVPN経由でトンネリング）を好みます。
- 「NetflixのIP範囲のみをトンネリングする」：安全ではありません。設定したAmazonの範囲外にNetflixのバックエンド（AWSでホストされている）のIPが変更されるまでの限られた時間しか機能しません。
  また、すべてのデバイスでこれを行うか、デバイスをMDM（モバイルデバイス管理）に登録する必要があります。

私が実装した解決策は、**DNSスプーフィング**と**フォワードプロキシ**に依存しており、次のような利点があります：
- ターゲットIPの変更の影響を受けません（プロキシが自宅ネットワークからドメインを解決します）
- スプリットトンネリングを維持できます
- メンテナンスが最小限で、クライアントデバイスに対して完全に透過的です
- VPN設定を変更せずに、これらのドメインに対して**DNSスプーフィング**を行うだけで、Netflix以外の他のウェブサイトでも機能します。

### DNSスプーフィングとは

DNSスプーフィングは、ハッカーや企業が使用する技術で、DNS要求に対して実際のドメインのIPとは異なるIPを返すことで、トラフィックを傍受/監視するものです。
例えば、公開DNSサーバー（`1.1.1.1` - CloudFlare）を使用してDNS要求を行い、`netflix.com`のIPアドレスを要求すると、Netflixの実際のIPが取得され、クライアントはそれを後続の要求に使用します：
```bash
$ dig netflix.com @1.1.1.1 +short
54.73.148.110
54.155.246.232
18.200.8.190
```
自分自身のDNSサーバーを持っている場合、あなたは制御下にあり、この`netflix.com`要求に対して何でも返すことができます：
```bash
$ dig netflix.com +short
192.168.1.108
```
これが、IP `192.168.1.108`でホストされている透過的な**フォワードプロキシ**を通じてすべてのNetflix要求をルーティングするために私が行っていることです。

### フォワードプロキシとは

**フォワードプロキシ**は、プライベートネットワーク上の接続を受け入れ、それらをパブリックインターネットに転送するネットワークコンポーネントです。クライアントは通常、フォワードプロキシを使用していることを認識していますが（ただし、DNS転送設定では、これはクライアントに対して透過的になる場合があります）。フォワードプロキシは、コンテンツキャッシング、トラフィックフィルタリングなどの追加的な利点を提供し、クライアントの元のIPアドレスを隠すことでクライアントの匿名性を保護するのに役立ちます。

これは**リバースプロキシ**とは逆で、リバースプロキシはパブリックインターネットからの接続を受け入れ、プライベートネットワーク上のリソースにルーティングします。リバースプロキシでは、外部クライアントは通常、プロキシを通して接続していることを認識していないため、複数のサーバー間でのロードバランシング、SSL終端の提供、アプリケーションファイアウォールとしての機能に理想的です。

### どのように機能するか

このアプローチはSNI（**Server Name Indication**）を活用することで可能になります。
<br>クライアントがHTTPSリクエストを行うと、[TLSハンドシェイク](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)を開始します。
最初のパケット（**クライアントハロー**とも呼ばれる）は暗号化されておらず、常にSNIを含んでいます。
このSNIは、透過的なフォワードプロキシ（RAW TCPパススルー）がパケットを復号化せずにどこに転送するかを決定するために重要です。
SNIがなければ、フォワードプロキシはTLS終端ポイントとして機能し、クライアントとサーバーの両方と独自のTLS接続を確立する必要があります。このアプローチでは
クライアントデバイス上で証明書を信頼する必要があり、[証明書ピンニング](https://developer.android.com/privacy-and-security/security-config)を実装するNetflixのようなアプリケーションでは機能しません。

SNIの平文性はプライバシーの懸念を提起するため、ECH（[Encrypted Client Hello](https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni-25)）と呼ばれる新しい標準が登場していますが、これについては後で説明します。

DNSスプーフィングを使用した透過的なフォワードプロキシを使用する際の完全なワークフローは以下の通りです（明確さを保つためにVPNサーバーは省略されています）：

{{< figure src="full-workflow.png" alt="完全なワークフロー" title="完全なワークフロー" >}}
1. デバイスはドメインIP（例：`netflix.com`）を解決するためにDNS要求を行います
2. PiHole DNSサーバーはフォワードプロキシのIPで応答します
3. デバイスは接続を開始し、**クライアントハロー**パケットをフォワードプロキシに送信します
4. フォワードプロキシはクライアントハローのSNIを読み取り、ドメインの実際のIPを解決するための新しいDNS要求を行います
5. 実際の（スプーフィングされていない）DNSサーバーが実際のIPで応答します
6. フォワードプロキシは生のTCPパケットをターゲットホストに転送します

この方法で追加されたIPアドレスは、再起動後も保持されないことに注意してください。これらを永続的にするには、ネットワーク設定システムで設定する必要があります。
ディストリビューションによって、これは`ifupdown`、`netplan`、`NetworkManager`または`systemd-networkd`にある場合があります。
<br>Raspberry Piでは、これらのコマンドを`/etc/rc.local`に追加します。
<details>
    <summary>/etc/rc.local</summary>

```Bash
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
printf "My IP address is %s\n" "$_IP"
fi

ip addr add 192.168.1.108/24 dev eth0
ip -6 addr add fd12:caf3:babe::1/64 dev eth0

exit 0
```
</details>

#### フォワードプロキシの設定

この目的のために利用できる数多くのプロキシ技術があります。
私は[NGinX](https://nginx.org/en/)に精通しているため、これを好みますが、他のソリューションも機能します。

注目すべき重要な機能は、プロキシによって送信されるヘッダーを制御する能力です。通常、
（[RFC 7239](https://www.rfc-editor.org/rfc/rfc7239.html#section-4)と[RFC 7230](https://www.rfc-editor.org/rfc/rfc7230#section-5.7.1)のおかげで）すべてのプロキシは`Via`または`Forwarded`ヘッダーを追加する必要があり、これらはしばしばハードコードされています。
そのようなヘッダーがあると、プロキシはNetflixによって簡単に検出される可能性があります。幸運なことに、NGinXは最小限のヘッダー哲学を守り、デフォルトではそれらを追加しません。

前述したように、私はすべてをKubernetesクラスターにデプロイしました。この`nginx-sni-proxy`のhelmチャートは[ここ](https://github.com/AlexPresso/helm.alexpresso.me/tree/main/charts/nginx-sni-proxy)で見つけることができます。
<br>Kubernetesを使用していない場合は、次の設定でdocker NGinXコンテナを実行できます：
<details>
    <summary>/etc/nginx/nginx.conf</summary>

```Bash
worker_processes auto;

events {
  worker_connections 1024;
}

stream {
  # TLS ClientHelloからのSNI抽出を有効にする
  ssl_preread on;

  resolver 1.1.1.1 ipv6=on valid=10s;

  log_format basic '[$time_local] $remote_addr -> $ssl_preread_server_name:$server_port '
                   'proto=$protocol bytes_sent=$bytes_sent bytes_received=$bytes_received duration=${session_time}s';

  server {
    listen 443;
    proxy_pass $ssl_preread_server_name:443;
    access_log /dev/stdout basic;
  }
}

http {
  resolver 1.1.1.1 ipv6=on valid=10s;

  log_format basic '[$time_local] $remote_addr -> $host:$server_port '
                   'proto=$server_protocol bytes_sent=$bytes_sent bytes_received=$request_length duration=${request_time}s';

  server {
    listen 80;
    access_log /dev/stdout basic;

    location / {
      proxy_pass http://$host;
    }
  }
}
```
</details>

この設定は`http`と`https`の両方のプロトコルに対するフォワードプロキシを実行します。`stream`ブロックはRAW TCPパススルーを可能にし、TLS終端なしでSNIベースのルーティングを実現します。

#### DNSスプーフィングの設定

フォワードプロキシがデプロイされたので、NetflixドメインエントリをDNSサーバーに追加し、フォワードプロキシのIP（私の場合は`192.168.1.108`と`fd12:caf3:babe::1`）を指すようにする必要があります。

もちろん、すべてのドメインとサブドメインに対して手動でレコードを追加するのは面倒で安全ではないでしょう。この目的のために、ワイルドカードドメインエントリ（例：`*.netflix.com`、netflixの任意のサブドメインに一致）が完璧ですが、PiHoleは
ネイティブにそれをサポートしていません。これは理解できることです。ワイルドカードドメインレコードは主にDNSスプーフィングに使用されるものであり、一般的には防止したいものだからです。

幸いなことに、PiHoleは[dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html)に依存しており、これは「ワイルドカード」ドメインエントリを処理します：
<details>
    <summary>/etc/dnsmasq.d/1-additional.conf</summary>

```Bash
address=/netflix.net/192.168.1.108
address=/netflixstudios.com/192.168.1.108
address=/netflix.com/192.168.1.108
address=/fast.com/192.168.1.108
address=/netflix.ca/192.168.1.108
address=/netflix.com/192.168.1.108
address=/netflix.net/192.168.1.108
address=/netflixinvestor.com/192.168.1.108
address=/nflxext.com/192.168.1.108
address=/nflximg.com/192.168.1.108
address=/nflximg.net/192.168.1.108
address=/nflxsearch.net/192.168.1.108
address=/nflxso.net/192.168.1.108
address=/nflxvideo.net/192.168.1.108
address=/netflixdnstest*.net/192.168.1.108
address=/amazonaws.com/192.168.1.108

address=/netflix.net/fd12:caf3:babe::1
address=/netflixstudios.com/fd12:caf3:babe::1
address=/netflix.com/fd12:caf3:babe::1
address=/fast.com/fd12:caf3:babe::1
address=/netflix.ca/fd12:caf3:babe::1
address=/netflix.com/fd12:caf3:babe::1
address=/netflix.net/fd12:caf3:babe::1
address=/netflixinvestor.com/fd12:caf3:babe::1
address=/nflxext.com/fd12:caf3:babe::1
address=/nflximg.com/fd12:caf3:babe::1
address=/nflximg.net/fd12:caf3:babe::1
address=/nflxsearch.net/fd12:caf3:babe::1
address=/nflxso.net/fd12:caf3:babe::1
address=/nflxvideo.net/fd12:caf3:babe::1
address=/netflixdnstest*.net/fd12:caf3:babe::1
address=/amazonaws.com/fd12:caf3:babe::1
```
</details>

または[helmチャート](https://github.com/AlexPresso/helm.alexpresso.me/tree/main/charts/pihole)を使用する場合：

<details>
    <summary>values.yaml</summary>

```yaml
# ...
additionalDnsmasq:
  # IPv4
  - "address=/netflix.net/192.168.1.108"
  - "address=/netflixstudios.com/192.168.1.108"
  - "address=/netflix.com/192.168.1.108"
  - "address=/fast.com/192.168.1.108"
  - "address=/netflix.ca/192.168.1.108"
  - "address=/netflix.com/192.168.1.108"
  - "address=/netflix.net/192.168.1.108"
  - "address=/netflixinvestor.com/192.168.1.108"
  - "address=/nflxext.com/192.168.1.108"
  - "address=/nflximg.com/192.168.1.108"
  - "address=/nflximg.net/192.168.1.108"
  - "address=/nflxsearch.net/192.168.1.108"
  - "address=/nflxso.net/192.168.1.108"
  - "address=/nflxvideo.net/192.168.1.108"
  - "address=/netflixdnstest*.net/192.168.1.108"
  - "address=/amazonaws.com/192.168.1.108"
  # IPv6
  - "address=/netflix.net/fd12:caf3:babe::1"
  - "address=/netflixstudios.com/fd12:caf3:babe::1"
  - "address=/netflix.com/fd12:caf3:babe::1"
  - "address=/fast.com/fd12:caf3:babe::1"
  - "address=/netflix.ca/fd12:caf3:babe::1"
  - "address=/netflix.com/fd12:caf3:babe::1"
  - "address=/netflix.net/fd12:caf3:babe::1"
  - "address=/netflixinvestor.com/fd12:caf3:babe::1"
  - "address=/nflxext.com/fd12:caf3:babe::1"
  - "address=/nflximg.com/fd12:caf3:babe::1"
  - "address=/nflximg.net/fd12:caf3:babe::1"
  - "address=/nflxsearch.net/fd12:caf3:babe::1"
  - "address=/nflxso.net/fd12:caf3:babe::1"
  - "address=/nflxvideo.net/fd12:caf3:babe::1"
  - "address=/netflixdnstest*.net/fd12:caf3:babe::1"
  - "address=/amazonaws.com/fd12:caf3:babe::1"
# ...
```
</details>

この時点で、すべてのローカルデバイスはすでにフォワードプロキシを通じてNetflixにアクセスしています。
フォワードプロキシの`stdout`ログ：
```log
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=24438 bytes_received=991 duration=2.510s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=1141422 bytes_received=1202 duration=3.895s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=572895 bytes_received=1635 duration=5.021s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=67514 bytes_received=991 duration=2.431s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=118640 bytes_received=991 duration=3.550s
10.42.0.1 -> ipv4-XXXX-XXXXXX-ix.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=91456 bytes_received=1444 duration=0.982s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X-isp.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=342429 bytes_received=3108 duration=4.524s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=112581 bytes_received=1834 duration=3.360s
10.42.0.1 -> ios.prod.ftl.netflix.com:443 proto=TCP bytes_sent=5996 bytes_received=2087 duration=0.702s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=27082 bytes_received=1214 duration=2.977s
```

#### Wireguardの設定

このセットアップをローカルネットワーク外から機能させるには、各VPNクライアントが少なくとも以下をトンネリングするようにWireGuardを設定する必要があります：
- PiHoleへのDNSリクエスト
- フォワードプロキシに向かうパケット

以下は非常に基本的なスプリットトンネリング設定です。特定のネットワーク要件に応じてカスタマイズしてください（[wireguard公式ドキュメント](https://www.wireguard.com/quickstart/)）。

特定のホストからの/へのルーティングを防止/許可するためのファイアウォール設定を行いたい場合も、`PostUp`スクリプトを使用して可能です。例は[helmチャートの値](https://github.com/AlexPresso/helm.alexpresso.me/blob/main/charts/wireguard/values.yaml#L37)で見つけることができます。

<details>
    <summary>サーバー wg0.conf</summary>

```ini
[Interface]
Address = 10.6.0.1/24, fd10:6::1/64
PrivateKey = # サーバーの秘密鍵
ListenPort = 51820
Table = off
DNS = 192.168.1.107 # PiHole DNSサーバーのIP
MTU = 1420

[Peer]
PublicKey = # ピアの公開鍵
AllowedIPs = 10.6.0.2/32, fd10:6::2/128
PersistentKeepAlive = 25
PresharedKey = # ピアのPSK
# ...
```
</details>

<details>
    <summary>クライアント wg0.conf</summary>

```ini
[Interface]
Address = 10.6.0.2/32, fd10:6::2/128
PrivateKey = # クライアントの秘密鍵
DNS = 192.168.1.107  # PiHole DNSサーバーのIP
MTU = 1420

[Peer]
PublicKey = # サーバーの公開鍵
Endpoint = # あなたのDynDNSまたは自宅の静的IP :51820
AllowedIPs = 192.168.1.107/32, 192.168.1.108/32, fd12:caf3:babe::1/128 # PiHole + フォワードプロキシのトラフィックのみをトンネリング
PersistentKeepAlive = 25
PresharedKey = # ピアのPSK
```
</details>

*クライアント設定の`AllowedIPs`設定が重要であることに注意してください - これはどのトラフィックがVPNを通じてルーティングされるかを決定します。ここでは、PiHoleとフォワードプロキシのトラフィックのみをトンネリングしています。*

## TLSv1.3とECHに関する注記

前述の通り、平文のCH（**クライアントハロー**）があることはプライバシーに対する重大な脅威です。
ネットワークを監視している人は誰でも、SNI（**サーバー名表示**）を読むだけで、あなたが訪問しているウェブサイトのドメイン名を知ることができます。

ECH（**暗号化クライアントハロー**）はまだ[ドラフト](https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni-25)状態ですが、標準になることを意図しています。[ロシア](https://portal.noc.gov.ru/ru/news/2024/11/07/%D1%80%D0%B5%D0%BA%D0%BE%D0%BC%D0%B5%D0%BD%D0%B4%D1%83%D0%B5%D0%BC-%D0%BE%D1%82%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D1%8C%D1%81%D1%8F-%D0%BE%D1%82-cdn-%D1%81%D0%B5%D1%80%D0%B2%D0%B8%D1%81%D0%B0-cloudflare/)のような一部の国々は
すでにその使用を検閲しています。暗号化されたSNIを監視するには、直接的なトラフィック傍受（**TLS終端ポイント**）が必要になるためです。

さて、あなたはこう考えるかもしれません：
> 「なるほど、ECHは良いものだけど、それはNetflixをバイパスするこのテクニックがもう機能しなくなるということ？」

簡単な答え：はい、ですが、それなしでもSNIのような転送を実現することは可能です。

完全な記事はコーディングが終わったら書きます。
それまでは、世界中からNetflixを楽しんでください。
