# はじめに

Julia は[`ccall`](https://docs.julialang.org/en/v1.7.0-rc1/base/c/#ccall)及び[`@ccall`](https://docs.julialang.org/en/v1.7.0-rc1/base/c/#Base.@ccall)で C の関数を呼び出すことが手軽にできます。

しかし、[公式ドキュメント](https://docs.julialang.org/en/v1.7.0-rc1/manual/calling-c-and-fortran-code/)の内容が誤解を招くっぽいので、適切なデータ型の選択に迷ったユーザーがかなりあります。

この記事には、初心者が陥りがちなミスの一つ、`Ref{T}`と`Ptr{T}`の使う場面を簡単に解説します。