---
title: bitfield
---

# bitfield

## 01-specifier-types

`B1` ~ `B64` の型を作る。 関連定数 `BITS` を持つ `Specifier` トレイトを作成し、 `B~N` で `BITS` に `N` を持たせる。

`seq!` が便利なので、 `Cargo.toml` の `dependencies` に `seq = { path = "../seq" }` を追加して使えるようにしておく。

```rust
pub trait Specifier {
    const BITS: usize;
}

seq!(N in 0..64 {
    pub enum B~N {}

    impl Specifier for B~N {
        const BITS: usize = N;
    }
});
```

コミット: df2e177

## 02-storage

テストコードのコメント中にあるような構造体に変換して出力する。 `#size` 相当箇所は `(0 + B1::BITS + B2::BITS + ...)/8` といった形式で出力する。

`quote`, `syn`, `proc_macro2` を `impl` ディレクトリに移動したうえで追加する。

```rust
#[proc_macro_attribute]
pub fn bitfield(args: TokenStream, input: TokenStream) -> TokenStream {
    let _ = args;
    let item_struct = parse_macro_input!(input as ItemStruct);

    let field_tys = item_struct.fields.iter().map(|field| &field.ty);

    quote! {
        #[repr(C)]
        pub struct MyFourBytes {
            data: [u8; (0 #(+ #field_tys::BITS)* )/8],
        }
    }.into()
}
```

コミット: e2d0714

## 03-accessors

実際の setter, getter を実装する。型は `u64` で統一する。

まず、実際の出力するコードを決めるため、 `main.rs` に `TheirFourBytes` を実装する。
複数にまたがる、オフセット付きの実装のために、レイアウトを以下のようにする。

```rust
#[bitfield]
pub struct MyFourBytes {
    a: B1,
    b: B3,
    c: B24,
    d: B4,
    e: B16,
}
```

すなわち、`c` は 全てのbytesにまたがる配列になる。 `e` はまたがらない場合の動作確認のために追加した。

コミット: 4a339a310a8afa2f74872007fe566147dcfe0b63

実装した内容で、別メソッドに切り出して、セッターとゲッターの内容をマクロ内にリファクタリングする。
一番の難所は仮処理の実装なので、特にマクロとしては困難なく実装(移植)できる。
よりよい bitfieldsの実装は他にもたくさんあると思うので、実装の解説は他を参照してもらう。

`new`, 各種セッター、ゲッターをマクロ内で展開する。

コミット: d0aa0a9fefb0b75abf7a8b5ff2d8764f28910a41

仮実装は `main.rs` から `examples` に移動した。

## 04-multiple-of-8bits

各フィールドのビット数の総和が8の倍数でなければ、コンパイルエラーを表示するようにする。
proc_macro_workshopの作者の `case-studies` というリポジトリに `bitfield-assertion` という項目があり、これがちょうどその内容なのでこれを参考に実装する。

