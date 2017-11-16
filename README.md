# 羊と狼パズルをSpinで解く

## 方針
なるべくシンプルに。

* 通信チャネルは使わない。大域変数のみ使う。

## ゲームのモデリング

### 登場人物

* 羊
* 狼
* 舟
* 岸A
* 岸B

### 必要な変数

* 羊: 岸A, 舟, 岸B それぞれの人数 (lambA, lambS, lambB)
* 狼: 　　　　　　〃　　　　　　 (wolfA, wolfS, wolfB)
* 舟: どちらの岸にいるか (ship)

### 振る舞い

* 舟がいる岸で，岸から舟に羊・狼が移動できる (定員2名)。
* 舟に1人以上乗っていたら舟を向こう岸に移動できる。
  * 移動したら乗員は一旦岸に移動する。
* 岸AまたはBにいる羊の数は0か同じ岸の狼の数以上。

下記は検証項目 (LTL式) で表す:

* 岸Bに全員移動したらゲームクリア

### Promela化

```
/* ゲームクリアの条件 */
#define goal  (lambB == N && wolfB == N)

#define N              3  /* 羊・狼 各人数 */
#define SHIP_CAPACITY  2  /* 舟の定員 */

#define BANK_A  0               /* 舟が岸Aにいる */
#define BANK_B  1               /* 舟が岸Bにいる */
#define OPP_BANK(b)  (1 - (b))  /* 向こう岸 */
#define N_CREW  (lambS + wolfS) /* 乗員数 */

byte lambA = N, lambS = 0, lambB = 0;
byte wolfA = N, wolfS = 0, wolfB = 0;
byte ship = BANK_A;

active proctype Game() {
  do
  :: 舟が岸Aにいる かつ 舟に空きがある ->
     羊か狼かどちらかを一人舟に移す
  :: 舟が岸Bにいる かつ 舟に空きがある ->
     羊か狼かどちらかを一人舟に移す
  :: 舟に誰か乗っている かつ 羊が安全 ->
     舟を向こう岸に移動;
     乗員を舟から岸に下ろす;
     羊が安全 (そうでなければここで停止)
  od
}
```

### 検証による求解

`<> goal` をnever claim化して検証 <br />
( `!<> goal` であることを検証 → goal に到達する実行系列を反例)。

```
$ spin -f '<> goal' >wolf.ltl      -- never claim生成
$ spin -a -N wolf.ltl wolf.pml     -- pan.cを生成
$ cc -O -DSAFETY pan.c -o pan      -- panをコンパイル
$ ./pan                            -- 検証
$ spin -t -p -g wolf.pml           -- 実行系列を表示
```

