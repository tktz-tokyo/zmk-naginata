# zmk-naginata 開発環境向けツール導入手順（提案）

このドキュメントは、`west` / `dtc` / `arm-none-eabi-gcc` が未導入の環境向けに、**実行順で整理した提案手順**です。

> 前提: Debian / Ubuntu 系 Linux を想定

---

## 1. 事前確認（提案）

以下の確認を行い、既存導入状況を把握します。

- `west --version`
- `dtc --version`
- `arm-none-eabi-gcc --version`

いずれかが `command not found` の場合、以降の導入手順へ進みます。

---

## 2. パッケージインデックス更新（提案）

```bash
sudo apt update
```

---

## 3. `dtc` の導入（提案）

`dtc` は Debian/Ubuntu では `device-tree-compiler` パッケージに含まれます。

```bash
sudo apt install -y device-tree-compiler
```

導入後確認:

```bash
dtc --version
```

---

## 4. `arm-none-eabi-gcc` の導入（提案）

GNU Arm Embedded Toolchain は以下のパッケージで導入します。

```bash
sudo apt install -y gcc-arm-none-eabi binutils-arm-none-eabi
```

導入後確認:

```bash
arm-none-eabi-gcc --version
```

> 補足: 環境によっては追加で `libnewlib-arm-none-eabi` / `libstdc++-arm-none-eabi-newlib` が必要になる場合があります。

---

## 5. `west` の導入（提案）

`west` は Python パッケージとして導入するのが一般的です。`pipx` 利用を推奨します。

### 5-1. pipx を使う場合（推奨）

```bash
sudo apt install -y pipx
pipx ensurepath
pipx install west
```

シェル再起動後、以下で確認:

```bash
west --version
```

### 5-2. pip を使う場合（代替）

```bash
python3 -m pip install --user --upgrade west
```

その後、`~/.local/bin` が `PATH` に含まれていることを確認します。

---

## 6. 依存関係をまとめて導入する場合（提案）

一括で進める場合の例:

```bash
sudo apt update
sudo apt install -y device-tree-compiler gcc-arm-none-eabi binutils-arm-none-eabi pipx
pipx ensurepath
pipx install west
```

---

## 7. 最終確認（提案）

以下 3 コマンドがすべて成功することを確認します。

```bash
west --version
dtc --version
arm-none-eabi-gcc --version
```

---

## 8. トラブル時の確認ポイント（提案）

- `west` のみ見つからない: `PATH` に `~/.local/bin`（または pipx の bin ディレクトリ）が入っているか確認。
- `arm-none-eabi-gcc` が見つからない: パッケージ導入後に新しいシェルを開き直す。
- バージョン競合がある: システム導入版とユーザー導入版（pip/pipx）が混在していないか確認。

