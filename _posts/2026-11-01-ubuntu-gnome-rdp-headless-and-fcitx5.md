---
layout: post
title: "Ubuntu 26.04 (GNOME 50) を RDP で使う: 解像度の壁をヘッドレスモードで越え、日本語入力を fcitx5 で安定させるまで"
date: 2026-11-01 02:00:00 +0900
categories: [Linux, GNOME]
tags: [ubuntu, gnome, rdp, gnome-remote-desktop, fcitx5, mozc, vmware]
description: VMware上のUbuntu 26.04をRDPで使う際の、解像度制限(ヘッドレスモードで解決)と日本語入力(IBus→fcitx5、半角/全角キー問題)の記録。
---

VMware 上の Ubuntu 26.04 LTS (GNOME 50) を Windows から RDP で操作する環境を整えた記録。
ハマりどころが多かったので、「何が原因で」「何を試して駄目で」「最終的にどうしたか」を残しておく。

## 環境

| 項目 | 値 |
|---|---|
| ゲスト OS | Ubuntu 26.04.1 LTS (GNOME 50, Wayland) |
| 仮想化 | VMware (VMware SVGA II / `vmwgfx` ドライバ) |
| RDP サーバー | `gnome-remote-desktop` 50.2 |
| 日本語入力 | fcitx5 5.1 + Mozc |
| RDP クライアント | Windows |

---

## 1. RDP の解像度が 1280x768 から上がらない

### 症状

GNOME の「設定 → システム → リモートデスクトップ → デスクトップ共有」で RDP を有効にして接続すると、
クライアントをフルスクリーンにしても **1280x768** のまま。

### 原因

デスクトップ共有モードは **ローカルのログイン済み画面をそのままミラー** するだけなので、
解像度はローカルの仮想モニタに縛られる。その仮想モニタが提供するモードを見ると:

```
$ cat /sys/class/drm/card0-Virtual-1/modes
1280x768 1280x768 1280x720 1024x768 800x600 800x600 848x480 640x480
```

1280x768 が上限。1280×768×4B ≒ 3.9MB は収まり、次の 1280×800 (4.1MB) は収まらないことから、
**VM のビデオメモリが 4MB** で制限されていると判断した(`sudo dmesg | grep vmwgfx` で確定できる)。

### 選択肢

| 方式 | 内容 | クライアント解像度への追従 |
|---|---|---|
| A. ヘッドレスモード(リモートログイン) | システム側の `gnome-remote-desktop.service` が接続ごとに仮想モニタを作る | **あり**(ウィンドウのリサイズにも追従) |
| B. VM のビデオメモリを増やす | `.vmx` に `svga.vramSize = "134217728"` 等 | なし(VMware コンソールのサイズには追従) |

「クライアントの解像度に合わせたい」が目的だったので **A** を選択。KMS/VRAM を使わないので VMware 側の制限も受けない。

### 手順(ヘッドレスモード)

```bash
sudo grdctl --system rdp enable
sudo grdctl --system rdp set-credentials <RDP用ユーザー名> <RDP用パスワード>
sudo systemctl enable --now gnome-remote-desktop.service
sudo grdctl --system status
```

`status` で `TLS certificate:` が空なら自分で作って登録する:

```bash
sudo openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/CN=$(hostname)" \
  -keyout /etc/gnome-remote-desktop/rdp-tls.key -out /etc/gnome-remote-desktop/rdp-tls.crt
sudo grdctl --system rdp set-tls-key  /etc/gnome-remote-desktop/rdp-tls.key
sudo grdctl --system rdp set-tls-cert /etc/gnome-remote-desktop/rdp-tls.crt
```

GUI なら「設定 → システム → リモートデスクトップ → **リモートログイン**」タブでも同じことができる。

### ハマった点 (3つ)

**(1) 証明書の所有者**
`openssl` を `sudo` で実行すると鍵が `root:root 0600` になるが、デーモンは `gnome-remote-desktop` ユーザーで動く。
ログに `RDP TLS certificate and key not yet configured properly` と出たらこれ。

```bash
sudo chown gnome-remote-desktop:gnome-remote-desktop /etc/gnome-remote-desktop/rdp-tls.{key,crt}
sudo systemctl restart gnome-remote-desktop.service
```

**(2) ポート 3389 の競合**
ユーザー側の「デスクトップ共有」が有効なままだと 3389 をそちらが掴んでいる。システム側に譲る:

```bash
grdctl rdp disable          # ユーザー側(sudo 不要)
```

**(3) ufw**
`ss -ltnp` で LISTEN していてもクライアントから繋がらず、ジャーナルに接続試行が一切出ない場合はファイアウォール。

