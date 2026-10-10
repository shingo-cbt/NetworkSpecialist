---
tags: [ネスペ, ICMP, ネットワーク層]
---

# ICMP まとめ（ネスペ出題観点）

出題度：◎ 午後で記述・穴埋めとして頻出／○ 午前・午後小問で出る／△ 深追い不要

## 1. ICMPの基本（○）

- IP通信の**エラー通知**と**診断**のためのプロトコル。ネットワーク層で動く
- IPヘッダのプロトコル番号は**1**（ICMPv6は**58**）
- ヘッダ先頭は **Type（種類）／Code（詳細）／Checksum**
- エラーメッセージには**元のIPヘッダとデータ先頭8バイト**を入れて返す → 送信元はどの通信（ポート番号）のエラーか特定できる
- 午前の誤り選択肢の定番：「ICMPはTCP／UDPの上で動く」→ 誤り

## 2. ICMPv4の主なメッセージ

| Type | 名前 | 用途・ポイント | 出題度 |
|---|---|---|---|
| 0 | Echo Reply | pingの応答 | ○ |
| 3 | Destination Unreachable | 宛先に届かない。詳細はCodeで表す | ◎（Code 4） |
| 5 | Redirect | 同一セグメント内の、より適切なルータをホストに通知 | ○ |
| 8 | Echo Request | pingの要求 | ○ |
| 11 | Time Exceeded | TTLが0になって破棄。tracerouteで利用 | ◎ |
| 12 | Parameter Problem | IPヘッダの値が不正 | △ |
| 4 | Source Quench | 送信抑制。廃止済み | △ |

### Type 3（到達不能）の主なCode

| Code | 意味 | ポイント | 出題度 |
|---|---|---|---|
| 0 | Network Unreachable | 宛先ネットワークへの経路がない | △ |
| 1 | Host Unreachable | 宛先ホストに届かない | △ |
| 2 | Protocol Unreachable | 宛先がそのプロトコルに未対応 | △ |
| 3 | Port Unreachable | 宛先ポートが待ち受けていない（UDP） | △（traceroute文脈で○） |
| 4 | Fragmentation Needed and DF set | DFのため分割不可。**PMTUD**で利用 | ◎ |
| 13 | Administratively Prohibited | フィルタで拒否された | △ |

### Type 11（時間超過）のCode

- Code 0：転送中にTTLが0になった（◎ tracerouteの根拠）
- Code 1：フラグメントの再組み立てタイムアウト（△）

## 3. ICMPv6の主なメッセージ

| Type | 名前 | ポイント | 出題度 |
|---|---|---|---|
| 1 | Destination Unreachable | v4のType 3に相当 | △ |
| 2 | Packet Too Big | IPv6はルータで分割しないため、PMTUDに必須 | ○ |
| 3 | Time Exceeded | v4のType 11に相当（Hop Limit） | △ |
| 4 | Parameter Problem | v4のType 12に相当 | △ |
| 128 | Echo Request | ping | △ |
| 129 | Echo Reply | ping応答 | △ |
| 133 | RS（Router Solicitation） | ホスト→ルータ「ルータいる？」 | ◎ |
| 134 | RA（Router Advertisement） | ルータ→ホスト。プレフィックス通知（SLAAC） | ◎ |
| 135 | NS（Neighbor Solicitation） | ARP要求の代わり、DAD（重複アドレス検出） | ◎ |
| 136 | NA（Neighbor Advertisement） | ARP応答の代わり | ◎ |
| 137 | Redirect | v4のType 5に相当 | △ |

- 133〜137はNDP（近隣探索プロトコル）。ARPの役割をICMPv6が担う
- NDPは**Type番号より名前と役割の対応**が問われる

### IPv6ではARPとRARPが不要（○）

- **IPv6にはARPもRARPもない**。それぞれの役割をICMPv6（NDP）などが引き継いだ

| IPv4 | 役割 | IPv6での代わり |
|---|---|---|
| **ARP** | 相手のIPアドレスからMACアドレスを知る | **NDPのNS／NA**（ICMPv6） |
| **RARP** | 自分のMACアドレスから自分のIPアドレスを知る | **SLAAC**（RAで受けたプレフィックス＋自分で作るインタフェースID）、**DHCPv6** |
| Gratuitous ARP | 重複検出、MACアドレス変更の通知 | **DAD**（NS）、**非要請NA** |

