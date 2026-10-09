---
title: Linuxコマンドを深掘り：top
author: koma77
date: 2026-09-01 00:00:00 +0900
categories: [Linuxコマンドを深掘り]
tags: [Linux]
render_with_liquid: false
---

# 「Linuxコマンドを深掘り」について
普段なんとなく使っているコマンドの知らない一面を知るために、ちょっとだけ深掘りしてみたもの


# 「top」について

## ManPage
[Man page of TOP](https://linuxjm.sourceforge.io/html/procps/man1/top.1.html)

## 実行例

![topコマンド](/assets/img/posts/top-sc.png)

## それぞれの内容

### １行目
```
top - 04:25:38 up 40 min,  1 user,  load average: 0.42, 0.42, 0.37
```

- `04:25:38`: 現在時刻
- `up 40 min`: 起動からの経過時間
- `1 user`: 現在のログインユーザ
- `load average: 0.42, 0.42, 0.37`: ロードアベレージ（実行待ち、I/O待ちのプロセス数）の直近１分、５分、１５分の平均値

実は、`top - `以降の文字列は `uptime` の出力と同じだったりする

### ２行目
```
Tasks: 413 total,   1 running, 411 sleeping,   0 stopped,   1 zombie
```

- `413 total`: 存在するプロセス数
- `1 running`: 実行中 or 実行可能プロセス数
- `411 sleeping`: 休止状態 or 待機中プロセス数
- `0 stopped`: 一時停止(`SIGTSTP`/`Ctrl+Z`)プロセス数
- `1 zombie`: ゾンビ(親プロで回収されるのを待っている)プロセス数

ZabbixとかApacheとか動かしてるとゾンビプロセス出来がち

### ３行目
```
%Cpu(s):  0.7 us,  0.4 sy,  0.0 ni, 98.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st 
```

- `0.7 us`: ユーザ空間の処理に使われた時間
- `0.4 sy`: カーネル空間の処理に使われた時間
- `0.0 ni`: nice値を落として実行したユーザ(空間のプロセスの)処理に使われた時間
- `98.9 id`: 何も処理をしていない時間
- `0.0 wa`: ディスクやネットワーク等のI/O待ちに使われた時間
- `0.0 hi`: ハードウェア割り込み(キーボード、NIC、ディスクコントローラ等からの通知処理)に使われた時間
- `0.0 si`: ソフトウェア割り込み(ネットワークパケットの受信処理等)に使われた時間
- `0.0 st`: 仮想化環境でホストOS(他の仮想マシン含む)にCPUリソースを奪われた時間

`wa`が高いとI/Oがボトルネックになってる(大概ディスクで詰まってる)

クラウド等の仮想環境では、Noisy Neighbor確認で `st(steal)`値を見たりする

### ４行目
```
MiB Mem :  64179.9 total,  46034.7 free,   6311.5 used,  12730.1 buff/cache     
```

- `64179.9 total`: 物理メモリ量
- `46034.7 free`: 完全な空きメモリ量
- `6311.5 used`: プロセスが実際に専有して使用しているメモリ量
- `12730.1 buff/cache`: OSがディスクキャッシュ等として一時的に保持しているメモリ量

Linux自体が空きメモリをなるたけキャッシュに使用しようとするため、freeだけ見れば18GiB近く使っている計算になる  
が、実際には即時開放できる buff/cache や shared 等もあるため精々7GiB程度しか使ってない

ただし、buff/cache によってI/O待ちが軽くなり、結果として処理全体の時間が短く済むこともあるため、メモリはあればあるほどよろし

### ５行目
```
MiB Swap:      0.0 total,      0.0 free,      0.0 used.  57615.7 avail Mem 
```

５行目は４行目がスワップになっただけなので割愛

ちなみに `57615.7 avail Mem`はスワップではなく、物理メモリに対する使用可能なメモリ量なのでスワップがオフ(0byte)でも表示される

### ６行目以降
```
    PID   UID  PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                     
   1985  1000  20   0 5516960 302232 154068 S   2.7   0.5   0:35.93 cinnamon                                                                                
```

- `PID`: プロセスID `pkill`とか`ps auxfww`とかでも共通利用できる
- `UID`: ユーザID 本当は`USERNAME`が表示されるが念の為`UID`に置換
- `PR`: Priority値 プロセス優先度で低値ほど優先度が高い(直接は変えられなくてカーネルがNice値やスケジューリングポリシーで判断)
- `NI`: Nice値 スケジューリング優先度で低値ほど優先度が高い(ユーザで直接変更可能)
- `VIRT`: 仮想メモリサイズ プロセスが要求した全ての領域で共有メモリや未割り当ても含む
- `RES`: 物理メモリサイズ 実際の物理メモリ上での使用領域
- `SHR`: 共有メモリサイズ 他のプロセスと共有可能なライブラリの使用領域
- `S`: プロセスステータス(`R`:Running/`S`:Interruptible Sleep/`D`:Uninterruptible Sleep/`T`:Stopped/`Z`:Zombie/`I`:Idle)
- `%CPU`: CPU使用率
- `%MEM`: `RES`の物理メモリに対する占有率
- `TIME+`: CPU時間 起動してからこれまでに消費したCPU時間の合計
- `COMMAND`: 実行されているプロセスの名前

ちなみに `cinnamon`というのはLinuxMintで使用されるデスクトップ環境のこと(`Xorg`とか`Wayland`とか)

実はここの表示項目は変えられたりする  
(実際に、この例ではUSERNAMEをUIDに変更していたりする)

1. 表示中に <kbd>f</kbd> キーを押す  
すると「Fields Management」が表示される  
![topコマンド-Fields Management](/assets/img/posts/top-sc-fm.png)
2. 表示したい項目を <kbd>↑</kbd> <kbd>↓</kbd>で選択して、<kbd>d</kbd> を押す  
すると `*`が項目の左側につく  
![topコマンド-Fields Management-有効化](/assets/img/posts/top-sc-fm-ena.png)
3. 非表示したい項目を <kbd>↑</kbd> <kbd>↓</kbd>で選択して、<kbd>d</kbd> を押す  
すると `*`が項目の左側から消える  
![topコマンド-Fields Management-無効化](/assets/img/posts/top-sc-fm-dis.png)
4. 順番を変更したい項目を <kbd>↑</kbd> <kbd>↓</kbd>で選択して、<kbd>→</kbd> を押す  
すると選択した項目がハイライトされ、<kbd>↑</kbd> <kbd>↓</kbd>で移動させることができる  
![topコマンド-Fields Management-移動](/assets/img/posts/top-sc-fm-mov.png)  
確定したい場合は <kbd>←</kbd> を押す  
![topコマンド-Fields Management-移動終了](/assets/img/posts/top-sc-fm-mov-end.png)
5. <kbd>q</kbd> を押すと通常の `top`コマンド画面に戻る

# ひとこと
topを開くだけで、CPU・メモリ・プロセスの状態を一目で確認できる  
最初の5行で大体の状況が分かるのが強み

```
top -n 1
```