```bash
sudo ufw allow 3389/tcp comment 'GNOME Remote Desktop'
```

### 結果

接続すると GDM のログイン画面が出て、Linux ユーザーでログインするとクライアント解像度の仮想モニタでセッションが始まる。
`set-credentials` で設定したものは RDP 層の認証用で、Linux のログイン情報とは別物。

> ローカル(tty)のセッションとは**別セッション**になる。同じユーザーでローカルにログインしたままだと競合しうるので、ローカル側はログアウトしておく。

---

## 2. 日本語入力: IBus+Mozc で RDP が不安定 → fcitx5 に移行

### 症状

`ibus-mozc` を入れて GNOME の入力ソースに「日本語 (Mozc)」を追加。
Mozc に切り替えて日本語入力モードにすると **RDP セッションが不安定になり操作不能** になった。

GNOME のヘッドレスセッション + IBus の組み合わせで入力が固まる報告は他にもあり、fcitx5 への切り替えで解消したという事例を見ていたので fcitx5 に移行した。

### fcitx5 の導入(GNOME Wayland 固有の罠が多い)

```bash
sudo apt install fcitx5 fcitx5-mozc fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 fcitx5-frontend-qt5 fcitx5-config-qt ibus-mozc-
```

(末尾の `ibus-mozc-` はマイナス付きで削除。IBus 本体は GNOME Shell が必要とするので残す)

GNOME の入力ソースは `jp` 配列だけに戻す(IBus を素通しにする):

```bash
gsettings set org.gnome.desktop.input-sources sources "[('xkb', 'jp')]"
```

ここからが罠。

#### 罠1: `im-config -n fcitx5` は Ubuntu 26.04 では効かない

`/etc/profile.d/im-config_wayland.sh` の中で環境変数設定が `if false` で無効化されており、
`/etc/xdg/autostart/im-launch.desktop` も `Exec=/usr/bin/true` に置き換えられている。
つまり **im-config 経由では Wayland セッションで何も起きない**。自分で環境変数と自動起動を用意する必要がある。

#### 罠2: GNOME Shell が `XMODIFIERS=@im=ibus` / `QT_IM_MODULE=ibus` を強制する

GNOME Shell は IBus を起動する際にこれらをセッション環境に上書きする。`GTK_IM_MODULE` は上書きされないので、
GTK アプリと Electron アプリ(Chromium は GTK の IM コンテキストを使う)は `GTK_IM_MODULE=fcitx` で fcitx5 に向けられる。

さらに環境変数に依存しない保険として GSettings の `gtk-im-module` も設定する。これは起動済みセッションにも即時反映される:

```bash
gsettings set org.gnome.desktop.interface gtk-im-module 'fcitx'
```

環境変数は `~/.config/environment.d/90-fcitx5.conf` に置く(ユーザー systemd マネージャ経由でセッションに入る):

```ini
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
GLFW_IM_MODULE=ibus
INPUT_METHOD=fcitx
```

#### 罠3: autostart に `X-GNOME-Autostart-Phase` を書くと GNOME 50 は無視する

ジャーナルにこう出て起動されない:

```
gnome-session-service: App org.fcitx.Fcitx5.desktop sets X-GNOME-Autostart-Phase,
but gnome-session no longer manages session services
```

#### 罠4: `fcitx5 -d`(デーモン化)すると systemd に殺される

GNOME は autostart アプリを systemd の一時ユニットとして起動する。`-d` でフォークして親が終了するとユニット終了扱いになり、
フォーク後の本体も cgroup ごと止められる。**`-d` を付けない**。

正解の `~/.config/autostart/org.fcitx.Fcitx5.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=Fcitx 5
Exec=fcitx5 -r
Icon=org.fcitx.Fcitx5
Terminal=false
NoDisplay=true
```

#### 罠5: ロケールが `en_US` だと Mozc が自動追加されない

`~/.config/fcitx5/profile` を先に書いておく:

```ini
[Groups/0]
Name=Default
Default Layout=jp
DefaultIM=mozc

[Groups/0/Items/0]
Name=keyboard-jp
Layout=

[Groups/0/Items/1]
Name=mozc
Layout=

[GroupOrder]
0=Default
```

ここまでで、ログアウト → 再ログインすると fcitx5 が立ち上がり、Ctrl+Space で日本語⇔英語が切り替わる。
日本語入力中も RDP セッションは安定した。

---

## 3. 半角/全角キーが RDP 経由だと「押しっぱなし」になる

### 症状

半角/全角キーを押すと **日本語⇔英語が超高速で切り替わり続ける**。Ctrl+Space は正常。

### 原因

RDP クライアント(Windows)が半角/全角キーの **キーリリースを正しく送ってこない**。
サーバー側では押下状態のままになり、