@[card](https://github.com/dtolnay/case-studies/blob/master/bitfield-assertion/README.md)

`Solution` の `require_multiple_of_eight` のマクロ以外の定義を `checks.rs` に記載する。
`require_multiple_of_eight` の代わりに、 `const _: self::checks::MultipleOfEight<[(); (0 #(+ #field_bits)* )% 8]> = ();` を `impl` ブロックの下に挿入する形でコンパイルエラーを出力させる形を取る。 `field_bits` は各フィールドの型 `ty` に対して、 `#ty::BITS` に変換した `Vec<TokenStream2>` である。

トレイト名が `bitfields::checks` が表示されない形でエラーが出力された。

`stderr` の上書きが許可されているので、現在の実装で `stderr` を上書きする。

`const _: self::checks::MultipleOfEight<[(); (0 #(+ #field_bits)* )% 8]> = ();` はコンパイル時に要素数が計算される。
`self::checks::MultipleOfEight<[(); 1]>` のような形式に変換される。

`MultipleOfEight<[(); 1]>` は `<<[(); 1] as Array>::Marker as TotalSizeIsMultipleOfeightBits>::Check;` に展開される。
`[(); 1]` は `Array` トレイトを実装しており、 `Marker = OneMod8` が定義されており、 `<OneMod8 as TotalSizeIsMultipleOfeightBits>::Check` の段階で、 `TotalSizeIsMultipleOfeightBits` を実装されていないのでエラーとなる。実装されている `ZeroMod8` だったら `<ZeroMod8 as TotalSizeIsMultipleOfeightBits>::Check` は `()` となるので、 `const _: () = ()` が成立し、コンパイルエラーとならない。

zjp-CN氏のアプローチはよりシンプルで、コンパイル時にフィールド総ビット数を計算し、0でなければ `panic!` を呼び出して、コンパイルエラーを起こしている。

https://github.com/zjp-CN/proc-macro-workshop/blob/c0b87cf933be7d713dec504d3c5f832a4df240e0/bitfield/impl/src/bit.rs#L82

背景として、このワークショップ自体は 2019年にカンファレンスで紹介されたもの(最初のコミットも2019年)であること ([多分セッションの動画はこれ](https://www.youtube.com/watch?v=g4SYTOc8fL0))と、 `const` 内に `panic!` が使用できるようになったことは [2021年12月にリリースされた1.57.0](https://blog.rust-lang.org/2021/12/02/Rust-1.57.0.html#panic-in-const-contexts) 以降であることに留意したい。

コミット: 67bc2e9

## 05-accessor-signatures

それぞれのゲッターとセッターの値を適切なサイズにする。セッターは受け取ったものは `u64` にキャストし、ゲッターは `try_from().unwrap()` を使用する。

`enum B~N` に対応する型 `T` を持たせる。また、それぞれで `seq!` を逐一呼び出すのは大変なので、 型と範囲を渡して `seq!` を呼び出す宣言的マクロを実装する。

```rust
macro_rules! define_bit_enums {
    ($t: ty, $range: pat_param) => {
        seq!(N in $range {
            pub enum B~N {}

            impl Specifier for B~N {
                const BITS: usize = N;
                type T = $t;
            }
        });
    }
}

define_bit_enums!(u8, 0..=8);
define_bit_enums!(u16, 9..=16);
define_bit_enums!(u32, 17..=32);
define_bit_enums!(u64, 33..=64);
```

`let field_type = quote! { <#ty as Specifier>::T };` を追加し、仮で `u64` にしていた箇所を `#field_type` に置き換える。 セッターの最初とゲッターの最後にキャスト処理を追加する。

コミット: b3202c3

## 06-enums

フィールドを `bool` と `enum` で管理できるように変更する。u64 から相互に変換する処理を `Specifier` 内に追加する。

```rust
pub trait Specifier {
    const BITS: usize;
    type T;

    fn convert_to_u64(item: Self::T) -> u64;
    fn convert_from_u64(v: u64) -> Self::T;
}
```

`T` は ゲッターとセッターで指定する型に使われる。 `B~N` の場合は直接数値を指定していたが、 `bool` や `enum` の型であればそれをそのままやりとりしたいので、 `T` は `bool` や `enum` となる。

`bool` の場合は、 `Specifier` はこのようになる。

```rust
impl Specifier for bool {
    const BITS: usize = 1;
    type T = bool;

    fn convert_to_u64(item: Self::T) -> u64 {
        match item {
            true => 1,
            false => 0,
        }
    }

    fn convert_from_u64(v: u64) -> Self::T {
        match v {
            0 => false,
            1 => true,
            _ => panic!("unexpected value: {}", v),
        }
    }
}
```

各、 `B~N` の追加実装はこのようになる。

```rust
fn convert_to_u64(item: Self::T) -> u64 {
    u64::from(item)
}

fn convert_from_u64(v: u64) -> Self::T {
    Self::T::try_from(v).unwrap()
}
```

ゲッター, セッターの箇所は `<#ty as Specifier>::convert_to_u64(v)` のような形に置き換える。

今回のテストでは `enum` に判別式がついている前提でよいとあるので、以下のような形式で、 `match` 用のアームを作成する。

```rust
let match_arms = data_enum.variants.iter().map(|variant| {
    let discriminant = &variant.discriminant.as_ref().unwrap().1;
    let name = &variant.ident;
    quote! {
        #discriminant => #ident::#name
    }
});
```

`enum` の `Specifier` 実装は以下のようになる。

```rust
quote! {
    impl Specifier for #ident {
        const BITS:usize = #bn::BITS;
        type T = #ident;

        fn convert_to_u64(item: Self::T) -> u64 {
            item as u64
        }

        fn convert_from_u64(item: u64) -> Self::T {
            match item {
                #(#match_arms,)*
                _ => panic!("unexpected value: {}", item),
            }
        }
    }
}.into()
```

コミット: e0a011e

## 07-optional-discriminant

判別式が省略されていた場合に、コンパイル時に決定された数値で変換できるように変更する。

```rust
let match_arms = data_enum.variants.iter().map(|variant| {
    let name = &variant.ident;
    quote! {
        _ if #ident::#name as u64 == item => #ident::#name
    }
});
```

これは、 `TriggerMode` であれば以下のようになる。

```rust
#[derive(BitfieldSpecifier, Debug, PartialEq)]
pub enum TriggerMode {
    Edge = 0,
    Level = 1,
}

impl Specifier for TriggerMode {
    const BITS: usize = B1::BITS;
    type T = TriggerMode;
    fn convert_to_u64(item: Self::T) -> u64 {
        item as u64
    }
    fn convert_from_u64(item: u64) -> Self::T {
        match item {
            _ if TriggerMode::Edge as u64 == item => TriggerMode::Edge,
            _ if TriggerMode::Level as u64 == item => TriggerMode::Level,
            _ => {
                ::core::panicking::panic_fmt(
                    format_args!("unexpected value: {0}", item),
                );
            }
        }
    }
}
```

コミット: 85089a0

## 08-non-power-of-two

`enum` の要素数が 2 の n 乗でなければ、 `BitfieldSpecifier` 処理時でエラーを出すようにする。 `BitfieldSpecifier` の処理内に以下のコードを加えれば完了。

```rust
if !variants_number.is_power_of_two() {
    return syn::Error::new(Span::call_site(), "BitfieldSpecifier expected a number of variants which is a power of 2").into_compile_error().into();
}
```

コミット: 753d03e

## 09-variant-out-of-range

判別式が `BITS` で扱える範囲より大きかった場合にエラーを出力するようにする。 `const _: () = {}` を使用したアプローチを取る。
以下のように、各要素の中から最大値を取り出し、扱えなければ `panic!` を呼び出す。
```rust
const _: () = {
    let mut max = 0_usize;
    let a = [TriggerMode::Edge as usize, TriggerMode::Level as usize];
    let mut i = 0;
    while i < a.len() {
        max = if max < a[i] {
            a[i]
        } else {
            max
        };
        i+=1;
    }

    let bits: usize = 1 << TriggerMode::BITS;

    if max >= bits {
        panic!("max discriminant is out of range");
    }
};
```

`const` 内でフォーマットを使用できるようにする、 `const_format_args` は現在 [Nightly](https://doc.rust-lang.org/std/macro.const_format_args.html) のみで有効なので、今回はオーバーフローが生じたことのメッセージのみ表示する。

本来の想定であろう、 `trait` を用いた方法は [occar421氏のリポジトリ](https://github.com/occar421/my-proc-macro-workshop/blob/ef287ccb5e445989f29ceec55914839c6ee5140b/bitfield/impl/src/lib.rs#L180-L184) で確認できる。

コミット: 672e712

## 10-bits-attribute, 11-bits-attribute-wrong

`#[bits = 1]` のように、 `bits` で指定された値がある場合、指定された値と違っていた場合はエラーを出力するようにする。
値の確認は以下のコードで確認する。

```rust
const _: [(); #bit_attributes] = [(); <#ty as Specifier>::BITS];
```

リテラルの位置をエラー箇所にしたいため、 `quote_spanned!` を使用する。

コミット: 0ec837b

## 12-accessors-edge

ビットフィールドが各配列の要素を跨いでいるときのテストケース。本来仮実装でカバーするべきだったが、できていなかったので修正する。マクロに関しては、特筆する項目はなし。

コミット: dbdb5f1
