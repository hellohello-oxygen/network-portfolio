# 小規模オフィスネットワーク設計・構築

Cisco Packet Tracerを使用した小規模オフィスネットワークの設計・構築ポートフォリオです。

---

## 構成図

> ※ Packet Tracerのキャンバスのスクリーンショットをここに貼り付けてください

---

## 使用技術

| 技術 | 内容 |
|---|---|
| VLAN | 部署ごとのネットワーク分離 |
| Router-on-a-Stick | サブインターフェースによるVLAN間ルーティング |
| DHCP | ルーターによるIPアドレス自動配布 |
| DNS / HTTP | Server0による名前解決とWebサービス |
| 無線LAN | WiFiアクセスポイントによる無線接続 |
| ACL | アクセス制御リストによるセキュリティ設計 |
| NAT/PAT | プライベートIPの隠蔽と外部通信 |

---

## ネットワーク設計

### IPアドレス設計

| 機器 | インターフェース | IPアドレス | 備考 |
|---|---|---|---|
| Router2 | Fa0/0.10 | 192.168.10.1/24 | VLAN10ゲートウェイ |
| Router2 | Fa0/0.20 | 192.168.20.1/24 | VLAN20ゲートウェイ |
| Router2 | Fa0/1 | 203.0.113.2/30 | 外部インターフェース |
| Server0 | Fa0 | 192.168.20.15/24 | 固定IP |

### VLAN設計

| VLAN | 用途 | ネットワーク | ゲートウェイ |
|---|---|---|---|
| VLAN10 | 営業部 | 192.168.10.0/24 | 192.168.10.1 |
| VLAN20 | 総務部・サーバー | 192.168.20.0/24 | 192.168.20.1 |

---

## 実装内容

### 1. VLAN / Router-on-a-Stick

**目的**
部署ごとにネットワークを分離し、情報漏えいリスクを低減するためにVLANを設定しました。
サブインターフェースを使ったRouter-on-a-Stickによりケーブル1本でVLAN間ルーティングを実現しています。

**主な設定**
```
! スイッチ側
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/3
 switchport mode trunk

! ルーター側
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
```

> ※ 動作確認のスクリーンショットをここに貼り付けてください

---

### 2. DHCP

**目的**
PCのIP設定を自動化し、管理コストを削減するためにDHCPを設定しました。
固定IPが必要なServer0は除外アドレスとして設定しています。

**主な設定**
```
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10

ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.20.15

ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.20.15
```

> ※ show ip dhcp bindingのスクリーンショットをここに貼り付けてください

---

### 3. DNS / Webサーバー

**目的**
IPアドレスではなくドメイン名でサーバーにアクセスできるようにするため、
Server0にDNSとWebサーバーを設定しました。

**設定内容**
- DNS：`server.local` → `192.168.20.15` のAレコードを登録
- HTTP：社内ポータルページを作成

> ※ ブラウザでserver.localにアクセスしている画面をここに貼り付けてください

---

### 4. 無線LAN

**目的**
ノートPCやモバイル端末からも社内ネットワークに接続できるよう、
WiFiアクセスポイントを追加しました。

**設定内容**
- SSID：OfficeWiFi
- 認証：WPA2-PSK
- 接続VLAN：VLAN10（営業部）

> ※ LaptopのWiFi接続画面をここに貼り付けてください

---

### 5. ACL（アクセス制御）

**目的**
部署間の不正アクセス防止と、サーバーへの通信を必要最小限に制限するため
ACLを設定しました。

**設計方針**
- 営業部（VLAN10）から総務部（VLAN20）への直接アクセスを拒否
- Server0への通信はHTTP（80番）のみ許可、それ以外は拒否

**主な設定**
```
ip access-list extended VLAN10-TO-VLAN20
 10 permit tcp 192.168.10.0 0.0.0.255 host 192.168.20.15 eq 53
 15 permit udp 192.168.10.0 0.0.0.255 host 192.168.20.15 eq 53
 16 permit tcp 192.168.10.0 0.0.0.255 host 192.168.20.15 eq www
 20 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 30 permit ip any any
```

> ※ show ip access-listsのスクリーンショットをここに貼り付けてください

---

### 6. NAT/PAT

**目的**
内部のプライベートIPアドレスを外部に公開しないようにするため、
NAT/PATを設定しました。複数の端末が1つのグローバルIPを共有します。

**主な設定**
```
ip access-list standard NAT-INSIDE
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255

ip nat inside source list NAT-INSIDE interface FastEthernet0/1 overload
```

> ※ show ip nat translationsのスクリーンショットをここに貼り付けてください

---

## セキュリティ設計の考え方

本構成では以下の3段階でセキュリティを実装しています。

```
段階①：VLAN分離
　部署ごとにネットワークを分離し、横断的なアクセスを制限

段階②：ACL
　VLAN間通信をルールベースで制御し、最小権限の原則を実現

段階③：NAT/PAT
　内部IPを外部に公開しないことで、外部からの直接攻撃を防止
```

---

## 詰まった点と解決方法

| 問題 | 原因 | 解決方法 |
|---|---|---|
| server.localにアクセスできない | ACLがDNS通信も拒否していた | port53（TCP/UDP）を明示的に許可 |
| NATが機能しない | サブインターフェースにip nat insideが未設定 | 各サブインターフェースに個別に設定 |
| WiFi接続できない | 有線NICが残ったままだった | Physical画面でNICを差し替え |

---

## 今後追加したい機能

- [ ] ASAファイアウォールの導入
- [ ] OSPFによる動的ルーティング
- [ ] 冗長化構成（STP / HSRP）
- [ ] VPN接続

---

## 使用ツール

- Cisco Packet Tracer 8.x
