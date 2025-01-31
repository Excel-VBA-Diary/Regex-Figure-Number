# Regex-Figure-Number
図表番号の書式を統一するための正規表現 (Regular expressions to unify the format of figure numbers)  
初回投稿日：2025年1月30日  
最終更新日：2025年1月31日  

## 1. 概要
図表番号の付与基準には、JIS Z 8301やISO/IEC専門業務用指針（ISO/IEC Directives）などの国内規格、国際規格がありますが、一般に使用される図表番号には統一したルールがなく、現場の暗黙のルールに従っているのが現状です。図1、図2、図3、・・・といった単純な連番もあれば、図1-2、図3.4 などのように図表番号の区切り文字も様々です。とは言え、一つの文書の中では図表番号の書式を統一したいという要求が出てきます。  
新規に文書を作成する場合は図表番号の書式を統一できますが、既存の文書を編集したり、複数のメンバーで文書を作成する場合、図表番号が不統一になることがあります。そこで、ここでは 図1-2、図3.4、表5.6-7 の区切り文字を -（ハイフン）に統一し、 図1-2、図3-4、表5-6-7 に変換するための正規表現を考えてみます。 
<br>  
## 2. 図表番号と正規表現
ここでは、図n1、表n1-n2、図n1-n2-n3、表n1-n2-n3-n4 のような図表番号を考えます。n1、n2、n3、n4 は1から始まる整数とします。  
この図表番号に対応する正規表現は
```
[図表]\d+(-[d+)*
```
区切り文字に -（ハイフン）と .（ピリオド）の混在を許す正規表現は
```
[図表]\d+([.-]\d+)*
```
<br>  
## 3. 変換のための正規表現と置換文字
### 3.1 とりあえずやってみる

上記に示した正規表現関数にキャプチャグループを加えて単純に変換してみます。  
```
正規表現：([図表]\d+)([.-](\d+))*
置換文字：$1-$3
```
結果：
|　変換前|　変換後|
|----------|----------|
|図1にの気温5.1℃、図2の気温14.8℃|図1-にの気温5.1℃、図2-の気温14.8℃|
|表1.2の気温5.1℃、表2.2の気温14.8℃|表1-2の気温5.1℃、表2-2の気温14.8℃|
|図1.2-3の気温5.1℃、図2.2-3の気温14.8℃|図1-3の気温5.1℃、図2-3の気温14.8℃|
|表1.2-3.4の気温5.1℃、表2.2-3.4の気温14.8℃|表1-4の気温5.1℃、表2-4の気温14.8℃|  
  
これはうまくいきません。量指定子（*）の繰り返しにおいて、キャプチャグループは最後に一致した文字をキャプチャするだけで、途中で一致した文字を処理しないこと、「図1」のように区分けがない場合、余計な-（ハイフン）が付けられてしまことが原因です。  
<br>  
### 3.2 区分けに対応してキャプチャグループを残す
そこで、表1.2-3.4 のような４つの区分けに対応して、キャプチャグループを残して変換する正規表現を考えてみます。  
```
正規表現：([図表]\d+)([.-]+(\d+))?([.-]+(\d+))?([.-]+(\d+))?
置換文字：$1${2:+-$3}${4:+-$5}${6:+-$7}
```
結果：
|　変換前|　変換後|
|----------|----------|
|図1にの気温5.1℃、図2の気温14.8℃|図1にの気温5.1℃、図2の気温14.8℃|
|表1.2の気温5.1℃、表2.2の気温14.8℃|表1-2の気温5.1℃、表2-2の気温14.8℃|
|図1.2-3の気温5.1℃、図2.2-3の気温14.8℃|図1-2-3の気温5.1℃、図2-2-3の気温14.8℃|
|表1.2-3.4の気温5.1℃、表2.2-3.4の気温14.8℃|表1-2-3-4の気温5.1℃、表2-2-3-4の気温14.8℃|  
  
置換文字の ${\<n\>:+\<string1\>:\<string2\>} という構文を「条件付き置換」と呼びます。  
${\<n\>:+\<string1\>:\<string2\>} は、キャプチャグループ n が存在する場合（つまり、一致した場合）、string1 を使用し、存在しない場合には string2 を使用します。  
存在しない場合にヌルを使う場合は ${\<n>\:+\<string1\>} と省略表記ができます。   
この構文は、Excelの正規表現関数（PCRE2に準拠）、その他のPerl互換の正規表現（PCRE）やPerlやPythonの正規表現ライブラリなどでも類似の機能がサポートされています。  
  
なお、3.2の正規表現は、「図1--2」「表1..2..3」のような例外表記を「図1-2」「表1-2-3」に変換します。  
<br>  
### 3.3 後読みと先読みを使って区切り文字を統一する
「条件付き置換」はサポートしていない場合、「後読み」をサポートしている場合は「後読み」と「先読み」の両方を使って区切り文字を統一することができます。  
```
正規表現：(?<=[図表]\d{1,9}([.-]\d{1,9}){0,2})\.(?=\d)
置換文字：-
```
結果：
|　変換前|　変換後|
|----------|----------|
|図1にの気温5.1℃、図2の気温14.8℃|図1にの気温5.1℃、図2の気温14.8℃|
|表1.2の気温5.1℃、表2.2の気温14.8℃|表1-2の気温5.1℃、表2-2の気温14.8℃|
|図1.2-3の気温5.1℃、図2.2-3の気温14.8℃|図1-2-3の気温5.1℃、図2-2-3の気温14.8℃|
|表1.2-3.4の気温5.1℃、表2.2-3.4の気温14.8℃|表1-2-3-4の気温5.1℃、表2-2-3-4の気温14.8℃|  
  
「後読み」は量指定子に制限があります。  
```
(?<=[図表]\d+([.-]\d+)*
```
というような量指定子の可変長には対応していませんので、次のように繰り返しを制限して固定長にする必要があります。  
```
(?<=[図表]\d{1,9}([.-]\d{1,9}){0,2})
```
ここでは、数値は9桁まで（例：図123456789）、図表番号の区分けは４つまで（例：図1-2-3-4）に制限しています。  

なお、3.3の正規表現は、「図1--2」「表1..2..3」のような例外表記には対応していません。  
<br>  
## 4. 制限事項
ここで紹介する正規表現はサンプルです。変換対象の図表番号のバリエーションに合わせて調整してください。  
また、正規表現サンプルはExcelの正規表現関数（REGEXREPLACE関数）を使って検証しています。  
この正規表現が利用可能かは、正規表現のエンジンやバージョン、ツールや言語に依存しますので、使用する環境を確認してください。  
