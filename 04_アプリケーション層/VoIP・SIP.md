---
tags: [ネスペ, VoIP, SIP, RTP, IP電話, アプリケーション層]
---

# VoIP・SIP まとめ（ネスペ出題観点）

出題度：◎ 午後で記述・穴埋めとして頻出／○ 午前・午後小問で出る／△ 深追い不要

## 1. VoIPの基本（○）

- **VoIP（Voice over IP）**：音声をIP（Internet Protocol）ネットワークで運ぶ技術（IP電話）
- 2種類の通信に分かれる

| 役割 | プロトコル | 内容 |
|---|---|---|
| **呼制御（シグナリング）** | **SIP（Session Initiation Protocol）** | 電話をかける・呼び出す・切る。相手の場所を探す |
| 通話内容の情報 | **SDP（Session Description Protocol）** | SIPの中に書く。**音声を受け取るIPアドレス・ポート番号**、使う符号化方式（コーデック） |
| **音声そのもの** | **RTP（Real-time Transport Protocol）** | 音声データを運ぶ（UDP（User Datagram Protocol）） |
| 品質の報告 | RTCP（RTP Control Protocol） | 遅延・損失を報告する（△） |

- SIPのポート：**UDP／TCP（Transmission Control Protocol） 5060**（TLS（Transport Layer Security）で暗号化すると5061）
- RTPのポート：**SDPで決めた番号**（毎回変わる）

## 2. SIPの登場人物と流れ（○）

- **SIPサーバ**
  - **レジストラ**：電話機の「今このIPアドレスにいる」という**登録（REGISTER）**を受け付ける
  - **プロキシサーバ**：発信の要求（INVITE）を相手まで中継する
- 発信の流れ

```
発信側の電話機          SIPサーバ           着信側の電話機
   │── INVITE（SDP）──→│── INVITE ──→│
   │←─ 100 Trying ────│              │
   │←─ 180 Ringing ───│←─ 180 Ringing│  （呼び出し中）
   │←─ 200 OK（SDP）──│←─ 200 OK ────│  （応答した）
   │── ACK ──────────────────→│
   │══════ RTP（音声：電話機同士で直接）══════│
   │── BYE ──────────────────→│  （切断）
   │←─ 200 OK ─────────────────│
```

- **SIP（呼制御）はSIPサーバを経由するが、RTP（音声）は電話機同士で直接やり取り**することが多い → **経路が別々**になる
- SIPの応答コードは**HTTP（Hypertext Transfer Protocol）に似ている**（1xx：途中経過、2xx：成功、4xx：要求の誤り など）
- 午前の定番：「SIPの役割」→ 呼制御（セッションの確立・変更・終了）

## 3. 音声の品質（◎）

- 音声は**遅延・揺らぎ（ジッタ）・パケット損失に弱い**。遅れて届いたデータは使えないので**再送しない（UDPを使う）**
- 目安：片方向の遅延は150ms以内
- **ジッタバッファ**：受信側で少し溜めてから再生し、揺らぎを吸収する（その分、遅延が増える）
- 対策：**QoS（Quality of Service）**で音声を優先する（DSCP（Differentiated Services Code Point）をEF（Expedited Forwarding）に）、**音声用VLAN（Virtual LAN）**を分ける
- **必要な帯域の計算**（G.711、20ms間隔）：1通話で片方向 **約80kbps（IPレベル）** → 同時通話数を掛ける
- → [[QoS]]

## 4. 午後で問われるテーマと記述の型

### ◎ NAPTを越えられない問題

- SDPには**電話機のIPアドレスとポート番号**が書かれている → **NAPT（Network Address Port Translation）**はアプリケーションのデータの中を書き換えないので、**プライベートアドレスのまま相手に届き、音声が届かない**（FTP（File Transfer Protocol）と同じ問題）
- 症状：**呼び出し・応答はできるのに、音声が聞こえない（片方向だけ聞こえる）**
- 対策
  - **SIP ALG（Application Level Gateway）**：ルータ・FW（Firewall）がSIP・SDPの中のアドレスを書き換え、RTPの通信を一時的に許可する
  - **SBC（Session Border Controller）**：境界でSIPとRTPを中継し、アドレスを変換する専用の装置
  - STUN（Session Traversal Utilities for NAT）など、端末がNAPTの外側のアドレスを知る仕組み（△）
- 記述の型：「SDPに記載された端末のプライベートIPアドレスがNAPTで変換されず、相手がそのアドレス宛てにRTPパケットを送信するため、音声が届かないから」
- → [[NAT・NAPT]]、[[FTP]]

### ○ FWのルール

- SIP：UDP／TCP 5060
- RTP：**ポート番号が毎回変わる** → 広い範囲を開けるか、**SIPの中身を見て必要なポートだけを一時的に開ける**機能（ALG）を使う

### ○ IP電話機の接続

- **PoE（Power over Ethernet）**でスイッチから電力を供給する（電源工事が不要）
- **音声VLAN**：電話機の先にPCをつなぐ構成で、電話機の通信とPCの通信を別のVLANに分ける（音声だけ優先しやすくする）
- 電話機の設定（SIPサーバのアドレス）をDHCP（Dynamic Host Configuration Protocol）のオプションで配る

### ○ クラウドPBX・SIPトランク

- 社内のIP-PBX（Private Branch Exchange）の代わりに、クラウドのPBXサービスを使う → インターネット経由のSIPとRTPの品質・FWの設定が論点になる

## 5. 覚え方

- **SIPは「電話の取り次ぎ（呼び出し・切断）」、RTPは「声そのもの」**、SDPは「声の届け先の住所」
- **流れは「INVITE → 100 → 180（呼び出し中）→ 200（応答）→ ACK（Acknowledgement） → 会話 → BYE」**
- **SIPはサーバ経由、RTPは電話機同士で直接**
- **「呼び出せるのに声が聞こえない」はNAPT（SDPの中のアドレス）が原因** → SIP ALG、SBC
- **1通話は約80kbps（G.711）**

## 6. 勉強の優先順位

1. NAPTの環境で音声が届かない理由と対策を30〜40字で書けるようにする
2. SIP・SDP・RTPの役割分担
3. 発信の流れ（INVITE〜BYE）と、SIPとRTPの経路の違い
4. 音声の品質（遅延・ジッタ・損失）とQoS、帯域の計算
5. IP電話機の接続（PoE、音声VLAN）
6. SBC、STUN、クラウドPBXは名前と目的を知っている程度でよい

> [!tip] 得点につながる覚え方
> VoIPの問題では、**SIP（取り次ぎ）とRTP（声）を別の矢印**で構成図に書き込む。「呼び出せるのに声が届かない」と問われたら、RTPの矢印の途中にあるNAPT・FWを疑う。

## 7. 関連

- [[QoS]]（音声の優先制御、帯域の計算）
- [[NAT・NAPT]]（SDPの中のアドレス、ALG）
- [[FTP]]（同じくアドレスをデータの中で通知する）
- [[FW]]（SIP・RTPの許可）
- [[VLAN]]（音声VLAN）
- [[HTTP]]（応答コードの形が似ている）
