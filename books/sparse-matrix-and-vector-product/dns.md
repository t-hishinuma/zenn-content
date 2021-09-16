---
title: 密行列の格納形式：DNS
---
# 概要
DNS形式などと呼ばれることもありますが，これは`Dense`，つまり密行列のことです．一覧にしたとき，密行列にあえて名前をつけたい場合に`DNS`と書くことがあります．

つまりこの章では密行列とベクトルの積，いわゆるGEMV (GEneral Matrix and Vector Product)について解説します．
当然ながらDNSは特殊なメモリレイアウトではありません．`A[N][N]`とか`A[N*N]`とか書いて連続なメモリ空間に完全な状態で行列をもちます．

「おいおい疎行列じゃないのかよ！」．と思うかもしれませんが，DNSを疎行列の格納形式から省いて良いかどうかは微妙な問題です．
2章で述べたように疎行列の厳密な定義はないので，非零要素の密度 (密度ならDensity, 疎率ならSparsityなどと呼ばれる)がどの程度かは分かりません．

2章で紹介したようなCOO形式は，1つの非零要素に対して行番号と列番号が必要になります．非零要素が64 bit float, 行番号と列番号が32 bit integerと考えれば，64 bitの要素を保持するのに128 bit (2倍)のメモリデータが必要ということになります．
つまり，全体の50%以上を非零要素が占めるような行列の場合，COOとDNSのメモリデータ量は同じになるので，計算の単純なDNSでも構わないことになります．

このように，非零要素の数によってDNSという選択肢は常に考慮されるべきであることから，本書でははじめにDNSについて取り扱います．

## DNS formatの構成
前述の通り，`A[N][N]`とか`A[N*N]`とか書いて連続なメモリ空間に完全な状態で行列をもちます．

行方向に格納するか，列方向に格納するかは任意ですが，ここでは行方向に1次元配列で`A[N*N]`と格納したとします．

データ量は$N^2$になります．

# DNS形式の行列の取り扱い
## DNS形式の行列ベクトル積 (逐次)
行数$N$，列数$N$の`A[N*N]`に対する愚直なGEMVです．

```cpp
for(int i=0; i<N; i++){
    y[i] = 0.0;
    for(int j=0; j<N; j++){
     y[i] += A[i*N+j] * x[j];
    }
}
```

計算量としては$2N^2$になります(最内側の計算量は足し算と掛け算があるため2で，それをN*N回行う)．

## DNS形式の疎行列ベクトル積 (スレッド並列)
Denseであれば，行単位で並列化すれば良いので，これもさほど難しくないです (Cの人はtmpとjをprivateにするのが必要ですが，今回はローカル変数にしてるので必要はない)．

```cpp
#pragma omp parallel for
for(int i=0; i<N; i++){
    y[i] = 0.0;
    for(int j=0; j<N; j++){
     y[i] += A[i*N+j] * x[j];
    }
}
```

これはスレッドごとの計算量は行数によって決まり，非常に簡単なスレッド並列モデルかと思います．
ブロック化などすることで更にスレッドごとの計算量を均等にするなどの工夫はありますが，十分に行数が大きければ問題になることは少ないため，このままのコードでもそれなりの性能は期待できると思います．

# 余談，密行列ベクトル積の高速なライブラリについて
BLASの話を書きます．未執筆