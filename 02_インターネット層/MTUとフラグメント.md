---
tags: [ネスペ, MTU, MSS, フラグメント, PMTUD, インターネット層]
---

# MTUとフラグメント まとめ（ネスペ出題観点）

出題度：◎ 午後で記述・穴埋め・計算として頻出／○ 午前・午後小問で出る／△ 深追い不要

PPPoE、IPsec、VXLAN、GREなど、ノートごとに出てくる「ヘッダが増えてMTUが小さくなる」話を横断でまとめる。

## 1. 用語（◎）

| 用語 | 意味 | Ethernetの代表値 |
|---|---|---|
| **MTU（Maximum Transmission Unit）** | 1回に送れる**IP（Internet Protocol）パケットの最大サイズ**（IPヘッダを含む） | **1500バイト** |
| **MSS（Maximum Segment Size）** | TCP（Transmission Control Protocol）が1回に運ぶ**データ部分の最大サイズ** | **1460バイト**（1500 − IPヘッダ20 − TCPヘッダ20） |
| フレームの最大長 | Ethernetフレーム全体 | 1518バイト（VLAN（Virtual LAN）タグ付きは1522） |
| ジャンボフレーム | 1500より大きいMTU（9000バイト前後）。データセンターなどで使う | − |

- **MSS ＝ MTU − 40**（IPv4）、**MTU − 60**（IPv6、IPv6ヘッダは40バイト）
- 経路の中で**一番小さいMTU（経路MTU）**より大きいパケットは、そのままでは通れない

## 2. フラグメント（分割）（○）

### IPv4

- MTUより大きいパケットを、ルータが**複数の断片（フラグメント）に分割**して送る
- IPヘッダの項目
  - **識別子（ID）**：どの元のパケットの断片か
  - **フラグ**：**DF（Don't Fragment：分割禁止）**、**MF（More Fragments：後ろに続きがある）**
  - **フラグメントオフセット**：元のパケットの何番目の部分か
- **組み立て直すのは宛先の端末だけ**（途中のルータは組み立てない）
- 分割の問題点
  - ルータの処理の負荷が増える
  - **断片が1つでも失われると、全体が使えない**（再送はTCPが全体をやり直す）
  - **2番目以降の断片にはTCP・UDP（User Datagram Protocol）のヘッダがない** → FW（Firewall）・NAPT（Network Address Port Translation）がポート番号を見られず、正しく扱えないことがある
- **DFビットが立っていると分割できない** → ルータはパケットを捨て、**ICMP（Internet Control Message Protocol）のType 3 / Code 4**を送信元に返す

### IPv6

- **途中のルータは分割しない**。分割するのは**送信元だけ**（拡張ヘッダのフラグメントヘッダを使う）
- 大きすぎるパケットは捨てられ、**ICMPv6のPacket Too Big**が返る
- **最小MTUは1280バイト**（すべてのリンクで1280以上が保証される）

## 3. ヘッダが増える技術と、小さくなる量（◎）

| 技術 | 増えるヘッダ（目安） | 通せるIPパケットの大きさ（元のMTUが1500の場合） | ノート |
|---|---|---|---|
| **PPPoE（Point-to-Point Protocol over Ethernet）** | **8バイト**（PPPoE 6＋PPP 2） | **1492**（フレッツ網の推奨は1454） | [[PPP]] |
| **GRE（Generic Routing Encapsulation）** | **24バイト**（新しいIPヘッダ20＋GRE 4） | 1476 | [[IPsec]] |
| **IPsec（Security Architecture for Internet Protocol）（ESP（Encapsulating Security Payload）、トンネルモード）** | **数十バイト**（新しいIPヘッダ、ESPヘッダ・トレーラ、認証データ。暗号方式で変わる） | 1400前後に設定することが多い | [[IPsec]] |
| NAT-T（NAT Traversal） | ＋8バイト（UDPヘッダ） | さらに小さくなる | [[IPsec]] |
| GRE over IPsec | GRE＋IPsecの両方 | さらに小さくなる | [[IPsec]] |
| **VXLAN（Virtual eXtensible LAN）** | **約50バイト**（外側のEthernet・IP・UDP・VXLANヘッダ） | アンダーレイのMTUを**1550以上**に広げるのが一般的 | [[仮想化]] |
| VLANタグ（IEEE802.1Q） | 4バイト（フレームが1522になる） | IPのMTUは1500のまま（スイッチが1522のフレームを扱えればよい） | [[VLAN]] |