- キーリピート ON → mutter が ~30Hz でキーイベントを生成し続ける → 高速トグル
- 加えて `gnome-remote-desktop` は同じキーの重複押下を無視するので、もう一度押しても届かない → 再接続まで死んだキーになる

xrdp にも同じ問題があり、X11 時代は `xset -r 49`(そのキーだけリピート無効)が定番の対処だった。
Wayland (mutter) ではそれが使えない。

### 試して駄目だったこと

| 試行 | 結果 |
|---|---|
| GNOME のキーリピートを全体 OFF | 高速トグルは止まるが、RDP クライアントは長押しリピートを送ってこないので Backspace 長押しが効かなくなる。却下 |
| `~/.config/xkb/` にユーザー xkb **レイアウト variant** を作り、対象キーを `repeat = No` | gnome-shell が未知の variant を拒否し、US 配列にフォールバック |
| `~/.config/xkb/` にユーザー xkb **オプション** を作る(gnome-shell の検証対象外) | mutter が `~/.config/xkb` を読まず、適用されない |
| fcitx5 で `EnumerateGroupForwardKeys=Zenkaku_Hankaku` にして no-op として吸収 | グループが1つしかないと fcitx5 はキーを消費せず Mozc に素通り |

(xkb の key 名はこの環境では `<HZTG>` ではなく **`<TLDE>`** (keycode 49) だった、という副産物の知見もあった)

### 最終的な対処

**半角/全角キーは使わない**と割り切り、代わりのキーを用意して、押しても暴走しないよう無害化した。

**fcitx5 のホットキー** (`~/.config/fcitx5/config`):

```ini
[Hotkey]
EnumerateWithTriggerKeys=True

[Hotkey/TriggerKeys]
0=Control+space

[Hotkey/AltTriggerKeys]

[Hotkey/ActivateKeys]
0=Henkan_Mode

[Hotkey/DeactivateKeys]
0=Muhenkan
```

- Ctrl+Space: トグル
- 変換キー: 日本語 ON、無変換キー: 日本語 OFF (Windows 11 の IME と同じ流儀)
- `AltTriggerKeys` を空にして Shift 単独でのトグルを無効化
- 変換キーのキーシンボルは `Henkan` ではなく **`Henkan_Mode`**(`jp` 配列ではこちら)。最初 `Henkan` と書いて効かなかった

**Mozc 側で半角/全角の割り当てを削除**(押しっぱなしになっても Mozc が反応しないように):

`/usr/lib/mozc/mozc_tool --mode=config_dialog` → 一般 → キー設定「編集...」→ 入力キーが `Hankaku/Zenkaku` の行をすべて選択 → 編集 ▼ → 選択された項目を削除 → 適用。
キー設定は自動で「カスタム」になる。

英字モード中に押した場合はアプリに未定義キーが届くだけで無害。

> 物理的な半角/全角キーをどうしても使いたいなら、クライアント側(Windows なら PowerToys Keyboard Manager)で
> 「半角/全角 → Ctrl+Space」に置き換えるのが筋。リリースを送らないのはクライアントの挙動なので、別キーに化けさせれば正常に動くはず。

---

## おまけ: Claude Desktop (Linux β) のインストール

公式 apt リポジトリがある。鍵のフィンガープリントは `31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE`。

```bash
sudo curl -fsSLo /usr/share/keyrings/claude-desktop-archive-keyring.asc https://downloads.claude.ai/claude-desktop/key.asc
gpg --show-keys /usr/share/keyrings/claude-desktop-archive-keyring.asc   # フィンガープリント確認
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/claude-desktop-archive-keyring.asc] https://downloads.claude.ai/claude-desktop/apt/stable stable main" \
  | sudo tee /etc/apt/sources.list.d/claude-desktop.list
sudo apt update && sudo apt install claude-desktop
```

推奨パッケージとして QEMU 一式(Cowork 用)が入る。VM 上でネストした仮想化が使えないなら `--no-install-recommends` で省いてよい。
Electron アプリなので、上記の fcitx5 設定(`gtk-im-module`)で日本語入力できる。

---

## まとめ

- RDP の解像度問題は「画面共有」ではなく **ヘッドレスモード(リモートログイン)** で解決する。VRAM の制限も関係なくなる
- Ubuntu 26.04 の GNOME Wayland で fcitx5 を使うなら、im-config は当てにせず **environment.d + gtk-im-module + 素の XDG autostart (`-d` なし)** で組む
- RDP 経由の半角/全角キーは構造的に壊れている。**Ctrl+Space / 変換 / 無変換** に逃げて、Mozc 側で無害化する
