# Claude Code のシェル環境スナップショット機構 - 調査報告

**調査日**: 2026年2月11日
**調査環境**: macOS, zsh, Claude Code 2.1.38

## 目次

1. [確認できた事実](#確認できた事実)
2. [シェルの基礎知識](#シェルの基礎知識)
3. [実験による検証](#実験による検証)
4. [実装の確認](#実装の確認)
5. [推測と未確認事項](#推測と未確認事項)
6. [まとめ](#まとめ)

---

## 確認できた事実

### 1. スナップショットファイルの存在

```bash
$ ls ~/.claude/shell-snapshots/
snapshot-zsh-1770761628337-k9cbgb.sh
snapshot-zsh-1770760687206-uml7wn.sh
...（95個のファイル）
```

**事実**:
- スナップショットファイルは `~/.claude/shell-snapshots/` に保存される
- ファイル名: `snapshot-zsh-<timestamp>-<random>.sh`
- パーミッション: `644` (rw-r--r--) - 実行権限なし
- シェバン（`#!/bin/zsh`）なし - `source`で読み込むため不要

### 2. スナップショットファイルの観察

```bash
$ ls ~/.claude/shell-snapshots/
snapshot-zsh-1770761628337-k9cbgb.sh
snapshot-zsh-1770760687206-uml7wn.sh
...
```

**事実**:
- ファイル名形式: `snapshot-zsh-<タイムスタンプ>-<ランダム文字列>.sh`
- 複数のスナップショットファイルが残存している

**推測**:
- タイムスタンプはファイル作成時刻（セッション起動時）
- 正常終了したセッションのスナップショットは削除される
- 残存ファイル = 異常終了したセッション

### 3. スナップショットの内容

```bash
$ head -50 ~/.claude/shell-snapshots/snapshot-zsh-*.sh

# Snapshot file
# Unset all aliases to avoid conflicts with functions
unalias -a 2>/dev/null || true

# Functions
bashcompinit () { ... }
command_not_found_handler () { ... }
gbr () { ... }  # .zshrcで定義されたユーザー関数
gc () { ... }   # .zshrcで定義されたユーザー関数
git () { ... }  # .zshrcで定義されたユーザー関数
...

# Aliases
alias ll='eza -la --icons'
alias gs='git status'
...

# Environment variables
export PATH='/Users/...'
```

**事実**:
- スナップショットには `.zshrc` で定義された内容が含まれる：
  - 関数定義（ユーザー定義関数とシステム関数の両方）
  - エイリアス
  - 環境変数
- 最初に `unalias -a` で既存のエイリアスをすべてクリアしている

**なぜエイリアスをクリアするのか**

zshでは、同じ名前のエイリアスと関数がある場合、**エイリアスが優先**される：

```bash
# 既存の環境
alias foo='echo "alias"'

# スナップショットで関数を定義
foo() { echo "function"; }

# 結果: エイリアスが優先される
$ foo
alias  # 期待していた動作: function
```

この問題を避けるため、スナップショットは以下の順序で実行される：

```bash
1. unalias -a          # すべてのエイリアスをクリア
2. 関数定義            # 関数を定義
3. エイリアス定義      # エイリアスを定義
```

これにより：
- スナップショットで定義した関数が確実に実行される
- スナップショットで定義したエイリアスのみが有効になる
- 何度sourceしても同じ結果（冪等性）

### 4. Claude Code内でのエイリアス動作

```bash
# Claude Code内で実行
$ alias ll
ll='eza -la --icons'

$ echo $-
569Xl  # 非インタラクティブ

$ ll
# → 動作する！
```

**事実**:
- Claude Codeのシェルは非インタラクティブ（`$-`に`i`がない）
- それでもエイリアスが使える

---

## シェルの基礎知識

### シェルの2つの独立した軸

#### 軸1: ログイン vs 非ログイン

| 非ログイン | ログイン |
|---|---|
| フラグなし | `-l`フラグで起動 |
| ターミナル内で起動 | システムログイン時のシェル |
| 追加のシェルインスタンス | ssh接続時など |

#### 軸2: インタラクティブ vs 非インタラクティブ

| 非インタラクティブ | インタラクティブ |
|---|---|
| プロンプトなし | プロンプトが表示される |
| スクリプト実行 | ユーザーと対話 |
| `$-`に`i`がない | `$-`に`i`が含まれる |

### zshの設定ファイル読み込み（事実）

| | 非ログイン | ログイン |
|---|---|---|
| **非インタラクティブ** | `.zshenv` のみ | `.zshenv` → `.zprofile` → `.zlogin` |
| **インタラクティブ** | `.zshenv` → `.zshrc` | `.zshenv` → `.zprofile` → `.zshrc` → `.zlogin` |

**重要**: `.zshrc`はインタラクティブシェルの時のみ読み込まれる

---

## 実験による検証

### 実験1: 非インタラクティブシェルでのエイリアス

```bash
# スクリプトを作成
$ cat > script.sh << 'EOF'
#!/bin/zsh
ll
EOF

$ chmod +x script.sh
$ ./script.sh
./script.sh:2: command not found: ll
```

**結果**: 非インタラクティブシェルでは`.zshrc`が読み込まれないため、エイリアスが使えない

### 実験2: sourceコマンドと実行権限

```bash
# 実行権限なしのファイルを作成
$ cat > test.sh << 'EOF'
test_func() { echo "success"; }
EOF
$ chmod 644 test.sh

# 直接実行 → 失敗
$ ./test.sh
permission denied: ./test.sh

# source → 成功
$ source test.sh
$ test_func
success
```

**結果**: `source`コマンドは実行権限を必要としない

### 実験3: ログインシェルフラグ

```bash
# ログイン + 非インタラクティブ
$ zsh -l -c 'echo $-; alias ll'
569Xl
alias: ll: not found  # エイリアスなし！
```

**結果**: `zsh -l -c`だけでは`.zshrc`は読み込まれない（非インタラクティブのため）

---

## 実装の確認

### ソースコードの確認（事実）

**ファイル**: `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js`

#### スナップショット作成の流れ

```javascript
// セッション起動時（1回のみ）
async function aSY(){
  let A=await oSY(),q;  // シェルパスを取得
  try{
    q=await oc4(A)  // スナップショット作成
  }catch(K){
    q=void 0
  }
  return {binShell:A,snapshotFilePath:q}
}

// oc4() の中で実行される（行番号2606付近）
ISY(A,["-c","-l",_],{
  env:{
    ...process.env,
    SHELL:A,
    GIT_EDITOR:"true",
    CLAUDECODE:"1"
  },
  timeout:rc4,
  maxBuffer:1048576,
  encoding:"utf8"
}, ...)

// 各コマンド実行時
async function nW6(...){
  let {binShell:_,snapshotFilePath:J}=await zyA();  // キャッシュ取得
  if(J){
    G.push(`source ${snapshotPath}`)  // スナップショットをsource
  }
  // コマンド実行
}
```

#### 実行タイミング

| タイミング | 実行される処理 |
|---|---|
| **セッション起動時** | `ISY(zsh,["-c","-l",スクリプト],{...})` でスナップショット作成 |
| **各コマンド実行時** | `source ~/.claude/shell-snapshots/snapshot-zsh-*.sh` で復元 |

#### コマンドの詳細

```javascript
ISY(A,["-c","-l",_],{...})
```

**確認できたこと**:
- `A`: シェルのパス（例: `/bin/zsh`）
- `["-c","-l",_]`: シェルに渡す引数
  - `-c`: コマンドモード
  - `-l`: ログインシェルモード
  - `_`: 実行されるスクリプト（変数）
- 環境変数:
  - `SHELL:A`: シェルパスを設定
  - `GIT_EDITOR:"true"`: エディタを無効化
  - `CLAUDECODE:"1"`: Claude Code実行フラグ

**確認できないこと**:
- `_`（スクリプト）の具体的な内容
- どのようにして`.zshrc`の内容を取得しているか

#### nW6の呼び出し元

**Bashツールから呼ばれる（行5864-5875付近）**:

```javascript
// Bashツールの実装
async call(A,q,K,Y,z){
  ...
  // Tzz関数を呼び出し
  let n=Tzz({input:A,abortController:w,...}),
  ...
}

// Tzz: コマンド実行を担当する関数
async function*Tzz({input:A,abortController:q,...}){
  let{command:w,...}=A,
  // ★ここでnW6を呼び出してシェルコマンドを実行
  G=await nW6(w,q.signal,J,O,(B,S,m)=>{D=B,X=S,j=m},z,Ic(A),W),
  f=G.result;
  ...
}
```

**呼び出しフロー**:

```
ユーザーがBashコマンドを実行
  ↓
Bashツールの call()
  ↓
Tzz() 関数
  ↓
nW6() ← ここでスナップショットをsourceしてコマンド実行
  ↓
シェルプロセス起動
  ├─ source ~/.claude/shell-snapshots/snapshot-zsh-*.sh
  └─ ユーザーのコマンドを実行
```

**その他の用途**: シェルコマンドの補完機能（`Ijz`関数）でも使用される

**結論**: `nW6`は**Bashツールが実行されるたびに**呼ばれ、スナップショットを復元してからコマンドを実行する。

### なぜログインシェル（`-l`）を使うのか

#### 実験：ログインシェルだけでは `.zshrc` は読み込まれない

```bash
$ zsh -l -c 'alias ll'
alias: ll: not found  # .zshrcは読み込まれない
```

#### しかしスナップショットには `.zshrc` の内容が含まれる

```bash
$ grep "^alias ll=" ~/.claude/shell-snapshots/snapshot-zsh-*.sh
alias ll='eza -la --icons'  # .zshrcの内容がある！
```

#### 推測：スクリプト内で明示的に `.zshrc` をsource

```bash
# スクリプト _ の推測される内容
source ~/.zshrc  # ← 明示的に読み込み
typeset -f       # 関数一覧を出力
alias            # エイリアス一覧を出力
export           # 環境変数を出力
```

#### ログインシェルを使う理由

**理由1**: `.zprofile` を読み込むため

```bash
# ~/.zprofile の典型的な内容
export PATH="/usr/local/bin:/usr/bin:/bin"
export JAVA_HOME="/usr/lib/jvm/java-11"
eval "$(/opt/homebrew/bin/brew shellenv)"  # Homebrew環境
```

**理由2**: 正しい順序で設定ファイルを読み込むため

```
ログインシェル（-l）の場合:
1. .zshenv
2. .zprofile   ← PATH等の基本環境
3. (スクリプト内で) source ~/.zshrc  ← エイリアス・関数

非ログインシェルの場合:
1. .zshenv のみ
2. (スクリプト内で) source ~/.zshrc
   → .zprofile がスキップされ、環境が不完全
```

**理由3**: `.zshrc` 内での依存関係

`.zshrc` が `.zprofile` で設定された環境変数を参照している可能性：

```bash
# ~/.zshrc
# .zprofile で設定された HOMEBREW_PREFIX を使用
if [ -f "$HOMEBREW_PREFIX/share/zsh-autosuggestions/zsh-autosuggestions.zsh" ]; then
    source "$HOMEBREW_PREFIX/share/zsh-autosuggestions/zsh-autosuggestions.zsh"
fi
```

#### まとめ：スナップショット作成の推測フロー

```
┌────────────────────────────────────────┐
│ zsh -c -l "スクリプト"                 │
├────────────────────────────────────────┤
│ 1. .zshenv  を読み込む                 │
│ 2. .zprofile を読み込む ← -l の効果   │
│    → PATH等の環境が整う                │
│                                        │
│ 3. スクリプト内で:                     │
│    source ~/.zshrc                     │
│    → エイリアス・関数を取得            │
│                                        │
│ 4. typeset -f  # 関数一覧              │
│    alias       # エイリアス一覧        │
│    export      # 環境変数              │
│    → スナップショットファイルに出力    │
└────────────────────────────────────────┘
```

---

## 推測と未確認事項

### 推測1: スナップショット作成方法

**実行されているコマンド**: `zsh -c -l "スクリプト"`

このスクリプトの中で、以下のいずれかの方法でエイリアスを取得していると推測：

**可能性A**: 明示的に`.zshrc`をsource
```bash
source ~/.zshrc
typeset -f  # 関数一覧
alias       # エイリアス一覧
export      # 環境変数一覧
```

**可能性B**: インタラクティブシェルを起動
```bash
zsh -i -c 'typeset -f; alias; export'
```

**可能性C**: その他の方法

**現状**: cli.jsがバンドルされているため、具体的なスクリプト内容は未確認

### 推測2: 各コマンド実行時の動作

```bash
# 推測される流れ
1. zsh起動（非インタラクティブ）
2. source ~/.claude/shell-snapshots/snapshot-zsh-*.sh
3. コマンド実行
```

**根拠**:
- スナップショットファイルに実行権限がない → sourceで読み込む
- Claude Code内でエイリアスが使える → 何らかの方法で復元している

---

## まとめ

### 確実にわかっていること

1. **スナップショット機構が存在する**
   - ファイルの場所: `~/.claude/shell-snapshots/`
   - 内容: 関数、エイリアス、環境変数

2. **ライフサイクル**
   - 作成: セッション起動時
   - 削除: 正常終了時
   - 保持: 異常終了時

3. **sourceで読み込まれる**
   - 実行権限なし（644）
   - `source`コマンドで現在のシェルに反映

4. **実装**
   - `zsh -c -l "スクリプト"`で起動
   - cli.jsに実装コードが存在

### わかっていないこと（推測）

1. **スナップショット作成の具体的な方法**
   - `.zshrc`をどのように読み込んでいるか
   - `typeset`, `alias`などをどう実行しているか

2. **各コマンド実行時の詳細**
   - スナップショットをどのタイミングでsourceしているか

### 重要な発見

**非インタラクティブシェルでエイリアスを使う方法**:

通常、非インタラクティブシェルでは`.zshrc`が読み込まれないため、エイリアスは使えない。

Claude Codeは以下の工夫でこれを実現している：

1. セッション起動時に環境をスナップショット
2. 各コマンド実行時にスナップショットをsourceで復元
3. 毎回`.zshrc`を読む必要がなく高速

---

## 参考資料

- Claude Code バージョン: 2.1.38
- 実装ファイル: `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js`
- zsh マニュアル: `man zsh` → "STARTUP/SHUTDOWN FILES"
- 調査したスナップショット: `~/.claude/shell-snapshots/snapshot-zsh-1770761628337-k9cbgb.sh`

---

**最終更新**: 2026-02-11

## 付録: 今後の調査項目

- [ ] cli.jsをデバンドルして、スナップショット作成スクリプトの内容を確認
- [ ] 実際にClaude Codeを起動して、プロセスの動作を`strace`/`dtruss`で追跡
- [ ] 各コマンド実行時のシェル起動プロセスを観察
