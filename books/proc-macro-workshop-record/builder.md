---
title: "builder"
---

# builder

## 01-parse

このテストでは空の[TokenStream](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html)を返却すれば良いとコメントに書かれているため、シンプルに空のTokenStreamを返す。[TokenStream::new](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html#method.new) がまさにそれを生成するメソッドのため、その結果を返す。

コミット: 662ee91

## 02-create-builder

Resourcesの項目に、マクロ内でRustの構文の作成を簡単にする [quote](https://github.com/dtolnay/quote) というCrateが紹介されている、 `cargo add quote` で追加する。また、01-parseでTokenStreamのパースのために[syn](https://github.com/dtolnay/syn) というCrateが追加されているので、そちらも追加する。デバッグのために `extra-traits` Feature を有効化して、`cargo add syn -F extra-traits` で追加する。

また、これらのライブラリのREADMEを参照すると、[proc-macro2](https://github.com/dtolnay/proc-macro2)をproc-macroのラッパーとして使用しているため、それも追加する。

### 嘘解答

テストファイルのコメント内に欲しい出力があるため、それを `quote!` マクロにそのまま渡す。テストは正常に通り、 `cargo expand` でも期待した通りの結果になる。  
もちろん、これだとテスト内の `Command` にしか使用できないマクロとなってしまうため、ここから汎用性を持つようにリファクタリングしていく。

コミット: ab98b954239987d6235aa2303cc0e56c27a88342

### パース

01-parseで紹介されていた `syn` を使用する。READMEにderive macroの例があるため、そこを参考にする。

```rust
let input = parse_macro_input!(input as DeriveInput);
```

01-parse では [docs.rsのDeriveInput](https://docs.rs/syn/2.0/syn/struct.DeriveInput.html) を見ると良いと言われているが、いきなり見てもわからないのでまずパースしたものを出力し、その内容から関連しそうな項目を確認する方法を取る。パース結果の出力は proc-macro-workshopの Debugging tips にある、 `eprintln!("INPUT: {:#?}", syntax_tree)` を使用することにする。今回はパースしてすぐの `input` を出力する。

```
DeriveInput {
    attrs: [],
    vis: Visibility::Public(
        Pub,
    ),
    ident: Ident {
        ident: "Command",
        span: #0 bytes(1374..1381),
    },
    generics: Generics {
        lt_token: None,
        params: [],
        gt_token: None,
        where_clause: None,
    },
    data: Data::Struct {
        ...略 構造体の中身の情報
    }
}
```

### 構造体名に関わる箇所の置き換え

上記の出力から、 `input.ident` が構造体名であることがわかったため、対象の `Ident` の参照を変数に保持する。

```rust
let name = &input.ident;
```

`impl Command { ... }` としていた箇所は、これで `impl #name { ... }` とすることが出来るようになる。

`CommandBuilder` は新たに `Ident` を生成する必要がある。02-create-builderのResourcesの項目にある、 [docs.rsのIdent](https://docs.rs/syn/2.0/syn/struct.Ident.html)を参照すると新たにIdentを生成する例が記載されているため、そちらに従って生成する。

::: message 
quoteのREADMEにも `syn::Ident` を生成する方法が記載されている。そちらのほうが分かり易いかもしれない。 [^quote-create-ident]
:::

[^quote-create-ident]: [記載箇所](https://github.com/dtolnay/quote/blob/bdb4b594076d78127b99a3da768e369499e324de/README.md#constructing-identifiers)

```rust
let builder_name = syn::Ident::new(&format!("{}Builder", name), proc_macro2::Span::call_site());
```

これで `CommandBuilder` の箇所を、 `#builder_name` に置き換えることが出来る。

### 構造体の内容に関わる箇所の置き換え

前提として、構造体である必要があるため、 `input.data` が `syn::Data::Struct` でなければ `panic!` を呼び出して処理を中断させる。  
`input.data` の中身を確認したところ、 `fields` にフィールドの情報が含まれていることがわかった。  

```rust
let Data::Struct(data_struct) = input.data else {
    panic!("builder is not used for struct");
};

```

まずはフィールド名だけ抜き出せば良い、 `CommandBuilder` の初期値のフィールドから実装する。[Field](https://docs.rs/syn/2.0.51/syn/struct.Field.html)では `ident` が `Option` となっているが、これは無名フィールドと共用しているためのようである。今回は名前があることがわかっているため、単に無視することとする。

`quote` のREADMEを眺めると、quote!の結果をquote!の中で使うことが出来るし、`IntoIterator` を実装していれば `macro_rules!` のような書式で繰り返し展開することが出来ることがわかる。

`#ident: None` を生成して、繰り返し出力させるようにする。

```rust
let fields = data_struct
    .fields
    .iter()

let builder_init = fields.clone().filter_map(|field| {
    field.ident.as_ref().map(|ident| {
        quote! {
            #ident: None
        }
    })
});

quote!{
    // 略
    impl #name {
        pub fn builder() -> #builder_name {
            #builder_name {
                #(#builder_init),*
            }
        }
    }
}
```

次に、`CommandBuilder` の型を実装する。 `CommandBuilder` の型はそれぞれの元の型を `Option` で包んだ型となっている。フィールドの型の情報は `field.ty` に格納されているため、以下のようにする。

```rust
let builder_field = fields.clone().filter_map(|field| {
    let ty = &field.ty;
    field.ident.as_ref().map(|ident| {
        quote! {
            #ident: Option<#ty>
        }
    })
});
```

コミット 24d56bb

## 03-call-setters

完成系のメソッドがコメント内にあるため、そこから作成する。

```rust
fn executable(&mut self, executable: String) -> &mut Self {
    self.executable = Some(executable);
    self
}
```

これは、これまで使用したプロパティの情報を用いれば用意に作成できる

```rust
fn #ident(&mut self, #ident: #ty) -> &mut Self {
    self.#ident = Some(#ident);
    self
}
```

## 04-call-build

`CommandBuilder` から実際の `Command` を作成する、 `build` メソッドを実装する。始めに、嘘解答を作成する。このとき、なるべく同じ構文を繰り返すようにすることと、現段階でテストケースに求められていないものはなるべく単純化するように方針を定める。今回は `None` が見つかったら即座に `Err(Box<dyn Error>)` を返却するようにする。エラーメッセージは同じ内容にしてしまう。

### 嘘解答

```rust
pub fn build(&mut self) -> Result<Command, Box<dyn Error>> {
    let Some(executable) = self.executable.clone() else {
        return Err("field is not enough".to_string().into());
    };
    let Some(args) = self.args.clone() else {
        return Err("field is not enough".to_string().into());
    };
    let Some(env) = self.env.clone() else {
        return Err("field is not enough".to_string().into());
    };
    let Some(current_dir) = self.current_dir.clone() else {
        return Err("field is not enough".to_string().into());
    };

    Ok(Command {
        executable,
        args,
        env,
        current_dir,
    })
}
```

上記コードで実装して展開しようと、以下のエラーが出る。

```
error[E0405]: cannot find trait `Error` in this scope
```

これは Rust が何の `Error` を使用するか認識できないためである。コンパイルエラー修正の提案として、 `use std::error::Error` を入れるように促しているが、マクロ側で `Error` の箇所を `std::error::Error` のようにフルパスで指定することで回避する。

### 実装方法を検討する

`build` メソッドは `let Some(ident) = self.ident.clone() else { ... }` でガードする項目と、 `Command` を作る項目の2つから構成されていると考えることができる。各フィールドのガードを `field_guards`, 各フィールド名を `field_idents` に格納していると過程すると、以下のようになる。

```rust
pub fn build(&mut self) -> Result<Command, Box<dyn Error>> {
    #(#field_guards)*

    Ok(Command {
        #(#field_idents),*
    })
}
```

### 実装

`field_guards` について検討する。フィールド名 `#ident` に対し、以下のコードを生成する。

```rust
let Some(#ident) = self.#ident.clone() else {
    return Err("field is not enough".to_string().into());
};
```

特に難しいことはなさそう。

```rust
let field_guards = fields.clone().filter_map(|field| {
    field.ident.as_ref().map(|ident| {
        quote! {
            let Some(#ident) = self.#ident.clone() else {
                return Err("field is not enough".to_string().into());
            };
        }
    })
});
```

`field_idents` については何も難しいことはなく、 `ident` を抜き出すだけで良い。

```rust
let field_idents = fields.clone().filter_map(|field| field.ident.as_ref());
```

## 05-method-chaining

このテストではメソッドチェインできるようにするものだが、03-call-setterの中で既に実装されているため、テストの有効化だけして終了。

コミット: c86c807

## 06-optional-chaining

このテストでは、 `Option` 型が指定されていれば、そのフィールドの指定をしなくても `build` が成功するように変更しなければならない。ここで、テスト先頭のコメントには "コンパイラはマクロの展開が完全に終了したあとに名前解決を実行する。そのため、マクロの評価中はトークンから実際の型が何かを調べることが出来ず、トークンのみを確認することのみ出来る" といったことが書かれている。

`Option` を表現する方法もたくさんあることが触れられているが、今回は `Option<T>` のケースのみ考慮する。(大抵の場合は `Option` は `std::option::Option` で使用されるので、考慮することは多分少ない。)

始めに、完成結果について考える。

```rust
pub struct CommandBuilder {
    ...
    current_dir: Option<String>, // Command.current_dirから変更が無い
}

impl CommandBuilder {
    ...
    // 引数の型がCommand.current_dirと違い、Optionの中のTを受け入れるようになっている
    pub fn current_dir(&mut self, current_dir: String) -> &mut Self {
        self.current_dir = Some(executable);
        self
    }

    pub fn build(&mut self) -> Result<Command, Box<dyn std::error::Error>> {
        ...
        // CommandBuilder.current_dirとCommand.current_dirの型は同じなので、
        // 単にcloneするだけで良くなる
        let current_dir = self.current_dir.clone();
        ...
    }
}
```

やることは以下の3つである。

1. `CommandBuilder` の型のフィールドを変更
1. セッターの変更
1. `build` 内の処理の変更

型が `Option<T>` か否かで変更をすれば良さそうにみえる。

`Type` が `Option<T>` の形式か調べるメソッド `fn is_option(ty: &Type) -> bool` を作成する。
`Option<T>` がどのような形式のデータになるかはコメントにあるため、有り難くそれを参照する。

```rust
fn is_option(ty: &Type) -> bool {
    let Type::Path(type_path) = ty else {
        return false;
    };

    type_path.path.segments.first().map(|segment| segment.ident == "Option").unwrap_or(false)
}
```

動作確認は以下のコードで行なう。

```rust
fields.clone().for_each(|field| {
    eprintln!("{}", is_option(&field.ty));
});
```

stderrに以下のように出力されていれば成功。これはフィールドの上から順の結果と対応している。

```
false
false
false
true
```

型の情報が不要な、 `CommandBuilder` のフィールドと `build` 内のガードについて考える。

```rust
let builder_field = fields.clone().filter_map(|field| {
    let ty = &field.ty;
    field.ident.as_ref().map(|ident| {
        if is_option(ty) {
            quote! {
                #ident: #ty
            }
        } else {
            quote! {
                #ident: Option<#ty>
            }
        }
    })
});

let field_guards = fields.clone().filter_map(|field| {
    let ty = &field.ty;
    field.ident.as_ref().map(|ident| {
        if is_option(ty) {
            quote! {
                let current_dir = self.current_dir.clone();
            }
        } else {
            quote! {
                let Some(#ident) = self.#ident.clone() else {
                    return Err("field is not enough".to_string().into());
                };
            }
        }
    })
});
```

セッターの実装について検討する。セッターの変更自体は引数のみの変更であるが、そのためには `Option<T>` の中の `T` の情報を取りださなければならない。

テストコードのコメントを参考に、最も単純に `GenericArgument::Type` の中を取得するコードを作成する。

```rust
fn get_type_in_generics(ty: &Type) -> Option<&Type> {
    let Type::Path(type_path) = ty else {
        return None;
    };
    let PathArguments::AngleBracketed(ref args) = type_path.path.segments.first()?.arguments else {
        return None;
};
    let Some(GenericArgument::Type(ty)) = args.args.first() else {
        return None;
    };
    Some(ty)
}
```

上記のメソッドを用いて、以下のように変更する。

```rust
let setters = fields.clone().filter_map(|field| {
    let ty = &field.ty;
    field.ident.as_ref().map(|ident| {
        if is_option(ty) {
            let arg_ty = get_type_in_generics(ty);

            quote! {
                pub fn #ident(&mut self, #ident: #arg_ty) -> &mut Self {
                    self.#ident = Some(#ident);
                self
            }
        } else {
            quote! {
                pub fn #ident(&mut self, #ident: #ty) -> &mut Self {
                    self.#ident = Some(#ident);
                self
            }
        }
    })
});
```

コミット: db9ea58

## 07-repeated-field

このテストでは `#[builder(each = "...")]` がついているフィールドについては、 型が `Vec<T>` であると仮定して、 `each` に対応するものをメソッド名としてセッターを作成するようにする。セッター内では、実際にセットする代わりにフィールドの `Vec` に追加する処理を行なう。

テストコードに着目すると、 `env` にも指定されているが、 `Option` 型を指定したとき同様、セッター関数を呼ばれなくても `build` が成功している。

```rust
fn arg(&mut self, arg: String) {
    self.args.push(arg);
    self
}
```

まず、マクロで `builder` を処理できるように、テストコードのコメント内で示されているように、 `proc_macro_derive` の行を `#[proc_macro_derive(Builder, attributes(builder))]` に変更する。

次に、フィールドに対して、 `builder(each="...")` の `each` に指定されている値を返すメソッドを作成する。常に指定されているとは限らないので、`Option<T>` を返却する。 `T` の具体的な型は、実際の属性のデータを確認してから決定することにする。

確認のために、 `args` フィールドの中のみを表示することにする。そのために、`field` のフィールド名が `args` かどうか比較する必要がある。 `field` の名前は `field.ident` から取得できることはこれまでの実装で分かっているので、 `field.ident` と `"args"` で比較する方法を考える。 `field.ident` は `Option<Ident>` 型であるため、`Ident` を `&str` と比較する方法を考える。 `Ident` のドキュメントを見ると、 `impl<T> PartialEq<T> for Ident where T: AsRef<str> + ?Sized` が実装されているので、そのまま `&str` と比較できることがわかる。そのため、以下のコードを仕込むことで、 `args` のみの情報を表示することができる。

```rust
fields.clone().for_each(|field| {
    if field.ident.as_ref().is_some_and(|ident| ident == "args")  {
        eprintln!("{:#?}", field);
    }
});
```

結果を確認すると、 `field.attrs` の最初の要素の `meta` に `builder(each = "env")` の情報が格納されていることがわかる。また、 `meta.tokens` には `each = "env"` の情報が格納されていることがわかる。

`each` の値を取得するメソッドを作成する。引数は `&[Attribute]` とし、返り値は `Option<T>` とする。 `T` の値は後ほど決定する。

まず、 `T` が `&proc_macro2::TokenStream` を返却する処理を作成する。

```rust
fn get_value_of_each(attrs: &[Attribute]) -> Option<&proc_macro2::TokenStream> {
    let Meta::List(ref list) = attrs.first()?.meta else {
        return None
    };

    Some(&list.tokens)
}
```

`Meta::List` 内のデータを直接見て解析しても良いが、 [`Attribute` の docs.rs](https://docs.rs/syn/2.0.50/syn/struct.Attribute.html) を確認すると、 `Meta::List` をパースする `parse_args` が提供されているので、そちらを利用する。 `parse_args` には型の指定が必要なので、どの型に対応するか検討する。 パースしたい対象は `each = "env"` のような構文である。 `Attribute` の docs.rsを確認すると、 `#[path = "sys/windows.rs"]` のような構文は `Meta::NameValue` として格納されるとある。 `Meta::NameValue` は `MetaNameValue` 構造体を含んだEnumであるため、 `MetaNameValue` としてパースしてみて結果を確認する。

```rust
Some(
    Ok(
        MetaNameValue {
            path: Path {
                leading_colon: None,
                segments: [
                    PathSegment {
                        ident: Ident {
                            ident: "each",
                            span: #0 bytes(254..258),
                        },
                        arguments: PathArguments::None,
                    },
                ],
            },
            eq_token: Eq,
            value: Expr::Lit {
                attrs: [],
                lit: Lit::Str {
                    token: "arg",
                },
            },
        },
    ),
)
```

正常にパースできた。この方針で進めることにする。欲しいデータが `Lit::Str` に格納されていることがわかった。これは `LitStr` を含む構造体である。 `LitStr` のdocs.rsを確認すると、 `value(&self) -> String` が実装されているので、 `get_value_of_each` は最終的に `String` を返却するように実装する。

```rust
fn get_value_of_each(attrs: &[Attribute]) -> Option<String> {
    attrs
        .first()
        .and_then(|attr| attr.parse_args::<MetaNameValue>().ok())
        .filter(|name_value| name_value.path.is_ident("each"))
        .and_then(|name_value| {
            let Expr::Lit(lit) = name_value.value else {
                return None;
            };

            let Lit::Str(lit_str) = lit.lit else {
                return None;
            };

            Some(lit_str.value())
        })
}
```

`get_each_arg` の返り値が `Some` かどうかで、 `each` が指定されているかどうかを判別する。

`CommandBuilder` の型を変更する。 `each` が設定されていれば、 `Vec` が設定されているはずなので `Command` のフィールドの型をそのまま使用することにする。

```rust
let builder_field = fields.clone().filter_map(|field| {
    let ty = &field.ty;
    field.ident.as_ref().map(|ident| {
        if is_option(ty) || get_value_of_each(&field.attrs).is_some() {
            ...
        } else {
            ...
        }
    })
});
```

初期値は、 `None` ではなく `Vec::new()` を使用することにする。

```rust
let builder_init = fields.clone().filter_map(|field| {
    let attrs = &field.attrs;
    field.ident.as_ref().map(|ident| {
        if get_value_of_each(attrs).is_some() {
            quote! {
                #ident: Vec::new()
            }
        } else {
            quote! {
                #ident: None
            }
        }
    })
});
```

セッター関数は、 `each` に設定されている文字列を関数名とする実装を追加する。本来、`args` と `arg` メソッドを用意するべきだが、簡単に実装するために `each` が設定されていればそちらのみを実装するようにする。

```rust
let setters = fields.clone().filter_map(|field| {
    let ty = &field.ty;
    let attrs = &field.attrs;
    field.ident.as_ref().map(|ident| {
        if is_option(ty) {
            ...
        } else if let Some(each) = get_value_of_each(attrs) {
            let each = Ident::new(&each, Span::call_site());
            let arg_ty = get_type_in_generics(ty);
            quote! {
                pub fn #each(&mut self, #each: #arg_ty) -> &mut Self {
                    self.#ident.push(#each);
                    self
                }
            }
        } else {
            ...
        }
    })
});
```

`build` 関数内に関しては、 `CommandBuilder` で `Vec` を入れるようにしたため、 `Option<T>` 型のときと同様、 単に `clone` だけ行なうようにする。

```rust
let field_guards = fields.clone().filter_map(|field| {
    let ty = &field.ty;
    let attrs = &field.attrs;
    field.ident.as_ref().map(|ident| {
        if is_option(ty) || get_value_of_each(attrs).is_some() {
            quote! {
                let #ident = self.#ident.clone();
            }
        } else {
            ...
        }
    })
});
```

コミット: 2df261072ba73b4360f48508a8188243c0471c36

## 08-unrecognized-attribute

このテストでは `each` 以外の値、例えば `eac` などが見つかったら、その箇所をコンパイルエラーとして表示する。
テストコードのコメントを見るに、 `syn::Error` を作成して `to_compile_error` を呼び出すことで、コンパイルエラーとして適切な表示にできるようだ。

始めに、全てのフィールドの属性を順番に検査して、1つでも見つかればその箇所のみのエラーを返却するメソッドを作成する。 `Error::new` の引数には、仮で現在検査している属性の `Span` である、 `attr.span()` を入れることにする。

```rust
fn get_unexpected_attributes(attrs: &[Attribute]) -> Option<syn::Error> {
    attrs
        .first()
        .filter(|attr|
            attr.parse_args::<MetaNameValue>()
                .ok()
                .is_some_and(|name_value| !name_value.path.is_ident("each")))
        .map(|attr| syn::Error::new(attr.span(), ""))
}
```

`syn::Error::to_compile_error` は `TokenStream` を返却するが、 `TokenStream` は `quote::ToTokens` を実装していることが [`ToTokens` の docs.rs](https://docs.rs/quote/latest/quote/trait.ToTokens.html#impl-ToTokens-for-TokenStream) からわかるので、結果を `quote!` の中に埋め込むことが出来る。また、 `impl<T: ToTokens> ToTokens for Option<T>` も実装されているので、特に `unwrap` せずに使用できることもわかる。

ひとまず `get_unexpected_attributes` の結果を `quote!` に埋め込み、実行する。

```rust
let unexpected_attrs = fields.clone().find_map(|field|
    get_unexpected_attributes(&field.attrs).map(|err| err.to_compile_error())
);
```

テストを実行すると、 `mismatch` と表示された。コンパイルエラーにすることは出来たものの、コンパイルエラーの表示が合っていないようである。

ではここで、期待されるエラーの表示を確認する。このworkshopではテストに [trybuild](https://docs.rs/trybuild/latest/trybuild/) を使用しているが、期待されるエラーメッセージは `*.stderr` というファイル名で保存されている。 `test/08-unrecognized-attribute.stderr` を確認すると、エラーメッセージは `` expected `builder(each = "...")` `` であることと、エラーの範囲が `builder(eac = "arg")]` であることが分かる。対して、現在の実装はエラーの範囲が `#[builder(eac = "arg")]` になっている。

まず、エラーメッセージを変更する。これは単に `syn::Error::new` の第2引数を変更すれば良い。次に、エラー範囲の変更である。該当箇所は `07-repeated-field` で処理した `attr.meta` から `Span` を取得すれば良い。幸い、 `syn::Meta` は `Spanned` を実装しているので `attr.meta.span()` を呼び出す処理に置き換えれば良い。

```rust
fn get_unexpected_attributes(attrs: &[Attribute]) -> Option<syn::Error> {
    attrs
        .first()
        .filter(|attr|
            attr.parse_args::<MetaNameValue>()
                .ok()
                .is_some_and(|name_value| !name_value.path.is_ident("each")))
        .map(|attr| syn::Error::new(attr.meta.span(), "expected `builder(each = \"...\")`"))
}
```

コミット: 013fda9
