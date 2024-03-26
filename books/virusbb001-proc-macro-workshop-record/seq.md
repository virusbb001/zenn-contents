---
title: seq
---

# seq

## 01-parse-header

`builder` の時と同様。

コミット: f905ae3

## 02-parse-body

このテストでは特に追加実装せずに次に進むことができる。事前に `input` の `TokenStream` の値を確認しておく。

```
TokenStream [
    Ident {
        ident: "N",
        span: #0 bytes(831..832),
    },
    Ident {
        ident: "in",
        span: #0 bytes(833..835),
    },
    Literal {
        kind: Integer,
        symbol: "0",
        suffix: None,
        span: #0 bytes(836..837),
    },
    Punct {
        ch: '.',
        spacing: Joint,
        span: #0 bytes(837..838),
    },
    Punct {
        ch: '.',
        spacing: Alone,
        span: #0 bytes(838..839),
    },
    Literal {
        kind: Integer,
        symbol: "4",
        suffix: None,
        span: #0 bytes(839..840),
    },
    Group {
        delimiter: Brace,
        stream: TokenStream [
            Ident {
                ident: "expand_to_nothing",
                span: #0 bytes(847..864),
            },
            Punct {
                ch: '!',
                spacing: Alone,
                span: #0 bytes(864..865),
            },
            Group {
                delimiter: Parenthesis,
                stream: TokenStream [
                    Ident {
                        ident: "N",
                        span: #0 bytes(866..867),
                    },
                ],
                span: #0 bytes(865..868),
            },
            Punct {
                ch: ';',
                spacing: Alone,
                span: #0 bytes(868..869),
            },
        ],
        span: #0 bytes(841..871),
    },
]
```
コミット: 76e961a

## tests/03-expand-four-errors.rs

