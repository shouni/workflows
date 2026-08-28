# ⚙️ Workflows

[![Status](https://img.shields.io/badge/Status-Active-brightgreen)](#)
[![GitHub tag (latest by date)](https://img.shields.io/github/v/tag/shouni/workflows)](https://github.com/shouni/workflows/tags)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

フリート共通の CI を持つリポジトリ。Go のコードは持たない。

| 中身 | 役割 |
|---|---|
| `.github/workflows/go-ci.yml` | 再利用可能ワークフロー（`workflow_call`）。**24 リポジトリが呼んでいる** |
| README（この文書） | 各リポジトリの `ci.yml` と `dependabot.yml` が理由を書かずに参照する先 |

---

## 🚀 なぜ共有するのか (Why)

**目的は重複の削減ではなく、対処の配布。** ランナーや外部アクションの都合で必要になった
回避策（下の「設計の判断」）を 1 箇所に書けば、24 本すべてに届く。個別に持たせると、
持っているリポジトリと持っていないリポジトリが静かに分かれる。

リポジトリごとに差が要るのは次の 5 軸だけで、それがそのまま入力になっている。

| 軸 | 入力 |
|---|---|
| apt パッケージが要る | `apt-packages` |
| カバレッジを測る | `coverage` / `upload-coverage` |
| fuzz を回す | `fuzz-targets` / `fuzz-time` |
| golangci-lint の版を変える | `golangci-lint-version` |
| ジョブの制限時間を変える | `timeout-minutes` |

---

## 📦 使い方 (Usage)

呼ぶ側の `.github/workflows/ci.yml` はこれだけになる。

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  ci:
    uses: shouni/workflows/.github/workflows/go-ci.yml@v1
```

`on` / `concurrency` / `permissions` は呼ぶ側に残る。`workflow_call` を宣言できるのは
呼ばれる側だけなので、トリガーの定義は共有できない。

---

## ⚙️ 入力とジョブ (Inputs and jobs)

| 入力 | 型 | 既定値 | 意味 |
|---|---|---|---|
| `timeout-minutes` | number | `15` | 各ジョブの制限時間 |
| `golangci-lint-version` | string | `v2.13.1` | golangci-lint の版 |
| `coverage` | boolean | `false` | カバレッジを測り、ステップサマリに出す |
| `upload-coverage` | boolean | `false` | `coverage.out` を artifact に上げる（7 日保持）。`coverage: true` が前提 |
| `apt-packages` | string | `''` | テスト前に入れる apt パッケージ（空白区切り） |
| `fuzz-targets` | string | `''` | `パッケージ#関数名` の空白区切り |
| `fuzz-time` | string | `20s` | 1 ターゲットあたりの fuzz 時間 |

ジョブは 4 つ。`fuzz` は `fuzz-targets` が空だと条件で落ち、**実行されずに skipped と
表示される**（一覧から消えるわけではない）。

| ジョブ | 内容 |
|---|---|
| `Build & Test` | apt → build → vet → gofmt → test（+ カバレッジ） |
| `Lint` | golangci-lint。**設定は各リポジトリの `.golangci.yml` を読む** |
| `govulncheck` | 手動 install して実行 |
| `Fuzz (smoke)` | `fuzz-targets` を順に探索 |

**fuzz が落ちると、`fuzz-corpus` という artifact に再現用シードが上がる**
（`**/testdata/fuzz/`、7 日保持）。CI で見つけたクラッシュ入力はここにしか残らないので、
回帰テストにするならこれを取ってコミットする。成功時は生成されない。

**`fuzz-targets` が実態とズレると、ジョブは緑のまま何も探索しない。** `go test -fuzz` は
一致するターゲットが 0 個でも `no fuzz tests to fuzz` と警告して exit 0 を返すため、
出力を捕まえて落としている（下の「設計の判断」）。

---

## 🗂 リポジトリごとの設定 (Per-repository settings)

`with:` が要るのは 6 本で、残り 18 本は呼ぶだけでよい。**下表は公開リポジトリのみ。**

| リポジトリ | 設定 |
|---|---|
| `gcp-kit` | `fuzz-targets: "./auth/session#FuzzBuildLoginRedirectURL ./auth/session#FuzzIsSafeRelativePath ./auth/oidc#FuzzExtractBearerToken"` |
| `audio` | `fuzz-targets: "./wav#FuzzExtractAudioData ./wav#FuzzCombineToMatchesCombineWavData ./wav#FuzzCombineToDoesNotPanic"` |
| `netarmor` | `coverage: true` / `upload-coverage: true` / `fuzz-targets: "./securenet#FuzzValidateURL ./securenet#FuzzIsSecureServiceURL"` / `fuzz-time: 60s` |
| `gemini-image-kit` | `coverage: true` / `fuzz-targets: "./internal/imgutil#FuzzDetectMIMEType ./internal/imgutil#FuzzCompressToJPEG"` / `fuzz-time: 60s` |
| `go-gemini-client` | `coverage: true` / `fuzz-targets: "./gemini#FuzzCleanJSONResponse"` / `fuzz-time: 60s` |

`fuzz-time` は 3 本が `60s`、`gcp-kit` と `audio` は既定の `20s`。**この差に理由の記録は
無い**ので、値だけ据えて説明は付けていない。揃える判断をするなら、まず理由を決めること。

---

## 📐 設計の判断 (Design decisions)

### なぜ独立した公開リポジトリなのか

消費側には private なリポジトリが混じる。**公開リポジトリのワークフローは private からも
追加設定なしで呼べる**が、逆は成立しない。private に置くと、公開リポジトリから呼ぶ側に
個別のアクセス許可が要る。

### govulncheck をアクションで動かさない

`golang/govulncheck-action@v1` は `GOTOOLCHAIN=local` を強制するため、`go.mod` が要求する
版がランナーに無いと、走査以前にパッケージの読み込みで落ちる。`setup-go` は同じ `go.mod`
から解決でき、`check-latest` も効くので、他のジョブと同じ toolchain で走る。

**アクション側が `GOTOOLCHAIN` を上書きしなくなったら戻せる。**

### `check-latest: true` を全ジョブに付ける

ランナーに焼かれている版をそのまま使わせない。標準ライブラリの脆弱性はパッチリリースで
直るため、キャッシュ優先だと govulncheck が古い toolchain を走査して落ち続ける
（`go.mod` の `go` 行はあくまで下限）。

### 外部入力は `env:` 経由で渡す

`apt-packages` と `fuzz-targets` は `${{ }}` を `run:` に直接埋めず、`env:` に置いてから
シェル変数として参照する。呼ぶ側は自分の管理下だが、テンプレート展開がシェルの構文解析より
先に起きる形を残さない。

### fuzz は「探索しなかった」ことを失敗として扱う

`go test -run '^$' -fuzz '^Name$'` は、`Name` が存在しなくても **exit 0 で返る**
（`testing: warning: no fuzz tests to fuzz` を出して PASS）。終了コードだけを見ていると、
関数の改名・綴り違い・パッケージの移動が、そのまま「緑の fuzz ジョブ」になる。

**fuzz が唯一の防波堤になっている入力経路がある**ので（`netarmor` の URL 検証、`gcp-kit` の
Bearer トークン、`gemini-image-kit` の画像バイナリ判定）、黙って止まる形は残さない。出力に
その警告が出たらステップを落とす。

実例として `gcp-kit` は `./auth` から `./auth/session` と `./auth/oidc` へ分割されており、
`fuzz-targets` の追随が必要になったことが既にある。

### apt はリトライで包む

ミラー側の不調は「失敗」ではなく「応答しない」形で出る。実測で `apt-get update` が
14 分応答せず、ジョブ全体の制限時間をこのステップだけで使い切った（同一コミットの
`pull_request` 側は 4 分半で完走しており、原因はコードではなくミラー）。
**コマンド単位で `timeout` を掛けないと、ハングしたときにリトライへ進めない。**

**リトライの回数は、ステップの制限時間と噛み合っていないと嘘になる。** 1 回あたり
`timeout 60`（update）+ `timeout 120`（install）+ `sleep 10` = 190 秒で、3 回ぶんは 570 秒。
ステップは 10 分にしてある。以前はコマンド側が 120/180 秒でステップが 6 分だったため、
**2 回目の途中で打ち切られて 3 回は回っていなかった。**

### `.golangci.yml` は共有しない

**版のピンだけを共有し、設定そのものは各リポジトリに残す。** 24 本中 17 本が一致するが、
残る 7 本の差は意図的なものを含む。たとえば `go-notify` は `exhaustive` を追加して
`check: [switch, map]` にしてあり、理由が設定に書いてある——`notify.Level` を実際に
扱い分けているのは `slack` の `levelColors` でこれはマップなので、種別を足したときに
抜けても参照が空文字で素通りし、拾えないと無色の通知になる。

**共通化するとこの知識の置き場所が消える。** 一致している 17 本も、一致していることが
検査されていないだけで、揃える理由が別にあるわけではない。

新しい golangci-lint を 1 本だけで試したい場合は `golangci-lint-version` を上書きする。

---

## 🏷 タグと反映 (Tags and rollout)

### 移動するタグ `@v1` で参照する

呼ぶ側は `@v1` を指す。このタグはリリースのたびに張り替える。

**不変のタグ（`@v1.0.0`）で固定して Dependabot に上げさせる形は採れない。** 消費側を
`@v1.0.0` に固定して新しい版を切り、日次ジョブと手動スキャンで複数回走らせて確かめた
結果がこれ。

| | |
|---|---|
| 依存グラフが参照を検出するか | **する**（`go-ci.yml` を版付きで記録） |
| Dependabot が版を上げる PR を出すか | **出さない**（毎回 "No PRs affected"、ブランチも作られない） |

**依存グラフの検出と、version update の対応範囲は別の系統。** ジョブレベルの `uses:`
（`jobs.<id>.uses`）は、ステップレベルの `uses:`（`steps[].uses`）と違って更新対象に
なっていない。依存グラフに載っていることを「Dependabot が追う」根拠にしないこと。

`sed` で全消費側の ref を一括置換する案も検討したが、更新のたびに全リポジトリを同時に
書き換えることになり、**段階的展開を守っているようで守れていない**ので採らなかった。

### 変更を反映する手順

移動タグは張り替えた瞬間に全消費側へ届く。**壊れた内容を張り替えれば全部が同時に落ちる。**
ガードが仕組みではなく手順にあるので、3 を飛ばさないこと。

1. develop で `go-ci.yml` を変更し、main へ
2. **不変タグ `v1.x.y` を切る**（履歴として、また壊れたときの退避先として残す）
3. **消費側 1 本だけ**を一時的に `@v1.x.y` へ向け、CI が緑になるのを確認する
4. 緑なら `v1` をそのコミットへ移す
5. 残りは次回の CI 実行から新しい内容になる
6. カナリアにした 1 本を `@v1` へ戻す

**`v1` を動かすのは `go-ci.yml` が変わったときだけ。** 消費側から見えるのは `go-ci.yml`
だけなので、README や `dependabot.yml` の修正で張り替えると、タグの履歴が
「いつ中身が変わったか」を示さなくなる。

裏返すと、**このリポジトリへのマージだけでは消費側に何も届かない。** マージした時点で
反映される普通のリポジトリとは効き方が違う。遅延ではなく仕様。

---

## 🤖 Dependabot の規約 (Dependabot conventions)

フリートの各リポジトリは同じ形の `.github/dependabot.yml` を置いている。**各ファイルは
設定だけを持ち、理由はここにある。**

### 全依存を 1 グループにまとめる

`patterns: ["*"]` で 1 本の PR に収める。個別に PR を立てると、直接依存の多いリポジトリで
大量に積み上がる。それ以上に問題なのは、**相互に整合が必要な依存が別々の PR に割れる**こと
（`gcp-kit` の `cloud.google.com/go*` / `google.golang.org/*` / `golang.org/x/oauth2` が
その例で、以前は専用グループを切っていた）。グループ化していると、既存の PR は作り直されず
更新される。

### 日次で確認する

内部ライブラリ（`github.com/shouni/*`）にタグを打った翌日には、依存側へ PR が届くように
するため。バージョンが枝分かれしたまま放置されるのを防ぐ。

### 定常更新は develop 宛。ただし脆弱性 PR は main へ行く

`target-branch: develop` にして、依存更新も他の変更と同じ経路（develop → main）を通す。
main を検証済みの状態に保つため。

**この指定が効くのは定常更新だけ。** 脆弱性の修正 PR は `target-branch` に関わらず、常に
デフォルトブランチ（main）宛に作られる。緊急対応を develop の進行状況から切り離すための
GitHub 側の仕様で、設定では変えられない。**main と develop が枝分かれしていると、
脆弱性 PR をマージした分が develop に降りてこない**ので、定期的に揃えること。

### 内部ライブラリの更新は手作業が先行する

実態として `github.com/shouni/*` のバージョンは手で一斉に上げていることが多く
（`deps: bump shouni libraries` の類）、Dependabot の PR は「もう上がっている」ことの
確認になりがち。**それでも設定を外さないのは、一斉更新の取りこぼしを拾うため。**
手作業は数十本を相手にすると必ず漏れる。

### エコシステムはリポジトリの中身で決まる

| リポジトリの種類 | 宣言するもの |
|---|---|
| Go のライブラリ / アプリ | `gomod` + `github-actions` |
| Terraform | `terraform` + `github-actions` |
| **このリポジトリ**（Go のコード無し） | `github-actions` のみ |

Go のコードが無いのに `gomod` を宣言すると、依存ゼロの走査が毎日回るだけになる。
**このリポジトリの `dependabot.yml` は標準形ではなくこの例外**なので、見本にするなら
Go のリポジトリのものを見ること。

---

## ⚠️ 限界 (Limitations)

**再利用可能ワークフローは任意のステップを差し込めない。** 入力で表現できない要件が出た
リポジトリは、自前の `ci.yml` に戻ることになる。入力を増やして対応するか、そのリポジトリ
だけ外れるかは、その都度の判断になる。

判断の目安は「**その要件が他のリポジトリにも起こりうるか**」。`apt-packages` は起こりうる
ので入力にしたが、1 本にしか起きない要件のために入力を増やすと、共有ワークフローが分岐の
集合になって共有する意味が薄れる。

---

## 📄 ライセンス (License)

MIT License. 詳細は [LICENSE](LICENSE) を参照。
