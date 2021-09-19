<!-- classes: title -->


Markdown を 黒塗りできる Monadic Parser

# Darkdown


@sadnessOjisan

---


# ⚠️この発表は政権批判の意図はなく、政治に対する何かしらの主張は含みません。

---

# About Me

* ついに就職した。
* Iris LT のことをクソアプリコンテストと思ってる

---

## 行政文書に Markdown !?

* https://twitter.com/hal_sk/status/1432849899553394691


---

## 僕も賛成

plain text は差分がわかりやすい

---

# ここでクイズです

---

# 行政文書に必要な機能はなんでしょうか？

---

# 正解は、■■■ です。

---

# 正解は、黒塗り です。

---

## 黒塗りは必要

* 都合の悪い情報を隠すため？
* 個人情報が含まれる
* 攻撃に使われる

---

# 黒塗りできるマークダウン言語を作ろう

---

## 政府のための要件を考える

* 対象はplain text
* 黒塗りできる文法を定義する
* 3rd party のライブラリを使ってはいけない

---

# Parser を自作しよう

---

## Parser Combinator VS Parser Generator

前述の縛りがあるので　Parser Combinator

---

## OCaml で実装する

* 演算子の定義ができる
* 組み込みの Result 型がある
  - parser の失敗を検知する必要がある
* 引数の適用がやりやすい
* 俺は Haskell が書けない

---

## Parser Combinator とは

パーサー を コンビネーション する

---

## Parser の定義

```ocaml
type input = { text : string; pos : int }


type parser_t = input -> input * ('a, string) result;
```

<br />

入力 を受け取り、消費しなかった入力と、パース結果を返す。
消費しなかった入力を返すのは、後続の parser に渡すため。

例: 最小限のパーサー

文字列の先頭一文字をパースする。

```ocaml
let get_char = function
  | [] -> None
  | c::cs -> Some (c, cs)
```

FYI: https://zehnpaard.hatenablog.com/entry/2019/07/05/090514

---

## コンビネーターを作る

パーサーが次の3つの性質を満たすようにする

* functor
* applicative fanctor
* monad
  - darkdown では実装しない。applicative のみで済むようにズルしてます。

---

## map

functor. 
パース結果の型を引数にとる関数を組み合わせられる。

```ocaml
let map (f : 'a -> 'b) (p : 'a parser) : 'b parser =
{
    run =
      (fun input ->
        match p.run input with
        | input', Ok x -> (input', Ok (f x))
        | input', Error error -> (input', Error error));
  }

let ( <$> ) = map
```

例: string を返す parser から、 AST に変換する parser を作り出せる

```ocaml
let h1_parse =
  map
    (fun x -> { kind = "h1"; content = x })
    (prefix "# " *> parse_while (fun x -> not (is_space x)))
```

---

## `*>`, `<*`

複数のパーサーの結果を組み合わせられる。連接コンビネーター (applicative) `<*>` から作ることが多いが、単体でも定義できる。

口が開いている方を捨てられる

```ocaml
let ( *> ) (p1 : 'a parser) (p2 : 'b parser) : 'b parser =
  {
    run =
      (fun input ->
        let input', result = p1.run input in
        match result with
        | Ok _ -> p2.run input'
        | Error e -> (input', Error e));
  }


let ( <* ) (p1 : 'a parser) (p2 : 'b parser) : 'a parser =
  {
    run =
      (fun input ->
        let input', result = p1.run input in
        match result with
        | Ok x -> (
            let input'', result' = p2.run input' in
            match result' with
            | Ok _ -> (input'', Ok x)
            | Error e -> (input'', Error e))
        | Error e -> (input', Error e));
  }
```

例: `[]` で囲まれた中身を取り出せる

```ocaml
let kakko_parse: string parser =
  prefix "[" *> parse_while (fun x -> x != ']') <* any_char
```

`[` の左と `]` の右を捨てている

---

## 選択パーサー

alternativeと呼ばれているもの。

複数のパーサーがあり、片方が失敗したらもう片方でパースできる。

```ocaml
let ( <|> ) (p1 : 'a parser) (p2 : 'a parser) : 'a parser =
  {
    run =
      (fun input ->
        let input', result = p1.run input in
        match result with
        | Ok x -> (input', Ok x)
        | Error _ -> p2.run input);
  }
```

---

# ではこれらを使って行政文書を黒塗りしよう

---

# LIVE

---

# おわりに

* 今回はMarkdown それ自体をパースするのは諦めているので簡単な実装
* ちゃんとしようとするとめちゃくちゃ大変なことになりそう. 引用ブロックやリストブロックが厄介すぎて投げた
* ちなみに Haskell だと Parsec というライブラリを使った例がゴロゴロ転がっています
* 完全な Markdown Parser を OCaml で作ることを最近の日課にしてるので、一緒に作ってくれる人がいましたら是非