実際に来た `TokenStream` をパースし、出力する。まず、パースする処理を書く。 [lazy_static](https://github.com/dtolnay/syn/blob/bd931069f5adb81168e7e6186ead0663cc605257/examples/lazy-static/lazy-static/src/lib.rs#L28-L46) にパース処理があるので、それを参考に実装する。

`parse` から受け取る型は `01-parse-header` のコメント内に (`Expr` 以外は) 記載されている。 `Expr` に関しては `lazy_static` の実装で確認できる。

```rust
#[derive(Debug)]
struct Seq {
    ident_replace: Ident,
    start: LitInt,
    end: LitInt,
    expr: Expr,
}

impl Parse for Seq {
    fn parse(input: syn::parse::ParseStream) -> syn::Result<Self> {
        let ident_replace = input.parse::<Ident>()?;
        input.parse::<Token![in]>()?;
        let start = input.parse::<LitInt>()?;
        input.parse::<Token![..]>()?;
        let end = input.parse::<LitInt>()?;
        let expr = input.parse::<Expr>()?;

        Ok(Seq {
            ident_replace,
            start,
            end,
            expr
        })
    }
}
```

`Expr` は `{}` を含んでいる。`seq!` で展開するときには `{}` が無いほうが望ましいので、 `Expr` から `{}` を取り除いた形が欲しい.
ここで、 `syn::parse` の [docs.rs](https://docs.rs/syn/2.0.52/syn/parse/index.html) を確認する。例は `struct` か `enum` をパースするもののようだが、ここで `ItemStruct::parse` の実装を見ると、 `fields` が `content` から生成されており、その `content` が `braced!` で何かしていることがわかる。 `braced!(content in input)` を実行すると `{...}` を読み込んで `content` の中に入れるらしい。 `content` の型は [`ParseBuffer`](https://docs.rs/syn/2.0.52/syn/parse/struct.ParseBuffer.html) であるが、これも中を `parse` でパースできるようである。今回の場合、特定の識別子だけを置換したいことと、READMEの coverに "low-level representation of token streams" とあるので、なるべく生の `TokenStream` を扱いたい。幸い、 `proc_macro2::TokenStream` が `Parse` を実装しているので、 `TokenStream` を格納するようにする。 `proc_macro::TokenStream` と名前が重複するので、以下 `proc_macro2::TokenStream as TokenStream2` とする。

`expr` を `tokens` に変更し、 `TokenStream2` を格納する。以下のように変更する。

```rust
#[derive(Debug)]
struct Seq {
    ...
    tokens: TokenStream2
}

impl Parse for Seq {
    fn parse(input: syn::parse::ParseStream) -> syn::Result<Self> {
        ...
        let content;
        braced!(content in input);
        let tokens = content.parse::<TokenStream2>()?;

        Ok(Seq {
            ident_replace,
            start,
            end,
            tokens,
        })
    }
}
```

ちなみに、以下のようなコードで `cargo expand` をすると、以下のようなエラーが出力される。

```rust
pub fn seq(input: TokenStream) -> TokenStream {
    quote! {
        {
            compile_error!(concat!("error number ", stringify!(0)));
        }
    }.into()
}
```

```
error: macro expansion ignores token `{` and any following
  --> main.rs:11:1
   |
11 | / seq!(N in 0..4 {
12 | |     compile_error!(concat!("error number ", stringify!(N)));
13 | | });
   | |__^ caused by the macro expansion here
   |
   = note: the usage of `seq!` is likely invalid in item context

```

試しに、 `tokens` をそのまま出力する。

```rust
#[proc_macro]
pub fn seq(input: TokenStream) -> TokenStream {
    let Seq {
        ident_replace,
        start,
        end,
        tokens,
    } = parse_macro_input!(input as Seq);

    quote! {
        #tokens
    }.into()
}
```

この状態で `cargo expand` を実行すると、以下のエラーが出力されたので、 `{...}` の中の取り出しはこれで問題なさそうだ。

```
error: error number N
  --> main.rs:12:5
   |
12 |     compile_error!(concat!("error number ", stringify!(N)));
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

ここから、 `tokens` の `ident_replace` に当てはまる箇所を `0` に置き換える。 `tokens` を `eprintln!("{:#?}",tokens)` で確認すると、`[]` で囲まれており、イテレータでアクセスして処理できそうな表記に見える。 docs.rs を確認すると `IntoIterator` を実装しているので、これを利用する。このとき要素は `TokenTree` である。 `tokens` の中を確認すると、 `Group` が更に入れ子になっているため、再帰処理が必要になる。以下のようにして置き変える。 `usize` は仮の型。

```rust
fn replace_tokens(
    token_tree: TokenTree,
    ident_replace: &Ident,
    n: usize,
) -> TokenTree {
    match token_tree {
        TokenTree::Group(group) => {
            let delimiter = group.delimiter();
            let stream = group.stream().into_iter().map(|token_tree| replace_tokens(token_tree, ident_replace, n)).collect::<TokenStream2>();
            TokenTree::Group(Group::new(delimiter, stream))
        },
        TokenTree::Ident(ident) => {
            if ident == *ident_replace {
                TokenTree::Literal(Literal::usize_unsuffixed(n))
            } else {
                TokenTree::Ident(ident)
            }
        },
        TokenTree::Punct(_) => token_tree,
        TokenTree::Literal(_) => token_tree,
    }
}
```

以下のようにして、 `N` を `0` 限定にして動作確認をすると、うまくいった。

```rust
let tokens = tokens.clone().into_iter().map(|tokentree| replace_tokens(tokentree, &ident_replace, 0)).collect::<TokenStream2>();
```

`start` と `end` をパースして `Range` に変換し、それぞれの値に対して `tokens` を変更する処理を生成する。 `LitInt` は `base10_parse` メソッドを実装しており、 `Return` の `Err` には `syn::Error` が格納されている。そのため、パースしたらコンパイルエラーとして返すことにする。

```rust
let start = start.base10_parse::<usize>();
let end = end.base10_parse::<usize>();
let (Ok(start), Ok(end)) = (start.clone(), end.clone()) else {
    return [start, end]
        .into_iter()
        .filter_map(|v| v.err())
        .map(|v| v.into_compile_error())
        .collect::<TokenStream2>()
        .into();
};
```

`tokens` 変換処理は以下のようになる。

```rust
let tokens = (start..end).map(|n| {
    tokens.clone().into_iter().map(|tokentree| replace_tokens(tokentree, &ident_replace, n)).collect::<TokenStream2>()
}).collect::<TokenStream2>();
```

`error: error number N` が数値に置き換わり、 `N` が 0, 1, 2, 3となっていれば成功。
`cargo test` を実行してみる。


```
EXPECTED:
┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
error: error number 0
  --> tests/03-expand-four-errors.rs:20:5
   |
20 |     compile_error!(concat!("error number ", stringify!(N)));
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
...
┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈

ACTUAL OUTPUT:
┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
error: error number 0
  --> tests/03-expand-four-errors.rs:19:1
   |
19 | / seq!(N in 0..4 {
20 | |     compile_error!(concat!("error number ", stringify!(N)));
21 | | });
   | |__^
   |
   = note: this error originates in the macro `seq` (in Nightly builds, run with -Z macro-backtrace for more info)
...
```

エラーが出た。エラーとして表示されるべき範囲が異なるようだ。 `Group` に元々の値の `Span` を設定するように変更する。

```rust
...
TokenTree::Group(group) => {
    let delimiter = group.delimiter();
    let stream = group.stream().into_iter().map(|token_tree| replace_tokens(token_tree, ident_replace, n)).collect::<TokenStream2>();
    let span = group.span();
    let mut group = Group::new(delimiter, stream);
    group.set_span(span);
    TokenTree::Group(group)
},
...
```

 `cargo test` して成功した。

コミット: 37ab083

## 04-paste-ident

`f~N` のように、識別子に `~N` と続いていれば、 `f0`, `f1` のように結合するように変更する。

`Iterator` だと `[f, ~, N]` を検知してから `f0` の様に加工することは難しそうなので、 `Peekable` に対してこれを実現可能にするメソッドを実装する。 次が `~` であるかを確認するために、`Peekable` に対して実装する。また、 `replace_tokens` に渡していた情報を `ReplaceTokenState` で保持するようにする。

```rust
struct ReplaceTokenState<'a, 'b, T>
    where T: Iterator<Item=TokenTree> {
        iter: &'a mut Peekable<T>,
        ident_replace: &'b Ident,
        n: usize,
}

trait ReplaceToken<T: std::iter::Iterator<Item = proc_macro2::TokenTree>> {
    fn replace_token<'a>(
        &mut self,
        ident_replace: &'a Ident,
        n: usize
    ) -> ReplaceTokenState<'_, 'a, T>;
}

impl<T> ReplaceToken<T> for Peekable<T>
    where T: Iterator<Item=TokenTree>
{
    fn replace_token<'a>(&mut self, ident_replace: &'a Ident, n: usize) -> ReplaceTokenState<'_, 'a, T> {
        ReplaceTokenState {
            iter: self,
            ident_replace,
            n
        }
    }
}

impl<T: std::iter::Iterator<Item = proc_macro2::TokenTree>> Iterator for ReplaceTokenState<'_, '_, T> {
    type Item = TokenTree;

    fn next(&mut self) -> Option<Self::Item> {
        self.iter.next()
    }
}
```

`replace_tokens` の中を `Iterator for ReplaceTokenState` の `next` の中に移動する。 `token_tree`は `self.iter.next()?` の値を取得し、それを代入する。 `ident_replace`, `n` は `self` を参照するように変更する。 `.map(|token_tree| replace_tokens(token_tree, self.ident_replace, self.n))` のようにしていた箇所は、 `.peekable().replace_token(&ident_replace, n)` に置き換える。既存のテストが通れば、リファクタリングは成功。

加工前の `tokens` を確認すると、 `Ident("f"), Punct("~"), Ident("N")` となっていることがわかる。 `Ident` の次が `Punct("~")` であれば、その次の `Ident` を確認し、 置換対象であれば置換して最初の `Ident` と結合する。

```rust
impl<T: std::iter::Iterator<Item = proc_macro2::TokenTree>> Iterator for ReplaceTokenState<'_, '_, T> {
    type Item = TokenTree;

    fn next(&mut self) -> Option<Self::Item> {
        let token_tree = self.iter.next()?;
        let token_tree = match token_tree {
            TokenTree::Group(group) => {
                let delimiter = group.delimiter();
                let stream = group.stream().into_iter().peekable().replace_token(self.ident_replace, self.n).collect::<TokenStream2>();
                let span = group.span();
                let mut group = Group::new(delimiter, stream);
                group.set_span(span);
                TokenTree::Group(group)
            },
            TokenTree::Ident(ident) => {
                if let Some(TokenTree::Punct(next)) = self.iter.peek() {
                    if next.as_char() == '~' {
                        self.iter.next()?; // consume ~
                        let TokenTree::Ident(next) = self.iter.next().unwrap() else {
                            panic!("element followed after ~ is not ident");
                        };
                        let next = if next == *self.ident_replace {
                            self.n.to_string()
                        } else {
                            next.to_string()
                        };
                        TokenTree::Ident(Ident::new(&format!("{}{}", ident, next), Span::call_site()))
                    } else if ident == *self.ident_replace {
                        TokenTree::Literal(Literal::usize_unsuffixed(self.n))
                    } else {
                        TokenTree::Ident(ident)
                    }
                } else if ident == *self.ident_replace {
                    TokenTree::Literal(Literal::usize_unsuffixed(self.n))
                } else {
                    TokenTree::Ident(ident)
                }
            },
            TokenTree::Punct(_) => token_tree,
            TokenTree::Literal(_) => token_tree,
        };
        Some(token_tree)
    }
}
```

コミット: 503b00b

## before 05-repeat-section

05-repeat-section のテストでは、`enum` の中で `seq!` を呼びだすことは出来ないため、新たな構文 `#(...)*` を追加する。このまま構文を増やすと大変なので、 `Iterator` の使用から `seq::parse` の使用に変更する。

参考になる実装として、Rustの以下のコードの解析について調べる。

```rust
{
    enum A {
        a,
        b,
    }
    let a = 42;
    let b= 43;
    {
        false
    }
    true
}
```

このコードは最終的には `true` を返す *式* なので、 `Expr` でパースさせ、その結果を確認する。

```rust
let expr: Expr = parse_quote! {
    {
        enum A {
            a,
            b,
        }
        let a = 42;
        let b= 43;
        {
            false
        }
        true
    }
};
eprintln!("{:#?}", expr);
```
結果は、以下となった。

:::details 結果
```rust
Expr::Block {
    attrs: [],
    label: None,
    block: Block {
        brace_token: Brace,
        stmts: [
            Stmt::Item(
                Item::Enum {
                    attrs: [],
                    vis: Visibility::Inherited,
                    enum_token: Enum,
                    ident: Ident {
                        ident: "A",
                        span: #5 bytes(161..223),
                    },
                    generics: Generics {
                        lt_token: None,
                        params: [],
                        gt_token: None,
                        where_clause: None,
                    },
                    brace_token: Brace,
                    variants: [
                        Variant {
                            attrs: [],
                            ident: Ident {
                                ident: "a",
                                span: #5 bytes(161..223),
                            },
                            fields: Fields::Unit,
                            discriminant: None,
                        },
                        Comma,
                        Variant {
                            attrs: [],
                            ident: Ident {
                                ident: "b",
                                span: #5 bytes(161..223),
                            },
                            fields: Fields::Unit,
                            discriminant: None,
                        },
                        Comma,
                    ],
                },
            ),
            Stmt::Local {
                attrs: [],
                let_token: Let,
                pat: Pat::Ident {
                    attrs: [],
                    by_ref: None,
                    mutability: None,
                    ident: Ident {
                        ident: "a",
                        span: #5 bytes(161..223),
                    },
                    subpat: None,
                },
                init: Some(
                    LocalInit {
                        eq_token: Eq,
                        expr: Expr::Lit {
                            attrs: [],
                            lit: Lit::Int {
                                token: 42,
                            },
                        },
                        diverge: None,
                    },
                ),
                semi_token: Semi,
            },
            Stmt::Local {
                attrs: [],
                let_token: Let,
                pat: Pat::Ident {
                    attrs: [],
                    by_ref: None,
                    mutability: None,
                    ident: Ident {
                        ident: "b",
                        span: #5 bytes(161..223),
                    },
                    subpat: None,
                },
                init: Some(
                    LocalInit {
                        eq_token: Eq,
                        expr: Expr::Lit {
                            attrs: [],
                            lit: Lit::Int {
                                token: 43,
                            },
                        },
                        diverge: None,
                    },
                ),
                semi_token: Semi,
            },
            Stmt::Expr(
                Expr::Block {
                    attrs: [],
                    label: None,
                    block: Block {
                        brace_token: Brace,
                        stmts: [
                            Stmt::Expr(
                                Expr::Lit {
                                    attrs: [],
                                    lit: Lit::Bool {
                                        value: false,
                                    },
                                },
                                None,
                            ),
                        ],
                    },
                },
                None,
            ),
            Stmt::Expr(
                Expr::Lit {
                    attrs: [],
                    lit: Lit::Bool {
                        value: true,
                    },
                },
                None,
            ),
        ],
    },
}
```
:::

`Expr::Block` として解析され、 [`ExprBlock`](https://docs.rs/syn/latest/syn/struct.ExprBlock.html) に情報が格納される。実際のそれぞれの文の情報は [`Block`](https://docs.rs/syn/latest/syn/struct.Block.html) に格納されている。 `Block` の構造体の情報を元に、実装を進めることにする。

`seq!` の変更対象ブロックのトップレベルにある、関数名について実装する。

パースする処理を実装する。パースする種類が定義された `SeqTokenTree` に対して、それを保存する `SeqTrees` を定義する。

```rust
#[derive(Debug, Clone)]
struct SeqTrees {
    pub token_trees: Vec<SeqTokenTree>,
}
```

`SeqTokenTree` について考える。 処理したいものは `Ident ~ Ident` のみで、他はそのまま出力してほしいため、 `TokenTree` をそのまま保管するものと処理したい `Ident` をまとめたものを保管する。単体の置換対象の `Ident` に関しては、 `Raw` に格納して `TokenStream` に変換するときに処理することにする。

```rust
#[derive(Debug, Clone)]
enum SeqTokenTree {
    Raw(TokenTree),
    Ident(Vec<syn::Ident>),
}
```

パース処理を実装する。 `Ident ~ Ident` であれば `SeqTokenTree::Ident` に格納し、そうでなければ `SeqTokenTree::Raw` に格納するようにする。

```rust
impl Parse for SeqTrees {
    fn parse(input: syn::parse::ParseStream) -> syn::Result<Self> {
        let mut trees = Vec::<SeqTokenTree>::new();
        loop {
            if input.is_empty() {
                break;
            }
            let tree = input.parse::<TokenTree>()?;
            if input.peek(Token![~]) {
                if let TokenTree::Ident(ident) = tree {
                    input.parse::<Token![~]>()?;
                    let next = input.parse::<Ident>()?;
                    trees.push(SeqTokenTree::Ident(vec![ident, next]))
                } else {
                    trees.push(SeqTokenTree::Raw(tree));
                }
            } else {
                trees.push(SeqTokenTree::Raw(tree));
            }
        }
        Ok(Self {
            token_trees: trees
        })
    }
}
```

元の `TokenStream2` に変換する処理を実装したいが、 `N` と `0..4` に対応する情報が無いため、それらを格納した構造体 `SeqGroupToReplace` を定義する。

```rust
struct SeqGroupToReplace<'a, 'b> {
    n: usize,
    group: &'a SeqTrees,
    ident: &'b Ident,
}

impl<'a, 'b> SeqGroupToReplace<'a, 'b> {
    pub fn new(
        n: usize,
        group: &'a SeqTrees,
        ident: &'b Ident,
    ) -> Self {
        SeqGroupToReplace {
            n,
            group,
            ident
        }
    }
}
```

`SeqTokenTree::Ident` であれば新たに `Ident` を作り、 `Raw` の中が `Ident` かつ置換対象ならば `Literal` に置き換える。 `to_tokens` は `&mut TokenStream2` を引数にとり、それに追加する形で実装する。 `TokenStream2` は `IntoIterator<Item = TokenTree>>` を `extend` に渡すことで拡張できるようなので、 `token_trees` を逐次処理して `TokenTree` を出力するイテレータを作成し、 `extend` に渡す。なお、後に `Group` で再帰的に処理できるようにするため、 `SeqTokenTree` にメソッドを追加する形を取る。

```rust
impl SeqTokenTree {
    fn to_token_tree(
        &self,
        n: usize,
        ident_replace: &Ident,
    ) -> TokenTree {
        match self {
            SeqTokenTree::Raw(tree) => {
                if let TokenTree::Ident(ident) = tree {
                    if ident == ident_replace {
                        TokenTree::Literal(Literal::usize_unsuffixed(n))
                    } else {
                        TokenTree::Ident(ident.clone())
                    }
                } else {
                    tree.clone()
                }
            },
            SeqTokenTree::Ident(idents) => {
                let new_ident_name = idents.iter().map(|ident| {
                    if ident == ident_replace {
                        n.to_string()
                    } else {
                        ident.to_string()
                    }
                }).collect::<Vec<_>>().join("");
                TokenTree::Ident(proc_macro2::Ident::new(&new_ident_name, Span::call_site()))
            },
        }
    }
}
impl ToTokens for SeqGroupToReplace<'_, '_> {
    fn to_tokens(&self, tokens: &mut TokenStream2) {
        let tokentree_iter = self.group.token_trees.clone().into_iter().map(|tree| {
            match tree {
                SeqTokenTree::Raw(tree) => {
                    if let TokenTree::Ident(ident) = tree {
                        if ident == *self.ident {
                            TokenTree::Literal(Literal::usize_unsuffixed(self.n))
                        } else {
                            TokenTree::Ident(ident)
                        }
                    } else {
                        tree
                    }
                },
                SeqTokenTree::Ident(idents) => {
                    let new_ident_name = idents.iter().map(|ident| {
                        if ident == self.ident {
                            self.n.to_string()
                        } else {
                            ident.to_string()
                        }
                    }).collect::<Vec<_>>().join("");
                    TokenTree::Ident(proc_macro2::Ident::new(&new_ident_name, Span::call_site()))
                },
            }
        });

        tokens.extend(tokentree_iter);
    }
}
```

以下のコードを追加して、 `fn f0() -> u64 { N * 2 }` と出力されているか確認する。

```rust
let seq_token_tree = syn::parse2::<SeqTrees>(tokens.clone());
eprintln!("{:#?}", seq_token_tree);
if let Ok(group) = seq_token_tree {
    let replaced = SeqGroupToReplace::new(0, &group, &ident_replace);
    eprintln!("{}", replaced.to_token_stream());
}
```

次は `Group` 内の処理を実装する。 `Group` には `delimiter` と `stream` の情報が必要であるため、それらを保管する `SeqGroup` を定義する。また、 03-expand-four-errors の実装の記録も考慮し、 `Span` も格納する。

```rust
#[derive(Debug, Clone)]
struct SeqGroup {
    delimiter: Delimiter,
    trees: SeqTrees,
    span: Span,
}
```

`Group` の `stream` を再帰的にパースし、 `SeqGroup` に格納する。

```rust
if input.peek(Token![~]) {
    ...
} else if let TokenTree::Group(group) = tree {
    let group_trees = syn::parse2::<SeqTrees>(group.stream())?;
    let seq_group = SeqGroup {
        delimiter: group.delimiter(),
        trees: group_trees,
        span: group.span(),
    };
    trees.push(SeqTokenTree::Group(seq_group));
} else {
    ...
}
```

`TokenTree` への変換は、以下のようになる。

```rust
match self {
    ...,
    SeqTokenTree::Group(seq_group) => {
        let stream = seq_group.trees.token_trees.iter().map(|v| v.to_token_tree(n, ident_replace)).collect::<TokenStream2>();
        let mut group = proc_macro2::Group::new(seq_group.delimiter, stream);
        group.set_span(seq_group.span);
        TokenTree::Group(group)
    },
}
```

コミット: 94eaaf2

## 05-repeat-section

`#(...)*` の構文を追加する。この構文は、もしなければ全体を、もしあればその箇所のみ繰り返すように処理する。
`#(...)*` の情報を格納するために、 `SeqTokenTree::Trees(SeqTrees)` を追加する。また、 `SeqTrees` には `#(...)*` から取得された情報か判別するために `to_expand: bool` を追加する。

```rust
fn parse(input: syn::parse::ParseStream) -> syn::Result<Self> {
    ...
    loop {
        if input.is_empty() {
            break;
        }
        if input.peek(Token![#]) && input.peek2(syn::token::Paren) {
            let content;
            input.parse::<Token![#]>()?;
            parenthesized!(content in input);
            input.parse::<Token![*]>()?;
            let mut seq_trees = content.parse::<SeqTrees>()?;
            seq_trees.to_expand = true;
            trees.push(SeqTokenTree::Trees(seq_trees));
            continue;
        }
        ...
    }
    ...
}
```

`#(...)*` が1つも無かった場合がトップレベルの `SeqTrees` の `to_expand` を `true` に出来るように、`SeqTrees` の中身を見て `to_expand` があるものがあれば `true` を返却する関数を作成する。

```rust
impl SeqTrees {
    fn has_to_expand(&self) -> bool {
        if (self.to_expand) {
            true
        } else {
            self.token_trees.iter().any(|tree| {
                match tree {
                    SeqTokenTree::Raw(_) => false,
                    SeqTokenTree::Ident(_) => false,
                    SeqTokenTree::Group(group) => group.trees.has_to_expand(),
                    SeqTokenTree::Trees(tree) => tree.has_to_expand(),
                }
            })
        }
    }
}
```

これで、 `!seq_token_tree.has_to_expand()` の結果をトップレベルの `SeqTrees` に代入すれば統一して処理できるようになる。

一部を繰り返さず、特定箇所のみを繰り返す処理が必要になったので、 `SeqGroupToReplace` に `start` と `end` の情報を持たせるようにする。

`SeqTokenTree::to_token_tree` で `SeqTokenTree::Trees` の処理をする。これまでは `SeqTokenTree` とそれぞれに対応していた `TokenTree` を返却していたが、対応していない `Trees` が追加されたため、汎用的に返却できるよう `TokenStream2` を返却するように変更する。 `From<TokenTree> for TokenTree` が実装されているので、既存の返却箇所には `.into()` をつけるだけで良い。

`Trees` の展開を実装する。 `to_expand` が `true` なら `start..end` のそれぞれの要素 `n` で展開する処理を行なう。このとき、 `start`, `end` それぞれは `n`, `n+1` とする。既存の `n` としていた箇所は `start` を参照するように変更する。

```rust
SeqTokenTree::Trees(trees) => {
    if trees.to_expand {
        (start..end).map(|n| {
            trees.token_trees.iter().map(|v| v.to_token_tree(n, n+1, ident_replace)).collect::<TokenStream2>()
        }).collect::<TokenStream2>()
    } else {
        trees.token_trees.iter().map(|v| v.to_token_tree(start, end, ident_replace)).collect::<TokenStream2>()
    }
},
```

コミット: ab3352c

## 06-init-array

特に変更の必要は無し。

コミット: 1e3e005

## 07-inclusive-range

`a..=b` の対応を行なう。 これまでは `start, end` の実装をしてきたが、この実装だと `b` が `usize::MAX` だったときに破綻するため、 `Iterator` を使用して出力するように変更する。

`Seq` に `..=` で格納されたかの情報 `inclusive` を追加する。パース処理は `..` パース処理後に以下を追記するだけである。

```rust
let inclusive = input.parse::<Option<Token![=]>>()?.is_some();
```

`a..b` は `std::ops::Range` であり、 `a..=b` は `std::ops::RangeInclusive` と異なる構造体が返却される。
統一して格納できるようにするため、 `Iterator<Item=usize>` を実装した `enum` を作成する。

```rust
#[derive(Clone)]
enum RangeWrapper {
    Range(Range<usize>),
    RangeInclusive(RangeInclusive<usize>)
}

impl Iterator for RangeWrapper {
    type Item = usize;

    fn next(&mut self) -> Option<Self::Item> {
        match self {
            RangeWrapper::Range(r) => r.next(),
            RangeWrapper::RangeInclusive(r) => r.next(),
        }
    }
}

impl From<Range<usize>> for RangeWrapper {
    fn from(value: Range<usize>) -> Self {
        Self::Range(value)
    }
}

impl From<RangeInclusive<usize>> for RangeWrapper {
    fn from(value: RangeInclusive<usize>) -> Self {
        Self::RangeInclusive(value)
    }
}
```

`From` を実装したので、 `RangeWrapper` は以下のように作成できる。

```rust
let range: RangeWrapper = if inclusive { (start..=end).into() } else { (start..end).into() };
```

`SeqGroupToReplace` では `start`, `end` をそれぞれ格納する代わりに `RangeWrapper` を `range` に格納するように変更し、 `start..end` は `range.clone()` に置き換える。

他の関連箇所も `range` を参照するように置き換えるが、 展開時に展開する数値を1つだけ出力するイテレータ [`std::iter::Once`](
https://doc.rust-lang.org/std/iter/struct.Once.html) を受け付けたいため、任意の `Iterator<Item=usize>+Clone` を受け付けるように変更する。

```rust
impl SeqTokenTree {
    fn to_token_tree<T>(
        &self,
        iter: &mut T,
        ident_replace: &Ident,
    ) -> TokenStream2 where T: Iterator<Item=usize> + Clone{
        ...
    }
}
```

コミット: 89f1a3b

## 08-ident-span

変数がなかったときに、エラーの箇所を `Ident` の箇所になるようにする。

```rust
SeqTokenTree::Ident(idents) => {
    ...
    let span = idents.first().map(|ident| ident.span()).unwrap_or(Span::call_site());
    TokenTree::Ident(proc_macro2::Ident::new(&new_ident_name, span)).into()
},
```

## 09-interaction-with-macrorules

このテストでは `seq!` 内で定数を使用したい場合の回避策を示している。追加の実装は必要無し。

コミット: a777db0
