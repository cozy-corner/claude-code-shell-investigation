# Claude Code のシェル環境スナップショット機構 - 調査報告

**調査日**: 2026年2月11日
**調査環境**: macOS, zsh, Claude Code 2.1.38

## 目次

1. [確認できた事実](#確認できた事実)
2. [シェルの基礎知識](#シェルの基礎知識)
3. [実験による検証](#実験による検証)
4. [実装の確認](#実装の確認)
5. [スナップショット作成スクリプトの詳細（確認済み）](#スナップショット作成スクリプトの詳細確認済み)
6. [まとめ](#まとめ)

---

## 確認できた事実

### 1. スナップショットファイルの存在と構造

```bash
$ ls ~/.claude/shell-snapshots/
snapshot-zsh-1770761628337-k9cbgb.sh
snapshot-zsh-1770760687206-uml7wn.sh
...（95個のファイル）
```

**基本情報**:
- 保存場所: `~/.claude/shell-snapshots/`
- ファイル名形式: `snapshot-zsh-<タイムスタンプ>-<ランダム文字列>.sh`
- パーミッション: `644` (rw-r--r--) - 実行権限なし
- シェバン（`#!/bin/zsh`）なし - `source`で読み込むため不要

**ファイル名の詳細**（cli.js 行2608, 2610付近で確認）:
- `<タイムスタンプ>`: `Date.now()`で生成（セッション起動時刻）
- `<ランダム文字列>`: `Math.random().toString(36).substring(2,8)`で生成（衝突回避用、約6文字）
- セッションIDではなく、単なるランダム識別子

**ライフサイクル**（確認済み）:
- **作成**: セッション起動時（`aSY()` → `oc4()` 関数）
- **削除**: 正常終了時（`b1().unlinkSync(O)`）
- **保持**: 異常終了時（削除処理が実行されず残存）

### 2. スナップショットの内容

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

### 3. Claude Code内でのエイリアス動作

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
  - `_`: 実行されるスクリプト（`mSY`関数で生成、詳細は後述）
- 環境変数:
  - `SHELL:A`: シェルパスを設定
  - `GIT_EDITOR:"true"`: エディタを無効化
  - `CLAUDECODE:"1"`: Claude Code実行フラグ

**スクリプト内容の詳細は「スナップショット作成スクリプトの詳細」セクションを参照**

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

## スナップショット作成スクリプトの詳細（確認済み）

### スクリプト生成関数（行2587-2608）

```javascript
async function mSY(A,q,K){
  let Y=qyA(A),  // .zshrc または .bashrc のパス
  z=Y.endsWith(".zshrc"),
  w=K?bSY(Y):!z?'echo "shopt -s expand_aliases" >> "$SNAPSHOT_FILE"':"",
  H=await BSY();

  return`SNAPSHOT_FILE=${L7([q])}
    ${K?`source "${Y}" < /dev/null`:"# No user config file to source"}

    # First, create/clear the snapshot file
    echo "# Snapshot file" >| "$SNAPSHOT_FILE"

    # Unset all aliases to avoid conflicts with functions
    echo "unalias -a 2>/dev/null || true" >> "$SNAPSHOT_FILE"

    ${w}  // 関数定義を出力
    ${H}  // PATH等の環境変数を出力
    ...
  `
}
```

### 確認できたこと

**1. `.zshrc`の読み込み方法**（行2588）

```bash
source ~/.zshrc < /dev/null  # stdin をクローズして source
```

- 明示的に`.zshrc`をsource
- `< /dev/null`でstdinをクローズ（対話的プロンプトを防ぐ）

**2. スナップショット作成の流れ**（行2524-2586）

```bash
# 1. 関数定義を取得
echo "# Functions" >> "$SNAPSHOT_FILE"
typeset -f > /dev/null 2>&1  # 関数を強制ロード
typeset +f | grep -vE '^(_|__)' | while read func; do
  typeset -f "$func" >> "$SNAPSHOT_FILE"  # ユーザー関数のみ
done

# 2. シェルオプション
echo "# Shell Options" >> "$SNAPSHOT_FILE"
setopt | sed 's/^/setopt /' | head -n 1000 >> "$SNAPSHOT_FILE"

# 3. エイリアス
echo "# Aliases" >> "$SNAPSHOT_FILE"
alias | sed 's/^alias //g' | sed 's/^/alias -- /' >> "$SNAPSHOT_FILE"

# 4. 環境変数
export PATH="..."
```

**3. 各コマンド実行時の復元**（nW6関数内）

```javascript
let G=[];
if(J){  // スナップショットファイルがあれば
  if(!QSY(J))  // ファイルが存在しない場合は再作成
    h(`Snapshot file missing, recreating: ${J}`),
    zyA.cache?.clear?.(),
    J=(await zyA()).snapshotFilePath;
  if(J){
    let B=tA()==="windows"?cx(J):J;
    G.push(`source ${L7([B])}`)  // スナップショットをsource
  }
}
```

**実行される流れ**:
1. zsh起動（非インタラクティブ、`-l`フラグ付き）
2. `source ~/.claude/shell-snapshots/snapshot-zsh-*.sh`
3. ユーザーのコマンド実行

---

## 残された未確認事項

特になし。スナップショット機構の主要な動作はすべて確認済み。

---

## まとめ

### Claude Codeのスナップショット機構（完全解明）

**1. スナップショットの構造**
- ファイルの場所: `~/.claude/shell-snapshots/`
- ファイル名: `snapshot-zsh-<timestamp>-<random>.sh`
  - `<random>`: 衝突回避用ランダム識別子（セッションIDではない）
- 内容: 関数、エイリアス、シェルオプション、環境変数
- パーミッション: 644（実行権限なし、sourceで読み込む）

**2. ライフサイクル**
- 作成: セッション起動時（`aSY()` → `oc4()` → `ISY()`）
- 削除: 正常終了時
- 保持: 異常終了時（デバッグ用）

**3. スナップショット作成フロー**

```
セッション起動
  ↓
zsh -l -c "スクリプト"
  ↓
1. .zshenv を読み込む
2. .zprofile を読み込む（← -l の効果、PATH等）
3. source ~/.zshrc < /dev/null（← 明示的にsource）
4. typeset -f でユーザー関数を取得
5. alias でエイリアスを取得
6. スナップショットファイルに出力
```

**4. スナップショット復元フロー**

```
Bashツール実行
  ↓
Tzz() 関数
  ↓
nW6() 関数
  ↓
zsh -l -c "source <snapshot>; <user command>"
  ↓
1. .zshenv、.zprofile を読み込む
2. source ~/.claude/shell-snapshots/snapshot-zsh-*.sh
3. ユーザーコマンド実行
```

**5. 重要な発見：非インタラクティブシェルでエイリアスを使う方法**

通常、非インタラクティブシェルでは`.zshrc`が読み込まれないため、エイリアスは使えない。

Claude Codeの解決方法：
- セッション起動時に環境を一度だけキャプチャ
- 各コマンド実行時にスナップショットをsourceで復元
- 毎回`.zshrc`を読む必要がなく高速
- ログインシェル（`-l`）で`.zprofile`も読み込み、環境を完全に再現

---

## 参考資料

- Claude Code バージョン: 2.1.38
- 実装ファイル: `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js`
- zsh マニュアル: `man zsh` → "STARTUP/SHUTDOWN FILES"
- 調査したスナップショット: `~/.claude/shell-snapshots/snapshot-zsh-1770761628337-k9cbgb.sh`

---

**最終更新**: 2026-02-11

## 付録: 調査完了項目

- [x] ~~cli.jsをデバンドルして、スナップショット作成スクリプトの内容を確認~~
  - **完了**: `mSY`関数（行2587-2608）で詳細確認済み
  - `source ~/.zshrc < /dev/null` で明示的に読み込み

- [x] ~~各コマンド実行時のシェル起動プロセスを観察~~
  - **完了**: `nW6`関数で`source <snapshot>`を確認
  - 呼び出しフロー: Bashツール → `Tzz()` → `nW6()` を確認

- [x] ~~スナップショットファイル名のランダム文字列の意味~~
  - **完了**: `Math.random().toString(36).substring(2,8)` で生成
  - セッションIDではなく衝突回避用のランダム識別子

**調査結果**: スナップショット機構の主要な動作をすべて解明
