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
