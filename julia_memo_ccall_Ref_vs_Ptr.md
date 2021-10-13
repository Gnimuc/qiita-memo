# JuliaからCを呼び出しの注意点: `Ref{T}`と`Ptr{T}`の使い
# はじめに

[![hackmd-github-sync-badge](https://hackmd.io/8G-lo9oKTTyiCQdHuyXk9w/badge)](https://hackmd.io/8G-lo9oKTTyiCQdHuyXk9w)


Juliaは[`ccall`](https://docs.julialang.org/en/v1.6/base/c/#ccall)および[`@ccall`](https://docs.julialang.org/en/v1.6/base/c/#Base.@ccall)でCの関数を呼び出すことが手軽にできます。

しかし、[公式ドキュメント](https://docs.julialang.org/en/v1.6/manual/calling-c-and-fortran-code/)の内容が誤解を招くっぽいので、適切な戻り値の型と引数の型の選択に迷ったユーザーがかなりあります。

この記事には、初心者が陥りがちなミスの一つ、`Ref{T}`と`Ptr{T}`の使う場面を簡単に解説します。

# 要点

__ポインタ引数および戻り値のデータ型を指定する時、いつも`Ptr{T}`を使っても構いません。__

例えば、Julia側の引数の型にかかわらず、下記のように書けます。

```julia:Julia
ccall(
    (:foo, "libfoo"),        # 関数名, ライブラリ名
    Ptr{Cuchar}, 　　　　     # `foo`の戻り値の型
    (Cint, Ptr{Cfloat}),     # `foo`の引数1の型, 引数2の型
    x,                       # Julia側の引数x
    y,                       # Julia側の引数y
)
```

引数yと引数2の型が一致しない場合には、[`Base.cconvert`](https://docs.julialang.org/en/v1.6/base/c/#Base.cconvert)と[`Base.unsafe_convert`](https://docs.julialang.org/en/v1.6/base/c/#Base.unsafe_convert)による, 引数yの型が引数2の型と暗黙に変換されます。`Base.cconvert`と`Base.unsafe_convert`関数呼び出しの順番は下記のようになります。

```julia:Julia
y1 = Base.cconvert(Ptr{Cfloat}, y)
y2 = Base.unsafe_convert(Ptr{Cfloat}, y1)
```