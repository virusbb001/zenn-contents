---
title: "debug"
---

# debug

## 01-parse

`builder` の時と同様。

コミット: 5c2f2dd

## 02-impl-debug

`builder` の時と同様、パースと出力のために `syn`, `quote`, `proc-macro2` を追加する。
今回のテストでは `std::fmt::Debug` を `Field` に対して実装する。

始めに、テストコードに直接 `std::fmt::Debug` を実装する。 `builder 09` の内容を考慮し、マクロ内では常に絶対パスを指定する。
[`Debug` の docs.rs](https://doc.rust-lang.org/std/fmt/trait.Debug.html) を参考に仮で実装する。

```rust
impl std::fmt::Debug for Field {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::result::Result<(), std::fmt::Error> {
        std::result::Result::Ok(())
    }
}
```

テストコードのコメント内で紹介されている [`DebugStruct` の docs.rs](https://doc.rust-lang.org/std/fmt/struct.DebugStruct.html) を確認すると、これを使用することで構造体の表示が出来るようなので、これを使用する。

```rust
impl std::fmt::Debug for Field {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::result::Result<(), std::fmt::Error> {
        f.debug_struct("Field").field("name", &self.name).field("bitmask", &self.bitmask).finish()
    }
}
```
これをテストコード内に直接記載するとテストが通ったため、この方針で進める。

まず、テストコードで追加したコードを、マクロの `quote!` で出力するように移動する。

次に、構造体名に関わる箇所を置き換える。識別子のところは取得した内容をそのまま使用できるが、 `debug_struct` 内でそのまま渡すと `Field` 変数を渡すことになるため、文字列リテラルに変更する必要がある。文字列リテラルが `LitStr` であることは `builder` の `07-repeated-field` で分かっているため、これを使用する。が、改めて `ToTokens` を確認すると、Rustのプリミティブな値や `String` に対して一通り `ToTokens` が実装されているため、 `LitStr::new` を呼ばずに直接 `String` を `quote!` 内に埋め込むことにする。構造体のフィールドは `builder` のときは `Fields::Named` から取り出して使用していたが、 `Fields` には `iter()` があるため、今回は `Fields::iter` を使用する。

```rust
let input = parse_macro_input!(input as DeriveInput);
let ident = &input.ident;
let ident_litstr = ident.to_string();
let Data::Struct(input_struct) = input.data else {
    return syn::Error::new(input.span(), "CustomDebug is not used for struct").into_compile_error().into();
};

let field_call = input_struct.fields.iter().filter_map(|field| {
    let ident = field.ident.as_ref()?;
    let ident_str = ident.to_string();
    Some(quote! {
        .field(#ident_str, &self.#ident)
    })
});
quote! {
    impl std::fmt::Debug for #ident {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::result::Result<(), std::fmt::Error> {
            f.debug_struct(#ident_litstr)
                #(#field_call)*
                .finish()
        }
    }
}.into()
```

## 03-custom-format

このテストには `debug` 属性が指定されていれば、それをフォーマットとして表示する。 `eprintln!("{:#?}", field.attrs);` で結果を確認すると、 `meta` には `Meta::NameValue` が格納されている。以下のコードで、 `debug` に入っている値を取り出す。

```rust
fn get_debug_attr(attrs: &[Attribute]) -> Option<Result<String, syn::Error>> {
    attrs
        .iter()
        .filter_map(|attr| {
            match &attr.meta {
                Meta::NameValue(name_value) => Some(name_value),
                _ => None,
            }
        })
        .find(|name_value| {
            name_value.path.is_ident("debug")
        })
        .map(|name_value| {
            let Expr::Lit(ref lit) = name_value.value else {
                return Err(syn::Error::new(name_value.span(), "value of debug is not string"));
            };
            match &lit.lit {
                Lit::Str(lit_str) => Ok(lit_str.value()),
                _ => Err(syn::Error::new(lit.lit.span(), "value of debug is not string"))
            }
        })
}
```

テストコードのコメント内に、フォーマットを指定して表示するために `format_args!` が紹介されているためそれを使用する。
以下のようにしたところ、コンパイルエラーが発生した。

```rust
if let Some(debug_attr) = debug_attr {
    match debug_attr {
        Ok(debug) => {
            Some(quote! {
                .field(#ident_str, format_args!(#debug, &self.#ident))
            })
        },
        Err(err) => Some(err.into_compile_error()),
    }
} else {
    Some(quote! {
        .field(#ident_str, &self.#ident)
    })
}
```


```
error[E0308]: mismatched types
  --> tests/03-custom-format.rs:26:10
   |
26 | #[derive(CustomDebug)]
   |          ^^^^^^^^^^^
   |          |
   |          expected `&dyn Debug`, found `Arguments<'_>`
   |          arguments to this method are incorrect
   |          in this macro invocation
  --> $RUST/core/src/macros/mod.rs
   |
   = note: in this expansion of `format_args!`
   |
   = note: expected reference `&dyn Debug`
                 found struct `Arguments<'_>`
note: method defined here
  --> $RUST/core/src/fmt/builders.rs
```

`field` の 第2引数は `&dyn Debug` を要求しているが、 `format_args!` が返却するのは `std::fmt:Arguments` である。 `Arguments`は `Debug` を実装しているため、単に参照を渡すようにするだけでコンパイルを通すことができる。

コミット: 21d7af7

## 04-type-parameter

このテストでは型パラメータがある場合の実装を行なう。まず、完成形を考える。

```rust
impl<T: std::fmt::Debug> std::fmt::Debug for Field<T> {
    fn fmt(
        &self,
        f: &mut std::fmt::Formatter<'_>,
    ) -> std::result::Result<(), std::fmt::Error> {
        f.debug_struct("Field")
            .field("value", &self.value)
            .field("bitmask", &format_args!("0b{0:08b}", &self.bitmask))
            .finish()
    }
}
```

`impl` に型パラメータの指定が増えた。マクロ内でこれを表示する。型パラメータの情報は `input.generics` に格納されている。 `generics` は `Option` 型ではないため、型パラメータが設定されていない構造体であっても、共通して処理が書ける。 `input.generics` には `<T>` の情報がそのまま格納されているため、それをそのまま使う。 `<T: std::fmt::Debug>` の箇所は、最初の型パラメータを取り出して `<#param: std::fmt::Debug>` の形に変更する。

ここで、テストコードのコメント内で紹介されている、 `syn` の `examples` ディレクトリ内にある `heapsize` の実装を見てみると、 [型パラメータに制約を追加する実装](https://github.com/dtolnay/syn/blob/94e3d765d7c98d3f900c7d5088c1ecb0c8da3b21/examples/heapsize/heapsize_derive/src/lib.rs#L37)があったので、それを参考にする。

実装を確認すると、 `add_trait_bounds` で、型パラメータ全てに `heapsize::HeapSize` を追加する実装をしているため、今回の実装もこれに習う。

```rust
fn add_trait_bounds(mut generics: Generics) -> Generics {
    for param in &mut generics.params {
        if let GenericParam::Type(ref mut type_param) = *param {
            type_param.bounds.push(parse_quote!(std::fmt::Debug));
        }
    }
    generics
}
```

さらに、 `Generics` には `impl` のために `split_for_impl` というメソッドが用意されているため、そちらも使用する。

```rust
let generics = add_trait_bounds(input.generics);
let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();

let field_call = ...;
quote! {
    impl #impl_generics std::fmt::Debug for #ident #ty_generics #where_clause {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::result::Result<(), std::fmt::Error> {
            f.debug_struct(#ident_litstr)
                #(#field_call)*
                .finish()
        }
    }
}.into()
```

コミット: 611e33d

## 05-phantom-data

`T` が `PhantomData<U>` という型の場合について検討する。 `PhantomData<U>` は `U` が `Sized` の場合のみ `Debug` を実装している。そのため、以下のように実装する必要がある。 `PhantomData` に関しては、 [Rust By Example 幽霊型パラメータ](https://doc.rust-jp.rs/rust-by-example-ja/generics/phantom.html)など他を参照する。

```rust
impl<T> Debug for Field<T>
where
    PhantomData<T>: Debug,
{...}
```

追加する形だと [E0119](https://doc.rust-lang.org/error_codes/E0119.html) が発生するので、 `PhantomData` が構造体のフィールドにあった場合あった場合は上記のコードを出力するように変更する。

少々雑だが、1つでも `PhantomData` のフィールドがあれば、`impl<T>` の型パラメータ内には `std::fmt::Debug` をつけないことにする。フィールドの型から、 `PhantomData<T>` の箇所を抜き出す処理を実装する。 中の `U` でなく、 `PhantomData<U>` なのは、 `where` 内での表示に使用するためである。

```rust
fn get_type_phantom_data_in_fields(fields: &Fields) -> Vec<&Type> {
    fields
        .iter()
        .filter_map(|field| match field.ty {
            Type::Path(ref path)
                if path
                    .path
                    .segments
                    .last()
                    .is_some_and(|segment| segment.ident == "PhantomData") =>
            {
                Some(&field.ty)
            }
            _ => None,
        })
        .collect()
}
```

`derive` 関数に以下のように処理を加える。

```rust
let phantom_type_fields = get_type_phantom_data_in_fields(&input_struct.fields);
let bound = phantom_type_fields
    .iter()
    .map(|ty| {
        quote! { #ty: std::fmt::Debug }
    })
    .collect::<Vec<_>>();

let generics = if bound.is_empty() { add_trait_bounds(input.generics) } else { input.generics };
...
let impl_clause = if bound.is_empty() {
    quote! { #impl_generics std::fmt::Debug for #ident #ty_generics #where_clause }
} else {
    quote! { #impl_generics std::fmt::Debug for #ident #ty_generics where #(#bound),* }
};
```

コミット: 5474d74

## 06-bound-trouble

このテストはこれまでの実装が済んでいれば通過する。コメントに一通り目を通しておく。

コミット: 5340c6b

## 07-associated-type

関連型について考慮する。最終形はテストコメントに記載されている。実装方針として、各フィールドの型もしくはその中の型パラメータが`Type::Path` である、かつ最初のパスが型パラメータにあるものを取得し、それに対して `: Debug` を付ける。
`T::Value` が来た場合そのまま返せば良いが、 `Vec<T::Value>` が来た場合は、型パラメータの中を見て `T::Value` を取得し、それに `: Debug` をつけないといけない。かつ、 `PhantomData` だったら除外する必要がある。以下のように取得する。

```rust
fn get_types_to_bind_debug<'a>(fields: &'a Fields, generics: &Generics) -> Vec<&'a TypePath> {
    let generics = generics.params.iter().filter_map(|param| {
        match param {
            GenericParam::Type(ty) => Some(&ty.ident),
            _ => None,
        }
    }).collect::<Vec<_>>();
    fields
        .iter()
        .filter_map(|field| {
            let Type::Path(ref ty) = field.ty else {
                return None;
            };
            Some(ty)
        })
        .filter_map(|ty| {
            let is_associated = ty.path.segments.first().is_some_and(|ty| {
                generics.contains(&&ty.ident)
            });
            if is_associated {
                Some(ty)
            } else {
                if ty.path.segments.first().is_some_and(|segment| {
                    segment.ident == "PhantomData"
                }) {
                    return None;
                }
                let Some(PathArguments::AngleBracketed(args)) = ty.path.segments.last().map(|segment| &segment.arguments) else {
                    return None;
                };
                args.args.iter().filter_map(|arg| {
                    if let GenericArgument::Type(ty) = arg {
                        Some(ty)
                    } else {
                        None
                    }
                }).filter_map(|ty| {
                    if let Type::Path(path) = ty {
                        Some(path)
                    } else {
                        None
                    }
                }).find(|ty| {
                        ty.path.segments.first().is_some_and(|ty| {
                            generics.contains(&&ty.ident)
                        })
                })
            }
        })
        .collect::<Vec<_>>()
}
```

上記で得たものを 05-phantom-data で作成したものと結合すればテストは通る。

コミット: 296759c

## 08-escape-hatch

これまでフィールドの型から情報を取得し、bindを設定していたが、このテストでは手動で設定できるように変更し、もし設定されていれば自動での設定は行わないようにする。テストコメント中ではフィールドごとの設定もあるが今回は構造体全体のみを行なう。

`bound` の値を取り出す関数を作成する。

```rust
fn get_bound_in_debug_attr(attrs: &[Attribute]) -> Option<Result<String, syn::Error>> {
    attrs
        .iter()
        .filter_map(|attr| match &attr.meta {
            Meta::List(attr_list) => Some(attr_list),
            _ => None,
        })
        .find(|list| list.path.is_ident("debug"))
        .and_then(|list| list.parse_args::<MetaNameValue>().ok())
        .filter(|name_value| name_value.path.is_ident("bound"))
        .map(|name_value| match name_value.value {
            Expr::Lit(ExprLit{lit: Lit::Str(litstr), ..}) => {
                Ok(litstr.value())
            }
            _ => Err(syn::Error::new(
                    name_value.span(),
                    "value of debug is not string",
                ))
        })
}
```

`String` をそのまま `quote` の中に埋め込むと文字列リテラルとなるので、パースする必要がある。 `syn` クレートは、文字列からパースする [`parse_str`](https://docs.rs/syn/2.0.49/syn/fn.parse_str.html) があるので、それを使用する。これまでに作成した `bound` の型は `Vec<TokenStream>` なので、それになるように加工する。

```rust
let struct_bound = get_bound_in_debug_attr(&input.attrs).map(|bound| {
    let token_stream = bound.and_then(|bound| {
        syn::parse_str::<WherePredicate>(&bound).map(|predicate| predicate.to_token_stream())
    });

    match token_stream {
        Ok(token_stream) => token_stream,
        Err(err) => err.into_compile_error(),
    }
}).into_iter().collect::<Vec<_>>();
```

`struct_bound` に要素があれば、 `bound` の値は使用しないように変更する。

コミット: 3ad9a4d
