# memory_order
* atomic[meta header]
* std[meta namespace]
* enum[meta id-type]
* cpp11[meta cpp]

```cpp
namespace std {
  // C++11
  enum memory_order {
    memory_order_relaxed, memory_order_consume, memory_order_acquire,
    memory_order_release, memory_order_acq_rel, memory_order_seq_cst
  };

  // C++20
  enum class memory_order {
    relaxed, consume, acquire, release, acq_rel, seq_cst
  };
  inline constexpr memory_order memory_order_relaxed = memory_order::relaxed;
  inline constexpr memory_order memory_order_consume = memory_order::consume;
  inline constexpr memory_order memory_order_acquire = memory_order::acquire;
  inline constexpr memory_order memory_order_release = memory_order::release;
  inline constexpr memory_order memory_order_acq_rel = memory_order::acq_rel;
  inline constexpr memory_order memory_order_seq_cst = memory_order::seq_cst;

  // C++26
  enum class memory_order : unspecified {
    relaxed = 0,              acquire = 2,
    release = 3, acq_rel = 4, seq_cst = 5
  };
  inline constexpr memory_order memory_order_relaxed = memory_order::relaxed;
  inline constexpr memory_order memory_order_acquire = memory_order::acquire;
  inline constexpr memory_order memory_order_release = memory_order::release;
  inline constexpr memory_order memory_order_acq_rel = memory_order::acq_rel;
  inline constexpr memory_order memory_order_seq_cst = memory_order::seq_cst;
}
```
* unspecified[italic]


## 概要
コンパイラに許可されている最適化の一つに、「プログラムの意味を変えない限りにおいて、メモリアクセスの順番を変えたり、省略したりしてもよい」というものがある。また、マルチコアCPUにおいては、あるCPUコアによるメモリアクセスの順序が他のコアからも同じように見えるとは限らない。このような挙動はマルチスレッドプログラミングにおいて問題になることがある。
この問題への対処として、C++11では各スレッドの実行に順序付けをするための「よりも前に発生する」という関係を定義し、それによってあるスレッドでの操作が他スレッドから可視になるか否かを定めている。
atomic変数においては、"release"操作によって書き込まれた値を"acquire"操作によって別のスレッドが読み出した場合に、そのrelease操作とacquire操作の間に順序付けが行われる。以下に例を挙げる。

```cpp example
#include <iostream>
#include <atomic>
#include <thread>
int data;
std::atomic<bool> ready(false);

void f()
{
  while (!ready.load(std::memory_order_acquire)) {
  }

  std::cout << data << std::endl;   // (2)
}

int main()
{
  std::thread t(f);

  data = 3;   // (1)
  ready.store(true, std::memory_order_release);

  t.join();
}
```
* std::memory_order_acquire[color ff0000]
* std::memory_order_release[color ff0000]
* load[link atomic/load.md]
* store[link atomic/store.md]

### 出力
```
3
```

[`atomic<bool>`](atomic.md)型の変数`ready`への読み書きに注目すると、`main()`では変数`ready`に `true` を"release"操作として書き込み、`f()`では"acquire"操作としての読み込みを `true` が返されるまで繰り返している。よって、`f()`の`while`ループを抜けた時点で、`main()`の`ready.store()`と`f()`の`ready.load()`の間に順序付け(「よりも前に発生する」関係)が成立している。
ここでさらに変数`data`への読み書き(1), (2)に注目すると、(1)は`ready.store()`より前、(2)は`ready.load()`より後にあるので、以下のようなスレッド間の順序付け(「よりも前に発生する」関係)が成立することになる。
   (1) → `ready.store()` → `ready.load()` → (2)
よって、(1)における書き込みが(2)の時点で可視であることが保証される。
このようにしてC++のマルチスレッドプログラムにおける実行順序および可視性を理解することができる。

以下の列挙値はatomic変数への操作に指定可能な順序付けの種類を表す。

| 列挙値 | 説明 |
|--------|------|
| `relaxed` | スレッド間の順序付けの効果は一切持たない。 |
| `consume` | C++26で非推奨 |
| `acquire` | acquire操作としての読み込みを行うことを指示する。[store()](atomic/store.md)など、書き込みのみを行う操作に対しては指定できない。 |
| `release` | release操作としての書き込みを行うことを指示する。[load()](atomic/load.md)など、読み込みのみを行う操作に対しては指定できない。 |
| `acq_rel` | 読み込みと書き込みを同時に行う操作(Read-Modify-Write操作)に対してのみ指定することができ、acquireとreleaseを合わせた効果を持つ。 |
| `seq_cst` | acquire(読み込み操作の場合)、release(書き込み操作の場合)、acq_rel(Read-Modify-Write操作の場合)としての効果を持つ。さらに、同じseq_cstが指定されている他のatomic操作との間での順序一貫性も保証する。これは最も強い保証であり、標準のatomic操作におけるデフォルトのメモリオーダーとして使用される。「seq_cst」は「sequential consistency(順序一貫性)」を意味する。 |


## 備考
非推奨化された`consume`列挙値の挙動はacquire操作と似ているが、それより弱い順序付けでの読み込みを行うことを指示する。acquire操作は後続の全ての操作に対して順序付けを行うのに対し、consume操作は読み込まれた値に依存(ただし条件分岐による依存は除く)する操作のみに順序付けを保証する点が異なる。
複雑なconsume操作を正しく実装するC++コンパイラは登場せず、より単純なacquire操作として扱われていた。C++20では仕様再検討に伴う一時的な利用回避が宣言され、最終的には役に立たないとの判断からC++26で非推奨となった。


## バージョン
### 言語
- C++11

### 処理系
- [Clang](/implementation.md#clang): ??
- [GCC](/implementation.md#gcc): 4.7.0 [mark verified]
- [ICC](/implementation.md#icc): ??
- [Visual C++](/implementation.md#visual_cpp): 2012 [mark verified], 2013 [mark verified]


## 参照
- [そろそろvolatileについて一言いっておくか - yamasaのネタ帳](http://d.hatena.ne.jp/bsdhouse/20090720/1248085754)
- [次期C++に導入されるメモリバリアについて解説してみる - yamasaのネタ帳](http://d.hatena.ne.jp/bsdhouse/20090816/1250446250)
- [C++0xのメモリバリアをより深く解説してみる - yamasaのネタ帳](http://d.hatena.ne.jp/bsdhouse/20090929/1254237835)
- [C/C++11 mappings to processors](http://www.cl.cam.ac.uk/~pes20/cpp/cpp0xmappings.html)
- [N1525: Memory-Order Rationale](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1525.htm)
- [The Purpose of memory_order_consume in C++11](http://preshing.com/20140709/the-purpose-of-memory_order_consume-in-cpp11/)
- [（抄訳）N4215 `memory_order_consume`の利用と実装に向けて［§5-6のみ］](http://d.hatena.ne.jp/yohhoy/20141115/p1)
- [P0439R0 Make `std::memory_order` a scoped enumeration](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0439r0.html)
    - C++20で`memory_order`をスコープ付き列挙型に変更。
- [P0371R1 Temporarily discourage `memory_order_consume`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0371r1.html)
- [P3475R2 Defang and deprecate memory_order::consume](https://open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3475r2.pdf)
    - C++26で`memory_order::consume`列挙子を非推奨化。