- 問われ方：「トンネルインタフェースに設定するMTU・MSSの値を求めよ」→ **1500 − 増えるヘッダ**、MSSはさらに40を引く
- 例：PPPoEのフレッツ網 → MTU 1454、**MSS 1414**

## 4. 大きいパケットが通らないときに起きること（◎）

### PMTUDとPMTUDブラックホール

- **PMTUD（Path MTU Discovery）**：送信元がDFビットを立てて送り、ルータから返る**ICMP（Type 3 / Code 4、IPv6はPacket Too Big）の中のMTU値**で経路MTUを知り、サイズを下げて送り直す
- **途中のFWがこのICMPを遮断している**と、送信元は経路MTUを知ることができず、**大きいパケットを送り続けて捨てられ続ける** → **PMTUDブラックホール**
- 症状（午後の定番）
  - **pingや小さいデータは通るのに、大きいデータだけ失敗する**
  - **Webページの一部（画像など）が表示されない、途中で止まる**
  - **TCPの接続（3ウェイハンドシェイク）はできるのに、データの転送が始まらない**
- 記述の型：「ICMPのフラグメント要求が経路上で破棄され、送信元が経路MTUを検出できないため、大きなパケットが破棄され続けるから」
- → [[ICMP]]

### 対策（◎）

| 対策 | 内容 | 効く範囲 |
|---|---|---|
| **MSSクランプ** | ルータが通過する**TCPのSYN（Synchronize）のMSS値を小さく書き換える** → 端末が最初から小さいサイズで送る | **TCPだけ**（UDPには効かない） |
| **トンネルのMTUを小さく設定** | トンネルインタフェースのMTUを、ヘッダの分だけ小さくする | ルータが分割するか、ICMPを返す |
| **ICMPを許可** | FWでICMP Type 3 / Code 4（IPv6はPacket Too Big）を通す | PMTUDが正しく動く |
| **下の層のMTUを大きく** | VXLANのアンダーレイなど、運ぶ側のMTUを広げる（ジャンボフレーム） | ヘッダが増えても1500のパケットをそのまま運べる |

- 記述の型（MSSクランプ）：「ルータでTCPのSYNパケットに含まれるMSSの値を、トンネルのMTUに合わせた値に書き換える」
- **UDPの大きなパケット**（DNS（Domain Name System）のEDNS0（Extension Mechanisms for DNS version 0）の大きい応答、VPN（Virtual Private Network）の中を通るUDPの通信）にはMSSクランプが効かない → MTUの設定、ICMPの許可で対応する
- → [[TCP]]

## 5. 覚え方

- **MTUは「IPパケットの大きさの上限（1500）」、MSSは「TCPの中身の上限（1460）」**：MSS ＝ MTU − 40
- **ヘッダの増加量は「PPPoE 8、GRE 24、VXLAN 約50、IPsec 数十」**
- **フレッツは「1454と1414」**
- **IPv4はルータが分割できる（DFが立っていなければ）、IPv6は送信元だけ。最小1280**
- **「小さいのは通るのに大きいのだけダメ」はPMTUDブラックホール** → MSSクランプ、ICMPを通す
- **MSSクランプはTCPだけ**

## 6. 勉強の優先順位

1. PMTUDブラックホールの症状・原因・対策を30〜40字で書けるようにする
2. MTUとMSSの関係（−40）と、技術ごとのヘッダの増加量からMTU・MSSを求める計算
3. MSSクランプの仕組みと、UDPには効かないこと
4. IPv4とIPv6の分割の違い（DFビット、ルータは分割しない、1280）
5. フラグメントの問題点（1つ失うと全体が無効、2番目以降にポート番号がない）
6. IPヘッダのフラグ・オフセットの細かい値は午前で確認する程度でよい

> [!tip] 得点につながる覚え方
> 午後で**「一部の通信だけ」「大きいデータだけ」「接続はできるのに転送が止まる」**と書かれていたら、まずMTUを疑う。構成図の中で**ヘッダが増える区間（PPPoE、IPsec、GRE、VXLAN）**を探し、その区間のMTU・MSSの設定と、ICMPを遮断していないかを確認する。

## 7. 関連

- [[ICMP]]（PMTUD、Type 3 / Code 4）
- [[TCP]]（MSS、MSSクランプ）
- [[PPP]]（PPPoEのMTU）
- [[IPsec]]（ESP、GRE over IPsec、NAT-T）
- [[仮想化]]（VXLANのMTU）
- [[IPv6アドレス・NDP]]（IPv6のヘッダ、最小MTU）
- [[ポート番号一覧]]
