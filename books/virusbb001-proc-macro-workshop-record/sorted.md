---
title: sorted
---

# sorted

## 01-parse-enum

`#[sorted]` で処理出来るようにテストを通す。 次に移る前に `input` を `syn::Item` で
パースしておくことが推奨されているため、 `syn::Item` でパースし、その結果を `TokenStream`
に変換して返却するようにする。

`cargo add -F full syn` で `syn::Item` を使用できるようにする。 `TokenStream` への変換のため
`quote` も追加する。

```rust
let item = parse_macro_input!(input as syn::Item);
item.to_token_stream().into()
```

コミット: dd6b1a3

## 02-not-enum

`enum` でなければエラーを出力するようにする。 `input` が `syn::Item::Enum` であればそのまま返却し、そうでなければコンパイルエラーを出力させる。 期待されるエラーの情報は例によって `.stderr` にあるのでそこから参照する。 `Span` を使用するので `proc-macro2` も追加する。

```rust
if matches!(item, syn::Item::Enum(_)) {
    item.to_token_stream().into()
} else {
    syn::Error::new(Span::call_site(), "expected enum or match expression").to_compile_error().into()
}
```

コミット: d8b2cdb

## 03-out-of-order

ソートされていないキーを見付けたら、それをエラーとして出力する。欲しい情報はそれぞれの対象に対して、対象の `Ident` 、本来後に来るべき `Ident`、 そしてエラー箇所指定のための対象の `Span`。

これらを保管する `struct` を作成する。

```rust
#[derive(Debug)]
struct WrongLocations {
    target: String,
    expected: String,
    span: Span,
}
```

`enum` のバリアント情報を抜き出し、 `Ident` でソートし、順番が変わっていないか、本来より先に来るべきものが後に来ていないかを探索して `WrongLocations` に格納する。得られた `WrongLocations` から `syn::Error` を作成し、得た `syn::Error` すべてを `TokenStream2` に変換して結合する。こうして得た `TokenStream2` が空ならばマクロ処理で受け取ったものをそのまま返し、空でなければエラーから得たものを返却する。

```rust
let mut sorted_variants = item_enum
    .variants
    .iter()
    .map(|variant| &variant.ident)
    .enumerate()
    .collect::<Vec<_>>();
sorted_variants.sort_by_key(|v| v.1);
let wrong_positions = sorted_variants
    .iter()
    .enumerate()
    .filter_map(|(i, item)| {
        sorted_variants
            .iter()
            .skip(i + 1)
            .find(|v| v.0 < item.0)
            .map(|other| WrongLocations {
                target: item.1.to_string(),
                expected: other.1.to_string(),
                span: item.1.span(),
            })
    })
    .map(|wrong| {
        syn::Error::new(
            wrong.span,
            format!("{} should sort before {}", wrong.target, wrong.expected),
        )
        .to_compile_error()
    })
    .collect::<TokenStream2>();
if wrong_positions.is_empty() {
    item_enum.to_token_stream().into()
} else {
    wrong_positions.into()
}
```

コミット: 3d39287

## 04-variants-with-data

データを持つ要素に対して実装を行なう。このままの実装では以下のような警告が出てしまい、テストに通過しない。

```
warning: unused import: `std::env::VarError`
 --> tests/04-variants-with-data.rs:7:5
  |
7 | use std::env::VarError;
  |     ^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default
...
```

これは先程の実装でエラーがあれば元の `enum` を出力しなくなってしまったためである。 `wrong_positions` を可変にして、 `item_enum` と結合するように変更することで、常に `enum` を出力するようになる。

コミット: 9c7ed7b

## 05-match-expr

`match` 式にも `sorted` を使用できるようにする。テストのコメントに書かれている手順で実装する。

`sorted::check` を追加する。そのなかで `input` を `syn::ItemFn` でパースする。

```rust
#[proc_macro_attribute]
pub fn check(args: TokenStream, input: TokenStream) -> TokenStream {
    let item = parse_macro_input!(input as syn::ItemFn);
    item.to_token_stream().into()
}
```

