---
tags: [ネスペ, LDAP, ディレクトリサービス, 認証, アプリケーション層]
---

# LDAP まとめ（ネスペ出題観点）

出題度：◎ 午後で記述・穴埋めとして頻出／○ 午前・午後小問で出る／△ 深追い不要

## 1. LDAPの基本（○）

- **ディレクトリサービスにアクセスするためのプロトコル**（Lightweight Directory Access Protocol）
- **ディレクトリサービス**：利用者、グループ、機器などの情報を**木構造で一元管理するデータベース**
  - 「社員の名前、ID、パスワード、所属部署、メールアドレス」などを1か所で管理する
  - 読み出し（検索、認証）が多く、書き込みが少ない用途に向く
- 代表的な製品：**Active Directory（AD）**、OpenLDAP
- ポート

| 方式 | ポート | 内容 |
|---|---|---|
| LDAP | **TCP（Transmission Control Protocol） 389** | 平文 |
| **LDAPS**（LDAP over SSL/TLS） | **TCP 636** | 最初からTLS（Transport Layer Security）で暗号化 |
| StartTLS | TCP 389 | 平文で接続した後、TLSに切り替える |

- 午前の定番：「LDAPはディレクトリにアクセスするプロトコル」「認証情報を一元管理する」

## 2. データの構造（○）

- **DIT**（Directory Information Tree、ディレクトリ情報ツリー）：情報を木構造で管理する
- 1件のデータ＝**エントリ**。エントリは**属性**（名前と値の組）を持つ
- 各エントリは**DN（Distinguished Name、識別名）**で一意に特定する。DNは、木の葉から根に向かって**RDN（Relative Distinguished Name、相対識別名）**を並べたもの

```
dc=jp
 └ dc=example
    └ ou=Sales
       └ cn=Taro Yamada      ← このエントリのDN：
                                cn=Taro Yamada,ou=Sales,dc=example,dc=jp
```

| 属性 | 意味 |
|---|---|
| dc（domainComponent） | ドメイン名の構成要素 |
| ou（organizationalUnit） | 部署などの組織単位 |
| cn（commonName） | 名前 |
| uid | ユーザID |
| objectClass | エントリの種類（どの属性を持てるか） |

- 穴埋めで「DN」「cn」「ou」「dc」の意味が問われることがある（△〜○）

## 3. 主な操作（○）

| 操作 | 内容 |
|---|---|
| **Bind** | LDAPサーバに対して**認証する**（ログイン） |
| **Search** | 検索の起点（ベースDN）、範囲、条件（例：`(uid=taro)`）を指定してエントリを探す |
| Compare | 属性の値が一致するか確認する |
| Add／Delete／Modify | エントリの追加・削除・変更 |
| Unbind | 切断する |

- Bindの方式
  - 匿名：認証なし
  - **シンプル認証**：DNとパスワードを送る → **LDAPのままだとパスワードが平文で流れる**（◎）
  - SASL（Simple Authentication and Security Layer）：Kerberosなど別の認証の仕組みを使う

## 4. 午後で問われるテーマと記述の型

### ◎ 認証基盤の統合（ID管理の一元化）

- 無線LAN（Local Area Network）、VPN（Virtual Private Network）、プロキシ、各サーバに**別々にIDとパスワードを登録すると管理が大変**（退職者のアカウント削除漏れなど）
- **LDAPサーバ（AD）に利用者情報をまとめ**、各システムはLDAPに問い合わせて認証する
- 典型的な構成（802.1X認証）

```
[端末] ──EAP── [AP／L2SW] ──RADIUS── [RADIUSサーバ] ──LDAP── [LDAPサーバ（AD）]
                 認証装置                  認証の判断           利用者情報の置き場
```

- 記述の型（利点）：「利用者情報を一元管理でき、追加・削除をLDAPサーバで行うだけで各システムに反映されるから」
- → [[IEEE802.1X]]

### ◎ RADIUSとLDAPの違い