- ARPはブロードキャストを使うが、NSは**マルチキャスト（要請ノードマルチキャストアドレス）**宛てに送る → 関係ない端末の負荷が減る
- RARPはIPv4でもすでにBOOTP／DHCPに置き換わっていた
- 午前の誤り選択肢の定番：「IPv6でもARPでMACアドレスを解決する」→ 誤り
- → [[Gratuitous ARP]]

## 4. 午後で問われるテーマと記述の型

### ◎ PMTUDとICMP遮断（PMTUDブラックホール）

- 流れ
  1. DFビット付きのパケットがMTUの小さい区間に入る
  2. ルータが **Type 3 / Code 4** を返す（次ホップのMTU値も通知）
  3. 送信元はサイズを小さくして再送する
- FWでこのICMPを遮断 → 大きいパケットだけ届かない
- 問われ方：「一部の通信（大きいデータ）だけ失敗する原因」「FWで許可すべき通信」
- 対策
  - FWで該当ICMPを許可する
  - ルータで **TCP MSSを小さく書き換える（MSSクランプ）**。PPPoE、IPsec、GREなどMTUが減る構成とセットで出る
- 記述の型：「ICMPのフラグメント要求が破棄され、送信元が経路MTUを検出できないから」

### ◎ tracerouteの仕組み

- TTLを1、2、3…と1ずつ増やして送る
- 途中ルータは **Time Exceeded（Type 11）** を返す → その送信元から経路上のルータを特定
- 最終到達の判定
  - Windowsの`tracert`：ICMP Echo → 宛先から **Echo Reply（0）**
  - Linux等の`traceroute`：UDPの大きいポート番号 → 宛先から **Port Unreachable（Type 3 / Code 3）**
- 途中機器がICMPを返さない・FWで遮断 → 「*」表示になる
- 記述の型：「TTLを1ずつ増やして送信し、TTL超過を通知するICMPの送信元から経路上のルータを特定する」

### ◎ IPv6のNDP

- RS／RA：ルータ探索、プレフィックス通知（SLAAC）
- NS／NA：アドレス解決（ARP代替）、DAD
- 不正なRAによるなりすまし → 対策は **RAガード**
- **ICMPv6を丸ごと遮断すると通信自体が成り立たない**
- IPv6移行・デュアルスタックの問題で出る

### ○ ICMPリダイレクト

- 同一セグメントに複数ルータがある構成で、デフォルトゲートウェイがより適切なルータをホストに通知
- 問われ方：「ホストのルーティングが途中で変わった理由」

### ○ pingによる疎通確認

- Echo Request（8）→ Echo Reply（0）
- 問われ方：FWフィルタへのICMP追加、監視でICMPを返さない機器を検知できない、冗長構成（VRRPなど）の切替確認

### △ ICMPエラーを返さないケース（知識として持つ程度）

- ICMPエラーメッセージに対しては返さない（エラーの連鎖防止）
- 2番目以降のフラグメントに対しては返さない
- ブロードキャスト／マルチキャスト宛てに対しては返さない

## 5. 覚え方

- **pingは「8で投げて0で返る」**
- **到達不能は3、時間切れは11**：tracerouteは「11を集めて、最後に3（または0）」
- **Type 3 / Code 4 はPMTUD**：「さん・よん（3-4）で、サイズ（MTU）を詰める」
- **v6のNDPは133から「R→R→N→N→リダイレクト」**：ルータを探す（RS）→ルータが答える（RA）→隣を探す（NS）→隣が答える（NA）→道案内（Redirect）。Solicitation（問い合わせ）とAdvertisement（回答）のペア
- **v6のエラーは1〜4に詰め直された**：v4のType 3・11・12との対応で覚える

## 6. 勉強の優先順位

1. PMTUD＋MSSクランプの流れを、自分の言葉で30〜40字で説明できるようにする
2. tracerouteの仕組みを同様に説明できるようにする
3. NDPの4つ（RS／RA／NS／NA）の役割とRAガードを押さえる
4. 残りは午前の過去問で出たら確認する程度でよい

> [!tip] 得点につながる覚え方
> ICMPは単独で大問になるより、**FW設計、VPN、IPv6移行の問題の中で「なぜ通信できないか」の理由として**出る。番号の暗記より、「どのICMPを止めると何が壊れるか」で覚える。