`visit_expr_match_mut` のみを実装した、 `VisitMut` を実装する。 [`syn::visit_mut`](https://docs.rs/syn/latest/syn/visit_mut/index.html) の例の項目が参考になる。 `cargo add -F visit-mut syn` を実行して `VisitMut` を使えるようにする。 エラーを後で出力できるようにするために、 `Vec<WrongLocations>` を保持する。保持されたエラーは `TokenStream2` に変換され、出力する。
`impl From<&WrongLocations> for syn::Error` も実装する。

```rust
struct SortedInFn(Vec<WrongLocations>);

impl VisitMut for SortedInFn {
    fn visit_expr_match_mut (&mut self, expr_match: &mut ExprMatch) {
        eprintln!("{:#?}", expr_match);
        visit_mut::visit_expr_match_mut(self, expr_match);
    }
}

#[proc_macro_attribute]
pub fn check(_: TokenStream, input: TokenStream) -> TokenStream {
    let mut item = parse_macro_input!(input as syn::ItemFn);
    let mut sorted_in_fn = SortedInFn(Vec::new());
    sorted_in_fn.visit_item_fn_mut(&mut item);
    let mut errors = sorted_in_fn.0
        .iter()
        .map(|wrong| syn::Error::from(wrong).to_compile_error())
        .collect::<TokenStream2>();
    errors.extend(item.to_token_stream());
    errors.into()
}

impl From<&WrongLocations> for syn::Error {
    fn from (wrong: &WrongLocations) -> Self{
        syn::Error::new(
            wrong.span,
            format!("{} should sort before {}", wrong.target, wrong.expected),
        )
    }
}
```

`eprintln` で `expr_match` の中を確認して、問題なさそうなことを確認する。
まず、 `#[sorted]` があれば、あったことを出力して、その要素を削除する。

```rust
impl VisitMut for SortedInFn {
    fn visit_expr_match_mut (&mut self, expr_match: &mut ExprMatch) {
        let sorted_position = expr_match.attrs.iter().position(|v| {
            v.meta.path().is_ident("sorted")
        });
        if let Some(sorted_index) = sorted_position {
            expr_match.attrs.remove(sorted_index);
        }
        visit_mut::visit_expr_match_mut(self, expr_match);
    }
}
```

統一して処理できるように、ソート部分を別関数に切り出す。

```rust
fn get_unsorted_items(idents: &[&Ident]) -> Vec<WrongLocations> {
    let mut idents = idents.iter().enumerate().collect::<Vec<_>>();
    idents.sort_by_key(|v| v.1);

    idents
        .iter()
        .enumerate()
        .filter_map(|(i, item)| {
            idents
                .iter()
                .skip(i + 1)
                .find(|v| v.0 < item.0)
                .map(|other| WrongLocations {
                    target: item.1.to_string(),
                    expected: other.1.to_string(),
                    span: item.1.span(),
                })
        })
        .collect::<Vec<_>>()
}
```

`sorted` の中は以下のように変わる。

```rust
let idents = item_enum
    .variants
    .iter()
    .map(|variant| &variant.ident)
    .collect::<Vec<_>>();
let mut wrong_positions = get_unsorted_items(&idents).iter()
    .map(|wrong| {
        syn::Error::new(
            wrong.span,
            format!("{} should sort before {}", wrong.target, wrong.expected),
        )
        .to_compile_error()
    })
    .collect::<TokenStream2>();
```

`ExprMatch` からアームのパターンを抜き出して `Fmt`, `Io` の箇所だけ抜き出し、 `Vec<&Ident>` を作る。作った `Vec` は先程作成した関数に渡し、結果を `self.0` に追加する。

```rust
let idents = expr_match.arms.iter().filter_map(|arm| {
    if let Pat::TupleStruct(tuple_struct) = &arm.pat {
        tuple_struct.path.segments.last().map(|segment| &segment.ident)
    } else {
        None
    }
}).collect::<Vec<_>>();
self.0.extend(get_unsorted_items(&idents));
```

コミット: c4071e0

## 06-pattern-path

従来の実装では、 `match` 内の エラー表示が各アームの `Path` の最後の要素のみだったが、全体でエラー表示されるように変更する。
`fn get_unsorted_items(idents: &[(String, Span)]) -> Vec<WrongLocations>` となるようにリファクタリングする。

`Pat::TupleStruct` 全体の名前を取得し、 `::` で結合する。結合したもので比較を行なうようにする。

```rust
let idents = expr_match.arms.iter().filter_map(|arm| {
    if let Pat::TupleStruct(tuple_struct) = &arm.pat {
        let path = &tuple_struct.path;
        let span = path.span();
        let str = path.segments.iter().map(|s| s.ident.to_string()).collect::<Vec<_>>().join("::");
        Some((str, span))
    } else {
        None
    }
}).collect::<Vec<_>>();
```

コミット: 355fa14

## 07-unrecognized-pattern

`path` が取れないなど比較できない場合コンパイルエラーを出力するようにする。 `SortedInFn` で保持する情報を `Vec<syn::Error>` に置き換える。

期待されるエラー出力を見ると、最初の1つだけをエラー箇所として出力しているので、 `Result<Path, syn::Error>` を作り、さらに `Ok(Path)` から `Vec<syn::Error>` を作成する。

```rust
let paths = expr_match.arms.iter().map(|arm| {
    if let Pat::TupleStruct(tuple_struct) = &arm.pat {
        Ok(&tuple_struct.path)
    } else {
        Err(syn::Error::new(arm.pat.span(), "unsupported by #[sorted]"))
    }
}).collect::<Result<Vec<_>, _>>();
let errors = match paths {
    Ok(paths) => {
        let idents = paths.iter().map(|path| {
            let span = path.span();
            let str = path.segments.iter().map(|s| s.ident.to_string()).collect::<Vec<_>>().join("::");
            (str, span)
        }).collect::<Vec<_>>();
        get_unsorted_items(&idents).iter().map(|wrong| wrong.into()).collect::<Vec<_>>()
    },
    Err(err) => vec![err],
};
```

コミット: ffc8625

## 08-underscore

ワイルドカードが最後に来るように検査する。
始めに、 `Pat::Ident` をサポートする。

```rust
fn path_to_str(path: &syn::Path) -> (String, Span) {
    let span = path.span();
    let str = path.segments.iter().map(|s| s.ident.to_string()).collect::<Vec<_>>().join("::");
    (str, span)
}

let ident_spans = expr_match.arms.iter().map(|arm| {
    match &arm.pat {
        Pat::TupleStruct(tuple_struct) => Ok(path_to_str(&tuple_struct.path)),
        Pat::Ident(pat_ident) => Ok((pat_ident.ident.to_string(), pat_ident.ident.span())),
        _ => Err(syn::Error::new(arm.pat.span(), "unsupported by #[sorted]")),
    }
}).collect::<Result<Vec<_>, _>>();
```

`PatWild` をサポートする。 `_` として比較すると最後に来るので、 `"_"` として処理する。

```rust
Pat::Wild(pat_wild) => Ok(("_".to_string(), pat_wild.span())),
```

コミット: 64f3967
