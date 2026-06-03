# libgc を使ってみる

理論の話が続きましたが、この章では手を動かします。保守的GCの代表的な実装
である **libgc**（正式には Boehm-Demers-Weiser GC、略して BDWGC）を実際に
インストールし、C プログラムから使ってみましょう。前章までで「保守的GCは
コンパイラの協力なしに動く」と述べましたが、それがどれほど手軽かを体感
できるはずです。

## libgc とは

libgc は、C と C++ のための保守的なごみ収集ライブラリです。Hans Boehm、
Alan Demers、Mark Weiser らの研究 [Boehm and Weiser](#cite:boehm1988) を
出発点に、長年にわたって開発が続けられてきました。今日では多くの実用
ソフトウェアで使われています。たとえば、

- GNU の言語処理系の一部（過去の GCJ など）
- いくつかの Scheme / Lisp 処理系（前章で触れた Bartlett の Scheme→C を
  含む [Bartlett](#cite:bartlett1989)）
- 各種スクリプト言語や組み込み言語のランタイム

などです。「既存の C コードに、`malloc`/`free` を書き換えるだけで GC を
導入できる」という手軽さが、その普及を支えてきました。

> [!NOTE]
> 「libgc」「BDWGC」「Boehm GC」「boehm-gc」「bdw-gc」はすべて同じものを
> 指します。Linux のパッケージ名やリンク時のライブラリ名（`-lgc`）では
> 「gc」、論文やドキュメントでは「Boehm GC」と呼ばれることが多い、と覚えて
> おけば混乱しません。

## インストール

多くの Linux ディストリビューションでは、パッケージマネージャから簡単に
入れられます。Debian/Ubuntu 系なら次のとおりです。

```bash
# 開発用ヘッダ（gc.h）とライブラリ本体をまとめて入れる
sudo apt install libgc-dev
```

Fedora/RHEL 系なら `sudo dnf install gc-devel`、macOS の Homebrew なら
`brew install bdw-gc` です。

最新版を使いたい、あるいは細かくビルドオプションを調整したい場合は、
ソースからビルドします。

```bash
git clone https://github.com/bdwgc/bdwgc
cd bdwgc
./autogen.sh         # 自動ツールで configure を生成
./configure
make
sudo make install
```

> [!TIP]
> マルチスレッドのプログラムから使う場合は、スレッド対応を有効にして
> ビルドする必要があります（`./configure --enable-threads=posix` など）。
> パッケージ版の `libgc-dev` は通常スレッド対応でビルドされています。
> スレッドとGCの関係は [](integrating.md) で改めて触れます。

## 最初のプログラム：malloc を GC_MALLOC に置き換える

libgc の使い方は驚くほど単純です。`malloc` を `GC_MALLOC` に置き換え、
`free` を **書かない**（消す）だけです。次のプログラムは、大量のメモリを
確保し続けますが、`free` を一切呼びません。それでもメモリは枯渇しません。
libgc が不要になったメモリを自動的に回収するからです。

```c
#include <stdio.h>
#include <gc.h>          /* libgc のヘッダ */

int main(void) {
    GC_INIT();           /* GC を初期化する（最初に一度だけ） */

    for (int i = 0; i < 10000000; i++) {
        /* 毎回 256 バイト確保するが、ポインタはすぐ上書きされる。
           つまり前回確保したメモリは到達不能になり「ごみ」になる。 */
        char *p = (char *)GC_MALLOC(256);
        p[0] = (char)i;                 /* 確保したメモリを使う */

        if (i % 1000000 == 0) {
            /* libgc が把握しているヒープサイズを表示してみる */
            printf("iter %8d, heap size = %zu bytes\n",
                   i, (size_t)GC_get_heap_size());
        }
    }
    return 0;
}
```

ループは1000万回まわり、毎回 256 バイトを確保します。単純計算では
約2.5GB ものメモリを要求していますが、`free` を書いていないのに
メモリは破綻しません。`GC_get_heap_size()` の表示を見ると、ヒープサイズ
がある程度で頭打ちになることが確認できます。確保したメモリがすぐ到達
不能になるため、libgc がそれを繰り返し回収して再利用しているのです。

> [!IMPORTANT]
> `GC_INIT()` はプログラムの最初（`main` の冒頭）で一度だけ呼びます。
> 特に共有ライブラリから libgc を使う場合や、特殊なプラットフォームでは
> 明示的な初期化が必要になるため、習慣として必ず呼んでおくのが安全です。

## コンパイルとリンク

このプログラムをビルドするには、libgc をリンクします。ライブラリ名は
`gc` なので、`-lgc` を指定します。

```bash
cc -o gctest gctest.c -lgc
./gctest
```

ヘッダ `gc.h` が標準の場所になければ `-I` で、ライブラリが見つからなけ
れば `-L` でパスを補います（ソースから `/usr/local` に入れた場合など）。

```bash
cc -I/usr/local/include -o gctest gctest.c -L/usr/local/lib -lgc
```

リンク以外にソースコードへ加える変更は、`#include <gc.h>` と
`malloc`→`GC_MALLOC` の置き換えだけです。これが「非協力的な環境でも動く」
保守的GCの威力です。

## 確保関数の使い分け：ポインタを含むか含まないか

libgc には用途別の確保関数があります。中でも重要なのが
`GC_MALLOC` と `GC_MALLOC_ATOMIC` の使い分けです。

- **`GC_MALLOC(size)`**：確保した領域の中に **ポインタが入っているかも
  しれない** とみなされます。GC はマーク時にこの領域の中身を走査し、
  ポインタらしき値があればたどります。構造体やポインタ配列はこちらを
  使います。
- **`GC_MALLOC_ATOMIC(size)`**：確保した領域の中に **ポインタは絶対に
  入っていない** とプログラマが保証する確保です。「atomic」は「これ以上
  たどる必要がない＝走査の対象としては不可分」という意味です。文字列、
  画像の画素データ、浮動小数点数の配列など、純粋なデータにはこちらを
  使います。

```c
/* ポインタを含むかもしれない構造体 → GC_MALLOC */
struct node {
    int value;
    struct node *next;     /* ポインタを含む！ */
};
struct node *n = (struct node *)GC_MALLOC(sizeof(struct node));

/* 純粋なバイト列（ポインタを含まない）→ GC_MALLOC_ATOMIC */
double *samples = (double *)GC_MALLOC_ATOMIC(1000 * sizeof(double));
```

なぜこの区別が重要なのでしょうか。前章で見た **偽ポインタ** を思い出して
ください。`GC_MALLOC_ATOMIC` で確保した領域を「ポインタを含まない」と
GC に教えておけば、GC はその中身を一切走査しません。すると、その領域に
たまたまヒープアドレスと一致する数値（画素値など）があっても、偽ポインタ
として誤認されることがなくなります。**`GC_MALLOC_ATOMIC` を正しく使うこと
は、偽ポインタによる無駄な保持を減らす最も効果的な手段の一つ** です。

> [!CAUTION]
> 逆に、ポインタを含む構造体を誤って `GC_MALLOC_ATOMIC` で確保すると、
> GC はその中のポインタをたどってくれません。すると参照先がまだ使われて
> いるのに回収され、ダングリングポインタになります。「ポインタを含むなら
> 必ず `GC_MALLOC`」を徹底してください。

## 明示的な GC 制御と統計

libgc はふだん自動でGCを走らせますが、手動で制御することもできます。
デバッグや性能測定で便利です。

```c
GC_gcollect();                 /* 今すぐフルGCを実行する */

size_t heap = GC_get_heap_size();      /* ヒープ全体のサイズ */
size_t free = GC_get_free_bytes();     /* そのうち空きバイト数 */
printf("heap=%zu free=%zu\n", (size_t)heap, (size_t)free);
```

`GC_FREE(p)` で明示的に解放することもできますが、保守的GCの利点を捨てる
ことになるので、通常は使いません。むしろ、確保はすべて GC に任せ、解放を
書かないのが libgc 流のスタイルです。

ここまでで、libgc を「使う」側の基本は身につきました。`malloc` を置き
換えるだけで GC が手に入る——この手軽さの裏で、libgc はいったいどんな
仕組みでポインタを探し、メモリを管理しているのでしょうか。次章
[](libgc-internals.md) で、その内部に踏み込みます。
