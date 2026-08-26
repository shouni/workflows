# ⚙️ Workflows

[![Status](https://img.shields.io/badge/Status-Active-brightgreen)](#)
[![GitHub tag (latest by date)](https://img.shields.io/github/v/tag/shouni/workflows)](https://github.com/shouni/workflows/tags)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 🚀 概要 (Overview)

Go リポジトリが共有する **再利用可能ワークフロー**（`workflow_call`）を置くリポジトリ。Go のコードは持たない。

25 本の `ci.yml` を突き合わせたところ、実質の差は **5 軸**しかなく、残りはジョブ名と
フラグの並び順だけだった。12 本が完全一致、別の 4 本は装飾だけの違い。
その 5 軸を入力として表現したものが `.github/workflows/go-ci.yml` にある。

| 軸 | 入力 |
|---|---|
| apt パッケージが要る | `apt-packages` |
| カバレッジを測る | `coverage` / `upload-coverage` |
| fuzz を短時間回す | `fuzz-targets` / `fuzz-time` |
| golangci-lint の版を変える | `golangci-lint-version` |
| ジョブの制限時間を変える | `timeout-minutes` |

**共有の主目的は重複の削減ではなく、対処の配布**にある。

下の「govulncheck を アクションで動かさない」がその実例で、4 本だけが持っていた回避策を 21 本が持っていなかった。

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
    uses: shouni/workflows/.github/workflows/go-ci.yml@v1.0.0
```

`on` / `concurrency` / `permissions` は呼ぶ側に残る。`workflow_call` を宣言できるのは、呼ばれる側だけなので、トリガーの定義は共有できない。

---

## ⚙️ 入力 (Inputs)

| 入力 | 型 | 既定値 | 意味 |
|---|---|---|---|
| `timeout-minutes` | number | `15` | 各ジョブの制限時間 |
| `golangci-lint-version` | string | `v2.13.1` | CI で使う golangci-lint の版 |
| `coverage` | boolean | `false` | カバレッジを測り、ステップサマリに出す |
| `upload-coverage` | boolean | `false` | `coverage.out` を artifact に上げる（7 日保持）。`coverage: true` が前提 |
| `apt-packages` | string | `''` | テスト前に入れる apt パッケージ（空白区切り） |
| `fuzz-targets` | string | `''` | `パッケージ#関数名` の空白区切り。空なら fuzz ジョブごと動かない |
| `fuzz-time` | string | `20s` | 1 ターゲットあたりの fuzz 時間 |

ジョブは `test` / `lint` / `vulncheck` / `fuzz` の 4 つ。`fuzz` は `fuzz-targets` が
空でない場合だけ走る。

---

## 🗂 リポジトリごとの設定 (Per-repository settings)

`with:` が要るのは少数で、残りは呼ぶだけでよい。**下表に挙げるのは公開リポジトリのみ。**

| リポジトリ | 設定 |
|---|---|
| `gcp-kit` | `fuzz-targets: "./auth#FuzzBuildLoginRedirectURL ./auth#FuzzIsSafeRelativePath ./auth#FuzzExtractBearerToken"` |
| `audio` | `fuzz-targets: "./wav#FuzzExtractAudioData ./wav#FuzzCombineToMatchesCombineWavData ./wav#FuzzCombineToDoesNotPanic"` |
| `netarmor` | `coverage: true` / `upload-coverage: true` / `fuzz-targets: "./securenet#FuzzValidateURL ./securenet#FuzzIsSecureServiceURL"` |
| `gemini-image-kit` | `coverage: true` / `fuzz-targets: "./internal/imgutil#FuzzDetectMIMEType ./internal/imgutil#FuzzCompressToJPEG"` |
| `go-gemini-client` | `coverage: true` / `fuzz-targets: "./gemini#FuzzCleanJSONResponse"` |

移行で挙動が変わるものが 2 つある。**`gcp-kit`** の fuzz は `test` ジョブの中に 埋まっていたが、独立ジョブになる。

**`audio`** は fuzz を 3 本持ちながら CI ジョブが無かったので、ここで初めて回り始める。

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
統合前にこの回避策を持っていたのは 4 本だけで、残り 21 本は Go がパッチを出すたびに
踏む位置にいた。

### `check-latest: true` を全ジョブに付ける

ランナーに焼かれている版をそのまま使わせない。標準ライブラリの脆弱性はパッチリリースで
直るため、キャッシュ優先だと govulncheck が古い toolchain を走査して落ち続ける
（`go.mod` の `go` 行はあくまで下限）。統合前は `go-review-kit` だけが持っていた。

### 外部入力は `env:` 経由で渡す

`apt-packages` と `fuzz-targets` は `${{ }}` を `run:` に直接埋めず、`env:` に置いてから
シェル変数として参照する。呼ぶ側は自分の管理下だが、テンプレート展開が
シェルの構文解析より先に起きる形を残さない。

### apt はリトライで包む

ミラー側の不調は「失敗」ではなく「応答しない」形で出る。実測で `apt-get update` が
14 分応答せず、ジョブ全体の制限時間をこのステップだけで使い切った（同一コミットの
`pull_request` 側は 4 分半で完走しており、原因はコードではなくミラー）。
**コマンド単位で `timeout` を掛けないと、ハングしたときにリトライへ進めない。**

### `.golangci.yml` は共有しない

**版のピンだけを共有し、設定そのものは各リポジトリに残す。**


25 本中 18 本が一致するが、 残る 7 本の差は意図的なものを含む。たとえば `go-notify` は `exhaustive` を追加して
`check: [switch, map]` にしてあり、理由が設定に書いてある——`notify.Level` を実際に
扱い分けているのは `slack` の `levelColors` でこれはマップなので、種別を足したときに
抜けても参照が空文字で素通りし、拾えないと無色の通知になる。

**共通化するとこの知識の置き場所が消える。** 一致している 18 本も、一致していることが
検査されていないだけで、揃える理由が別にあるわけではない。

### タグで固定する

呼ぶ側は `@v1.0.0` のように**不変のタグ**で固定する。Dependabot の `github-actions`
エコシステムが `uses:` の ref を追うので、新しい版を切ると**リポジトリごとに 1 本ずつ
PR が立つ**。共有ワークフローが壊れたときに 25 本が同時に落ちることを、これで避けられる。

**移動するタグ（`@v1` を毎リリース張り替える形）は使わない。** GitHub Actions ではよくある
流儀だが、ref の文字列が変わらないので Dependabot に上げるものが無く、全消費側が次の実行で
いきなり新しい内容を引く。**段階的な展開という目的と正面からぶつかる。**

新しい golangci-lint を 1 本だけで試したい場合は `golangci-lint-version` を上書きする。

---

## ⚠️ 限界 (Limitations)

**再利用可能ワークフローは任意のステップを差し込めない。** 

入力で表現できない要件が 出たリポジトリは、自前の `ci.yml` に戻ることになる。入力を増やして対応するか、
そのリポジトリだけ外れるかは、その都度の判断になる。

判断の目安は「その要件が他のリポジトリにも起こりうるか」。

`apt-packages` は 起こりうるので入力にしたが、1 本にしか起きない要件のために入力を増やすと、共有ワークフローが分岐の集合になって共有する意味が薄れる。

---

## 📄 ライセンス (License)

MIT License. 詳細は [LICENSE](LICENSE) を参照。