| | RADIUS（Remote Authentication Dial In User Service） | LDAP |
|---|---|---|
| 役割 | **認証の要求と結果をやり取りする**（AAA（Authentication, Authorization, Accounting）：認証・認可・アカウンティング） | **利用者情報（ディレクトリ）を読み書きする** |
| やり取りの相手 | AP（Access Point）、スイッチ、VPN装置などの認証装置 ↔ RADIUSサーバ | RADIUSサーバ、各サーバ ↔ LDAPサーバ |
| ポート | UDP（User Datagram Protocol） 1812（認証）／1813（アカウンティング） | TCP 389／636 |

- 「RADIUSが窓口、LDAPが名簿」と分けて覚える
- 問われ方：「RADIUSサーバがLDAPサーバに問い合わせる目的」

### ◎ シンプル認証の平文パスワード

- シンプル認証をLDAP（389番）のまま使うと、**パスワードが平文でネットワークに流れる**
- 対策：**LDAPS（636番）** または **StartTLS** で暗号化する
- 記述の型：「LDAPのシンプル認証ではパスワードが平文で送信されるため、TLSで通信を暗号化する」

### ○ 属性を使った認可（ダイナミックVLAN）

- LDAPの**所属グループ・部署の属性**をもとに、RADIUSサーバが接続先のVLAN（Virtual LAN）を指定する
- 同じSSID（Service Set Identifier）・同じスイッチでも、利用者ごとに別のVLANに振り分けられる
- 問われ方：「部署ごとにアクセスできるネットワークを分ける方法」
- → [[VLAN]]

### ○ 冗長化

- LDAPサーバが止まると、**それを使う全システムで認証できなくなる**（単一障害点）
- 対策：複数台のLDAPサーバで**レプリケーション（複製）**し、RADIUSサーバなどから複数台を参照する
- 問われ方：「LDAPサーバを冗長化する理由」

### ○ FWのルール

- RADIUSサーバや各サーバ → LDAPサーバ：**TCP 389**（LDAPSなら636）
- DMZ（DeMilitarized Zone）のサーバから内部のLDAPサーバに問い合わせる構成では、この通信を許可する必要がある

### △ Active Directory

- MicrosoftのディレクトリサービスでLDAPに対応している。ほかに**Kerberos**（認証）と**DNS**（Domain Name System、サーバの場所をSRV（Service）レコードで探す）を組み合わせて動く
- ADに参加したPCは、Kerberosにより**一度のログオンで社内の各サーバを利用できる（シングルサインオン）**
- LDAP自体はシングルサインオンの仕組みではない → Webサービス間のSSO（Single Sign-On）はSAML（Security Assertion Markup Language）など別の仕組み（△）

## 5. 覚え方

- **LDAPは「社内の電話帳（名簿）」**：名前から、所属、ID、パスワードを引く
- **「RADIUSが窓口、LDAPが名簿」**：認証装置の相手をするのがRADIUS、利用者情報を持つのがLDAP
- **DNは「住所を番地から県まで書く」**：cn（名前）→ ou（部署）→ dc（ドメイン）の順
- **ポートは「389（さんぱっく）、暗号化すると636」**
- **シンプル認証は「名前の通りシンプル＝平文」** → LDAPSで包む

## 6. 勉強の優先順位

1. 802.1X構成での役割分担（認証装置、RADIUS、LDAP）を図で説明できるようにする
2. RADIUSとLDAPの違いを一言で書けるようにする
3. シンプル認証が平文である問題とLDAPS／StartTLSの対策を30〜40字で書けるようにする
4. 認証を一元化する利点、LDAPサーバを冗長化する理由
5. ポート番号（389、636）
6. DNの構造、操作の名前、ADの仕組みは名前を知っている程度でよい

> [!tip] 得点につながる覚え方
> LDAPは単独で深く問われるより、**無線LANやVPNの認証構成（802.1X＋RADIUS）の一部**として出る。構成図にRADIUSサーバとLDAPサーバ（AD）があったら、「誰が誰に何を問い合わせるか」を矢印で書き込んで整理する。

## 7. 関連

- [[IEEE802.1X]]（RADIUS、EAP（Extensible Authentication Protocol））
- [[VLAN]]（ダイナミックVLAN）
- [[TLS]]（LDAPS、StartTLS）
- [[DNS]]（ADのSRVレコード）
