# sourceコマンドとは何か

## 定義

**現在のシェルプロセス内でスクリプトを読み込んで実行するコマンド**

```bash
source filename
# または
. filename  # 同じ意味（POSIXの標準形式）
```

---

## 直接実行との違い

### 直接実行（./script.sh）の場合

```bash
$ cat script.sh
#!/bin/bash
MY_VAR="hello"
cd /tmp

$ ./script.sh
$ echo $MY_VAR
          # 空（変数は設定されていない）
$ pwd
/home/user  # ディレクトリも変わっていない
```

**理由**: 新しいサブシェル（子プロセス）で実行される
- サブシェルで設定された変数や環境はサブシェル内だけで有効
- サブシェル終了後、親シェルには影響しない

### sourceで読み込んだ場合

```bash
$ source script.sh
$ echo $MY_VAR
hello  # 変数が設定されている！
$ pwd
/tmp   # ディレクトリも変わっている！
```

**理由**: 現在のシェル内で実行される
- 変数、関数、エイリアスがすべて現在のシェルに反映される
- カレントディレクトリも変更される

---

## プロセスの違い

### 直接実行（./script.sh）

```
┌─────────────┐
│ 親シェル    │
│  (bash)     │
└──────┬──────┘
       │ fork + exec
       ↓
┌─────────────┐
│ 子シェル    │  ← 新しいプロセス
│  (bash)     │     スクリプト実行
└─────────────┘     終了後、変数は消える
```

### source script.sh

```
┌─────────────┐
│ 現在のシェル│  ← 同じプロセス内
│  (bash)     │     スクリプトを読んで実行
│             │     変数・関数が残る
└─────────────┘
```

---

## 本来の用途

### 1. 設定ファイルの読み込み

```bash
source ~/.bashrc  # シェル設定の再読み込み
source ~/.zshrc
```

設定ファイルを編集した後、新しいシェルを起動せずに変更を反映できる。

### 2. 環境変数の設定

```bash
# env.sh
export API_KEY="secret"
export DB_HOST="localhost"
export DB_PORT="5432"

# 現在のシェルに環境変数を設定
$ source env.sh
$ echo $API_KEY
secret
```

スクリプト内で設定した環境変数が、現在のシェルで使えるようになる。

### 3. 関数やエイリアスの定義

```bash
# functions.sh
my_function() {
    echo "hello from function"
}

alias ll='ls -la'

# 現在のシェルに関数とエイリアスを追加
$ source functions.sh
$ my_function
hello from function
$ ll
# ls -la が実行される
```

---

## 比較表

| | 直接実行 (`./script.sh`) | source (`source script.sh`) |
|---|---|---|
| **プロセス** | 新しいサブシェルを起動 | 現在のシェル内で実行 |
| **変数の保持** | ❌ 保持されない | ✅ 保持される |
| **関数の保持** | ❌ 保持されない | ✅ 保持される |
| **エイリアス** | ❌ 保持されない | ✅ 保持される |
| **ディレクトリ変更** | ❌ 影響しない | ✅ 影響する |
| **実行権限** | ✅ 必要（`chmod +x`） | ❌ 不要 |
| **シェバン** | ✅ 必要（`#!/bin/bash`） | ❌ 不要 |
| **用途** | スクリプトの実行 | 設定ファイルの読み込み |
| **終了ステータス** | サブシェルの終了ステータス | 最後のコマンドの終了ステータス |

---

## 実用例

### 例1: プロジェクトごとの環境変数

```bash
# project-env.sh
export PROJECT_NAME="myapp"
export API_URL="https://api.example.com"
export DEBUG=true

# プロジェクトディレクトリに入ったら実行
$ cd ~/projects/myapp
$ source project-env.sh
$ echo $PROJECT_NAME
myapp
```

### 例2: よく使う関数をまとめる

```bash
# ~/.my-functions.sh
# Git関連の便利関数
gacp() {
    git add .
    git commit -m "$1"
    git push
}

mkcd() {
    mkdir -p "$1"
    cd "$1"
}

# .bashrc や .zshrc から読み込む
source ~/.my-functions.sh
```

### 例3: Python仮想環境の有効化

```bash
# Python venv
$ source venv/bin/activate
(venv) $  # プロンプトが変わる

# Node.js nvm
$ source ~/.nvm/nvm.sh
```

---

## Claude Codeでの利用

Claude Codeは、各コマンド実行時にスナップショットファイルを`source`で読み込む：

```bash
# 各コマンド実行時
source ~/.claude/shell-snapshots/snapshot-zsh-*.sh

# 効果:
# - .zshrcの関数が現在のシェルに追加される
# - エイリアスが現在のシェルに設定される
# - 環境変数が現在のシェルに設定される
```

これにより、非インタラクティブシェルでも`.zshrc`の設定が利用できる。

---

## sourceの本質

**「別のファイルの内容を、今のシェルで実行しているかのように扱う」**

あたかも、ファイルの内容をコピー＆ペーストしてシェルに入力したかのような動作をする。

```bash
# これと
$ source script.sh

# これは同じ
$ cat script.sh
export FOO="bar"
my_func() { echo "hello"; }

# ↓ コピー＆ペーストして実行
$ export FOO="bar"
$ my_func() { echo "hello"; }
```

---

## 注意点

### 1. 無限ループに注意

```bash
# script.sh
source script.sh  # 自分自身をsource → 無限ループ
```

### 2. 相対パスに注意

```bash
# 現在のディレクトリではなく、スクリプトがあるディレクトリ基準
source ./config.sh  # カレントディレクトリのconfig.sh

# 確実にするには絶対パスを使う
source ~/.config/myapp/config.sh
```

### 3. エラーハンドリング

```bash
# sourceしたスクリプトでエラーが起きると、現在のシェルに影響する
$ source buggy-script.sh
# 最悪の場合、シェルが終了することも

# 安全のため
$ source script.sh || echo "Failed to load script"
```

---

## まとめ

- `source`は現在のシェル内でファイルを実行する
- 設定ファイルの読み込みが主な用途
- 変数・関数・エイリアスが現在のシェルに残る
- 実行権限やシェバンは不要
- `.`コマンドと同じ意味（POSIXの標準）

**覚え方**: 「source = 今のシェルで実行」「直接実行 = 新しいシェルで実行」
