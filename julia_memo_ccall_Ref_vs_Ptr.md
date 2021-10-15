# JuliaからCを呼び出しの注意点: `Ref{T}`と`Ptr{T}`の使いわけ

[![hackmd-github-sync-badge](https://hackmd.io/8G-lo9oKTTyiCQdHuyXk9w/badge)](https://hackmd.io/8G-lo9oKTTyiCQdHuyXk9w)

# はじめに

Juliaは[`ccall`](https://docs.julialang.org/en/v1.6/base/c/#ccall)あるいは[`@ccall`](https://docs.julialang.org/en/v1.6/base/c/#Base.@ccall)でCの関数を呼び出すことが手軽にできます。

しかし、[公式ドキュメント](https://docs.julialang.org/en/v1.6/manual/calling-c-and-fortran-code/)の内容が誤解を招くっぽいので、適切な戻り値の型と引数の型の選択に迷ったユーザーがかなりあります。

この記事には、ポインタ型の選択ルールを簡単に解説します。

# 要点

__`ccall`のポインタ引数および戻り値のデータ型を指定する時、いつも`Ptr{T}`を使っても構いません。__

例えば、Julia側の引数の型にかかわらず、下記のように書けます。

```julia:Julia
# C関数`foo`のシグネチャ:
# const unsigned char * foo(int x, float * y);
ccall(
    (:foo, "libfoo"),        # 関数名, ライブラリ名
    Ptr{Cuchar}, 　　　　     # `foo`の戻り値の型
    (Cint, Ptr{Cfloat}),     # `foo`の引数1の型, 引数2の型
    x,                       # Julia側の引数x
    y,                       # Julia側の引数y
)
```

実際は、`ccall`は普通の関数じゃなくて[^1]、C関数にJulia側の引数を直接に渡すわけじゃないです。

引数yと引数2の型が一致しない場合には、[`Base.cconvert`](https://docs.julialang.org/en/v1.6/base/c/#Base.cconvert)と[`Base.unsafe_convert`](https://docs.julialang.org/en/v1.6/base/c/#Base.unsafe_convert)による, 引数yの型が引数2の型と暗黙に変換されます。`Base.cconvert`と`Base.unsafe_convert`呼び出しの順番は下記のようになります[^2]。

```julia:Julia
Base.unsafe_convert(Ptr{Cfloat}, Base.cconvert(Ptr{Cfloat}, 引数))
```

Juliaの`Base`モジュールの中には、ここでの`unsafe_convert`関数と`cconvert`関数を事前に定義してありますので、引数の型を自動的に変換することができます。しかも、__適切なメソッドが見つからない場合、オーバーロードする必要があります。__


```julia:Julia
julia> methods(Base.cconvert)
# 21 methods for generic function "cconvert":
[1] cconvert(::Type{T}, x::Enum{T2}) where {T<:Integer, T2<:Integer} in Base.Enums at Enums.jl:20
[2] cconvert(::Type{Ref{T}}, t::Tuple{Vararg{T, N}}) where {N, T} in Base at refpointer.jl:170
...

julia> methods(Base.unsafe_convert)
# 69 methods for generic function "unsafe_convert":
[1] unsafe_convert(T::Union{Type{Ptr{Nothing}}, Type{Ptr{Base.Libc.FILE}}}, f::Base.Libc.FILE) in Base.Libc at libc.jl:94
[2] unsafe_convert(::Type{Ref{BigFloat}}, x::Ptr{BigFloat}) in Base.MPFR at mpfr.jl:130
[3] unsafe_convert(::Type{Ref{BigFloat}}, x::Ref{BigFloat}) in Base.MPFR at mpfr.jl:131
...
```

## `Ref{T}`を使うべき主な場面

`Ref{T}`[^3]は、実際に`RefValue{T}`[^4]として使う場面が多いです。

例えば、前文のJulia側の引数yは、下記のように定義します。

```julia:Julia
julia> y = Ref{Cfloat}(0.0f0)
Base.RefValue{Float32}(0.0f0)
```

なぜこれが必要なのかというと、Juliaの`Int`, `Float32`, `Ptr{Float32}`といった__[`isbitstype`](https://docs.julialang.org/en/v1/base/base/#Base.isbitstype)型は、固定のアドレスがないからです。__

C関数がこのJuliaから受け取ったアドレスに何を記入するかもしれないので、固定のアドレスを使わなければならないでしょう。

```julia:Julia
julia> pointer_from_objref(0.0f0)
ERROR: pointer_from_objref cannot be used on immutable objects
Stacktrace:
 [1] error(s::String)
   @ Base ./error.jl:33
 [2] pointer_from_objref(x::Any)
   @ Base ./pointer.jl:146
 [3] top-level scope
   @ REPL[13]:1

julia> pointer_from_objref(y)
Ptr{Nothing} @0x000000010f65f8b0
```

# まとめ

* `ccall`/`@ccall`に関わるポイント型を選択する必要がない。常に`Ptr{T}`が使えます。

* デフォルトメソッドで、`Ptr{T}`に変換エラーが発生する時、`Base.unsafe_convert`をオーバーロードする必要があります。

* Cの関数を呼び出すために、Julia側のポインタ型変数を`Ref{T}`で定義するのがほとんどです。

# クイズ

下記のコードの中に、間違えたところがありますか。なぜですか。

```julia:Julia
y = Ref{Cfloat}(0.0f0)
y1 = Base.cconvert(Ptr{Cfloat}, y)
y2 = Base.unsafe_convert(Ptr{Cfloat}, y1)
ccall((:foo, "libfoo"), Ptr{Cuchar}, (Cint, Ptr{Cfloat}), 0, y2)
y[]
```

[^1]: https://github.com/JuliaLang/julia/blob/master/src/ccall.cpp 
[^2]: https://github.com/JuliaLang/julia/blob/master/base/c.jl
[^3]: https://github.com/JuliaLang/julia/blob/master/base/refpointer.jl
[^4]: https://github.com/JuliaLang/julia/blob/master/base/refvalue.jl