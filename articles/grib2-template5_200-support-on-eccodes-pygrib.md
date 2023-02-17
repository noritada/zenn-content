---
title: "ecCodesに入ったGRIB2 Template 5.200/7.200サポートを用いてpygribで気象庁全国合成レーダーGPVを読む"
emoji: "🛰️"
type: "tech"
topics: ["気象データ", "GRIB", "GRIB2", "ecCodes", "pygrib"]
published: true
---

この記事では、ecCodesの開発最新版に入ったGRIB2 Template 5.200/7.200サポートを用いて、これまで簡単には読めなかった気象庁全国合成レーダーGPVをpygribで読んでみます。結論としては、気象庁定義のTemplate 4.50008の設定を自分で追加してあげる必要があります。また、Template 5.200のレベル値が `0` のケースの扱いも少し気になります。しかし、それ以外は気持ちよく使えることがわかりました。

# はじめに：用語とか概念とか

## GRIB、Template 5.200/7.200

気象業界には、数値予報や解析値などの格子点群の値からなる面的データ (GPV, grid point value) がたくさん存在しますが、それらを格納するデータフォーマットの1つとして、GRIBというバイナリフォーマットがあります。そのほかに、気象以外にも使われるデータフォーマットであるHDF5やNetCDFなども用いられます。

GRIBデータは `b"GRIB"` で始まり `b"7777"` で終わるメッセージから成り、現在の最新editionであるGRIB edition 2 (GRIB2) では、1つのメッセージに複数のレイヤーが含まれています^[仕様として複数のレイヤーを含めることが可能、という意味です。実際には、1つのメッセージに複数のレイヤーを含めて1つのデータとする場合と、1つのメッセージに1つのレイヤーしか含めず、複数のメッセージの集合としてデータを表現する場合が存在します。日本の気象庁は前者のスタイル、筆者の認識している範囲では、欧米のデータは後者のスタイルが多いように見えます。]。最も一般的な複数レイヤーの使い方を以下に紹介します（あくまで例であり、実際にはこれらの混合や、これら以外の使い方も存在します）。

- 同じ経緯度を持つ格子点群の2次元平面について、気温、湿度、風速（経度方向、緯度方向）などの気象要素を1つずつレイヤーとして含む
- 同じ経緯度を持つ格子点群の2次元平面について、平均海面、地上2m、地上10m、気圧1000hPa、975hPaなど、複数の高度レベルを1つずつレイヤーとして含む
- 同じ経緯度を持つ格子点群の2次元平面について、現在、1時間先、2時間先など、複数の予報時間を1つずつレイヤーとして含む

各メッセージは次のような構造となっています。

- **Section 0 (Indicator Section, 指示節)**
  `b"GRIB"`、GRIBバージョン、メッセージサイズなど。
- **Section 1 (Identification Section, 識別節)**
  発行機関、参照時刻などのデータソースに関する情報。
- **Section 2 (Local Use Section, 地域使用節)**
  各地域の機関で定義した情報。
- **Section 3 (Grid Definition Section, 格子系定義節)**
  格子系の定義。格子系の種類（正距円筒図法やランベルト正角円錐図法など）と、それぞれの格子系を具体的に定義するパラメータ群（四隅の経緯度、緯度方向や経度方向の格子点数など）。種類ごとにTemplateが存在。
- **Section 4 (Product Definition Section, プロダクト定義節)**
  含まれる値の意味に関する定義。例えば前述の気象要素、高度レベル、予報時間など。さまざまなパターンを網羅できるよう、Templateが存在。気象要素についてはテーブルでコードが（単位と一緒に）定義されており、そのコードで表現される。
- **Section 5 (Data Representaion Section, 資料表現節)**
  格子点値をどのような表現として含むかに関する定義。簡単に言えば、圧縮方法や、その方法での圧縮に使われるパラメータ群の値の定義。必要となるパラメータ群は圧縮方法ごとに異なるので、圧縮方法ごとのTemplateが存在。
- **Section 6 (Bit-map Section, ビットマップ節)**
  配列中の一連の欠損値を表すビットマップ。各格子点値が欠損値であるかどうかの情報を格子点値の配列から切り離してこのセクションに格納し、欠損値以外の格子点値のみをSection 5で定義した方法で圧縮してSection 7に格納する、という使い方をする^[ただしTemplate 5.200の場合、後述のとおり、欠損値をlevel 0としてSection 7に含める方法がとられている。]。
- **Section 7 (Data Section, 資料節)**
  Section 5で定義した方法で圧縮されたデータ本体。一部のパラメータはSection 5に含まれず、こちらに含まれる。基本的にSection 5のTemplateに一対一で対応するTemplateが存在する。
- **Section 8 (End Section, 終端節)**
  `b"7777"`。

上記説明に記載したとおり、GRIBには気象要素や高度レベルの情報も含まれており、単に一連の格子点値や経緯度の値をシリアライズしたものではない、という点が業界フォーマットらしい特徴です。また、さまざまな格子系、さまざまな気象要素や高度レベルや予報時間、さまざまな圧縮方法が利用可能で、そのため汎用のデータ処理系はそれらに対応する必要があります（特定のGRIB2を読めればよい処理系を作るだけならそこまで難しくありません）^[さまざまな国の気象機関から公開されているデータもあればクローズドなデータもあり、Templateやパラメータによって、使用しているデータを見つけやすい（動作確認しやすい）ものもあれば、あまり使われておらず見つけにくい（動作確認しにくい）ものもある、という点も、汎用の処理系を作る上での難しさです。]。

今回取り上げるTemplate 5.200/7.200は、それぞれSection 5とSection 7についてのTemplateです。

## ecCodes

GRIBフォーマットを扱うのに、アメリカ国立環境予測センター (National Centers for Environmental Prediction, NCEP)^[アメリカ海洋大気庁 (National Oceanic and Atmospheric Administration, NOAA) の1部局である国立気象局 (National Weather Service, NWS) に含まれます。] で開発、公開している wgrib2 というコマンドラインツールが昔からよく使われてきました。一方、最近ではヨーロッパ中期予報センター (European Centre for Medium-Range Weather Forecasts, ECMWF) の提供、公開する [ecCodes](https://github.com/ecmwf/eccodes) というライブラリ、およびそれに付属するコマンドラインツール群もオープンソースで公開され、よく使われています。ecCodesは2016年10月にbeta版の終えてv2.0.0としてリリースされてから、順調に[リリースを重ねています](https://confluence.ecmwf.int/display/ECC/History+of+Changes)。

そのecCodesですが、日本の気象庁の配信するGRIB2データは扱えないことが多いのが、日本の気象データをよく扱う者としては辛いところでした。これは、気象庁のGRIB2の多く、例えば全国合成レーダーGPVなどは、GRIB2のSection 5およびSection 7にTemplate 5.200、7.200 (run length packing with level values packing^[値とその値がいくつ連続するかで符号化するアルゴリズムです。]) を用いており、これがecCodesでサポートされていないためでした。なお、全球数値予報モデル (GSM) GPVやメソ数値予報モデル (MSM) GPVなど、一部のデータはTemplate 5.0/7.0 (simple packing^[参照値からの差分を離散化したものを符号化するアルゴリズムです。]) を用いています。このsimple packingは世界的に見ると（ほかの圧縮方法との合わせ技も数に含めれば）ほとんどのデータで使われており、どんなGRIB処理ツールでも扱えます。

## pygrib

Pythonの科学技術計算界隈には、[pygrib](https://github.com/jswhit/pygrib) という、筆者もいくつかパッチを送って取り込んでもらっているPythonモジュールがあります。このライブラリは ecCodes の高レベルインタフェースとして作られており、Cython で書かれています。もともとは wgrib2 に使われている NCEP の grib_api というCライブラリもバックエンドとして使えましたが、2014 年にリリースされた v2.0 からはデフォルトが ecCodes に切り替わり、2020 年の v2.1 からは ecCodes のみをバックエンドとして使用可能になっています。

この pygrib は、GRIB 内のメッセージにアクセスし、そのメッセージ内のレイヤーの一連の格子点の経緯度や格子点値をNumPy配列として取り出せるため、データサイエンスで気象データを使う場合にはとても重宝します。ただ、これまでは下層である ecCodes で前述のとおり Template 5.200/7.200 が扱えませんでした。そのため、Pythonで日本の気象庁のGRIBデータを扱いたい場合、これまでは大抵は以下のような方法をとっていたと思います。

- wgrib2で目的のレイヤーをfloat32のフラットバイナリとして出力し、[`numpy.frombuffer`](https://numpy.org/devdocs/reference/generated/numpy.frombuffer.html)などを用いてNumPy配列として読み込む方法。検索すればごろごろ出てくる。
- wgrib2で目的のレイヤーをNetCDFフォーマットのデータとして出力し、読み込む方法。検索すればごろごろ出てくる。
- 自前でパースする方法。筆者は昔この方法をとっていたが、世の中にはたぶんほとんどいない。
  - 例：[GRIB2 ファイルを自力でパースしてみた [python]](https://qiita.com/y_k/items/5c3c61a1b6e3bcf53993)

# ecCodesでのTemplate 5.200/7.200サポート

1月下旬に、ecCodesにTemplate 5.200/7.200サポートが加わりました。

https://github.com/ecmwf/eccodes/pull/72

https://github.com/ecmwf/eccodes/commit/b5732d2b7133a624e8e78f16e99efd22550bd626

# ecCodes でデータを読む

まずは ecCodes で、Template 5.200/7.200 でエンコードされたデータにアクセスしてみます。

Homebrewなど一部のパッケージングシステムではリリース版がバイナリパッケージとして配布されているのですが、このTemplate 5.200/7.200サポートは入ったばかりで本記事執筆時点ではまだリリースされていませんので、開発最新版を用いて動作を確認してみましょう。

## 環境準備

ecCodes はシステムにもインストールされているので、それとは別に、今回の作業専用のディレクトリに開発版をインストールします。

以下のようなディレクトリ構成を前提とします（`${WORK_DIR}` が作業用ディレクトリです）。なお、説明は、筆者の使用する macOS 環境を用いたものとなっていますが、インストール後の使い方は OS によって大きく変わらないはずです。

- `${WORK_DIR}/build`
  ecCodesのビルド用ディレクトリ（書き込みあり）。
- `${WORK_DIR}/install`
  ecCodesのインストール用ディレクトリ（書き込みあり）。
- `${WORK_DIR}/Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin`
  竜巻発生確度ナウキャストデータ。[気象庁のサンプルデータページ](https://www.data.jma.go.jp/developer/gpv_sample.html)からダウンロードしたもの。
- `~/ghq/github.com/ecmwf/eccodes`
  https://github.com/ecmwf/eccodes のクローン。read-onlyな使用。
- `~/ghq/github.com/ecmwf/ecbuild`
  https://github.com/ecmwf/ecbuild のクローン。read-onlyな使用。

## ecCodes インストール

```shellsession
% # just show a commit hash of the current HEAD, which will be used to build
% ECCODES_SRC_DIR=~/ghq/github.com/ecmwf/eccodes
% (cd ${ECCODES_SRC_DIR} && git rev-parse --short HEAD)
83821a501

% # build and install eccodes from HEAD of the development repository
% mkdir build install
% cd build
% cmake ${ECCODES_SRC_DIR} -DCMAKE_INSTALL_PREFIX=${PWD}/../install
% make
% ctest
% make install
% cd ..
```

## Template 5.200/7.200サポートの簡単な確認

まずはTemplateの定義ファイルがインストールされていることを確認します。

```shellsession
% # check the existence of template 5.200 definitions
% ls install/share/eccodes/definitions/grib2/template.5.200.def
install/share/eccodes/definitions/grib2/template.5.200.def
```

続けて、ecCodesに含まれるコマンドラインツールの1つである `grib_dump` でダンプしてみます。全国合成レーダーデータでもよいのですが、まずは題材として、竜巻発生確度ナウキャストのデータを用います。

```shellsession
% # check the result of template 5.200 processing
% TNOWC_PATH=Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin
% ./install/bin/grib_dump -O -w count=1 ${TNOWC_PATH}
***** FILE: Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin
#==============   MESSAGE 1 ( length=1567 )                ==============
1-4       identifier = GRIB
5-6       reserved = MISSING
7         discipline = 0 [Meteorological products (grib2/tables/5/0.0.table) ]
8         editionNumber = 2
9-16      totalLength = 1567
======================   SECTION_1 ( length=21, padding=0 )    ======================
1-4       section1Length = 21
5         numberOfSection = 1
6-7       centre = 34 [Japanese Meteorological Agency - Tokyo (RSMC)  (common/c-11.table) ]
8-9       subCentre = 0
10        tablesVersion = 5 [Version implemented on 4 November 2009 (grib2/tables/1.0.table) ]
11        localTablesVersion = 1 [Unknown code table entry () ]
12        significanceOfReferenceTime = 0 [Analysis (grib2/tables/5/1.2.table) ]
13-14     year = 2016
15        month = 8
16        day = 22
17        hour = 2
18        minute = 0
19        second = 0
20        productionStatusOfProcessedData = 0 [Operational products (grib2/tables/5/1.3.table) ]
21        typeOfProcessedData = 2 [Analysis and forecast products (grib2/tables/5/1.4.table) ]
======================   SECTION_3 ( length=72, padding=0 )    ======================
1-4       section3Length = 72
5         numberOfSection = 3
6         sourceOfGridDefinition = 0 [Specified in Code table 3.1 (grib2/tables/5/3.0.table) ]
7-10      numberOfDataPoints = 86016
11        numberOfOctectsForNumberOfPoints = 0
12        interpretationOfNumberOfPoints = 0 [There is no appended list (grib2/tables/5/3.11.table) ]
13-14     gridDefinitionTemplateNumber = 0 [Latitude/longitude (Also called equidistant cylindrical, or Plate Carree)  (grib2/tables/5/3.1.table) ]
15        shapeOfTheEarth = 4 [Earth assumed oblate spheroid as defined in IAG-GRS80 model (major axis = 6,378,137.0 m, minor axis = 6,356,752.314 m, f = 1/298.257222101)  (grib2/tables/5/3.2.table) ]
16        scaleFactorOfRadiusOfSphericalEarth = MISSING
17-20     scaledValueOfRadiusOfSphericalEarth = MISSING
21        scaleFactorOfEarthMajorAxis = 1
22-25     scaledValueOfEarthMajorAxis = 63781370
26        scaleFactorOfEarthMinorAxis = 1
27-30     scaledValueOfEarthMinorAxis = 63567523
31-34     Ni = 256
35-38     Nj = 336
39-42     basicAngleOfTheInitialProductionDomain = 0
43-46     subdivisionsOfBasicAngle = MISSING
47-50     latitudeOfFirstGridPoint = 47958333
51-54     longitudeOfFirstGridPoint = 118062500
55        resolutionAndComponentFlags = 48 [00110000 (grib2/tables/5/3.3.table) ]
56-59     latitudeOfLastGridPoint = 20041667
60-63     longitudeOfLastGridPoint = 149937500
64-67     iDirectionIncrement = 125000
68-71     jDirectionIncrement = 83333
72        scanningMode = 0 [00000000 (grib2/tables/5/3.4.table) ]
======================   SECTION_4 ( length=34, padding=0 )    ======================
1-4       section4Length = 34
5         numberOfSection = 4
6-7       NV = 0
8-9       productDefinitionTemplateNumber = 0 [Analysis or forecast at a horizontal level or in a horizontal layer at a point in time (grib2/tables/5/4.0.table) ]
10        parameterCategory = 193 [Unknown code table entry (grib2/tables/5/4.1.0.table) ]
11        parameterNumber = 0 [Unknown code table entry () ]
12        typeOfGeneratingProcess = 0 [Analysis (grib2/tables/5/4.3.table) ]
13        backgroundProcess = 153
14        generatingProcessIdentifier = 255
15-16     hoursAfterDataCutoff = 0
17        minutesAfterDataCutoff = 0
18        indicatorOfUnitOfTimeRange = 0 [Minute (grib2/tables/5/4.4.table) ]
19-22     forecastTime = 0
23        typeOfFirstFixedSurface = 1 [Ground or water surface (grib2/tables/5/4.5.table) ]
24        scaleFactorOfFirstFixedSurface = MISSING
25-28     scaledValueOfFirstFixedSurface = MISSING
29        typeOfSecondFixedSurface = 255 [Missing (grib2/tables/5/4.5.table) ]
30        scaleFactorOfSecondFixedSurface = MISSING
31-34     scaledValueOfSecondFixedSurface = MISSING
======================   SECTION_5 ( length=23, padding=0 )    ======================
1-4       section5Length = 23
5         numberOfSection = 5
6-9       numberOfValues = 86016
10-11     dataRepresentationTemplateNumber = 200 [Unknown code table entry (grib2/tables/5/5.0.table) ]
12        bitsPerValue = 8
13-14     maxLevelValue = 3
15-16     numberOfLevelValues = 3
17        decimalScaleFactor = 0
18-19     levelValues = 1
20-21     levelValues = 2
22-23     levelValues = 3
======================   SECTION_6 ( length=6, padding=0 )     ======================
1-4       section6Length = 6
5         numberOfSection = 6
6         bitMapIndicator = 255 [A bit map does not apply to this product (grib2/tables/5/6.0.table) ]
======================   SECTION_7 ( length=1391, padding=0 )   ======================
1-4       section7Length = 1391
5         numberOfSection = 7
6-1391    codedValues = (86016,1386) {
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03,
9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03, 9.9990000000e+03
... 85916 more values
} # data_run_length_packing codedValues
======================   SECTION_8 ( length=4, padding=0 )     ======================
1-4       7777 = 7777
```

注目すべきは、Section 5の内容です。`dataRepresentationTemplateNumber = 200` の下に、Templateに含まれる各種パラメータの値がきちんと表示されており、[仕様に関する気象庁資料](https://www.data.jma.go.jp/suishin/cgi-bin/catalogue/make_product_page.cgi?id=Nowcast)と整合的なので、きちんと読めていることがわかります。`Unknown code table entry (grib2/tables/5/5.0.table)` と、`Unknown` となっているのが気になった方は、[おまけ](#おまけ：grib2のバージョンとコード)を参照してください。

```
10-11     dataRepresentationTemplateNumber = 200 [Unknown code table entry (grib2/tables/5/5.0.table) ]
12        bitsPerValue = 8
13-14     maxLevelValue = 3
15-16     numberOfLevelValues = 3
17        decimalScaleFactor = 0
18-19     levelValues = 1
20-21     levelValues = 2
22-23     levelValues = 3
```

また、デコードで得られた配列が（冒頭部分のみ）Section 7 に表示されており、デコードできていそうだというのもわかります。

なお、このTemplate 5.200/7.200サポートが入るまでは、以下のようにTemplate 5.200のダンプ以降がエラーになっていました。

```shellsession
% grib_dump -O -w count=1 ${TNOWC_PATH}
***** FILE: Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin 
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
#==============   MESSAGE 1 ( length=1567 )                ==============
1-4       identifier = GRIB
5-6       reserved = MISSING
7         discipline = 0 [Meteorological products (grib2/tables/5/0.0.table) ]
8         editionNumber = 2
9-16      totalLength = 1567
======================   SECTION_1 ( length=21, padding=0 )    ======================
1-4       section1Length = 21
5         numberOfSection = 1
6-7       centre = 34 [Japanese Meteorological Agency - Tokyo (RSMC)  (common/c-11.table) ]
8-9       subCentre = 0
10        tablesVersion = 5 [Version implemented on 4 November 2009 (grib2/tables/1.0.table) ]
11        localTablesVersion = 1 [Unknown code table entry () ]
12        significanceOfReferenceTime = 0 [Analysis (grib2/tables/5/1.2.table) ]
13-14     year = 2016
15        month = 8
16        day = 22
17        hour = 2
18        minute = 0
19        second = 0
20        productionStatusOfProcessedData = 0 [Operational products (grib2/tables/5/1.3.table) ]
21        typeOfProcessedData = 2 [Analysis and forecast products (grib2/tables/5/1.4.table) ]
======================   SECTION_3 ( length=72, padding=0 )    ======================
1-4       section3Length = 72
5         numberOfSection = 3
6         sourceOfGridDefinition = 0 [Specified in Code table 3.1 (grib2/tables/5/3.0.table) ]
7-10      numberOfDataPoints = 86016
11        numberOfOctectsForNumberOfPoints = 0
12        interpretationOfNumberOfPoints = 0 [There is no appended list (grib2/tables/5/3.11.table) ]
13-14     gridDefinitionTemplateNumber = 0 [Latitude/longitude (Also called equidistant cylindrical, or Plate Carree)  (grib2/tables/5/3.1.table) ]
15        shapeOfTheEarth = 4 [Earth assumed oblate spheroid as defined in IAG-GRS80 model (major axis = 6,378,137.0 m, minor axis = 6,356,752.314 m, f = 1/298.257222101)  (grib2/tables/5/3.2.table) ]
16        scaleFactorOfRadiusOfSphericalEarth = MISSING
17-20     scaledValueOfRadiusOfSphericalEarth = MISSING
21        scaleFactorOfEarthMajorAxis = 1
22-25     scaledValueOfEarthMajorAxis = 63781370
26        scaleFactorOfEarthMinorAxis = 1
27-30     scaledValueOfEarthMinorAxis = 63567523
31-34     Ni = 256
35-38     Nj = 336
39-42     basicAngleOfTheInitialProductionDomain = 0
43-46     subdivisionsOfBasicAngle = MISSING
47-50     latitudeOfFirstGridPoint = 47958333
51-54     longitudeOfFirstGridPoint = 118062500
55        resolutionAndComponentFlags = 48 [00110000 (grib2/tables/5/3.3.table) ]
56-59     latitudeOfLastGridPoint = 20041667
60-63     longitudeOfLastGridPoint = 149937500
64-67     iDirectionIncrement = 125000
68-71     jDirectionIncrement = 83333
72        scanningMode = 0 [00000000 (grib2/tables/5/3.4.table) ]
======================   SECTION_4 ( length=34, padding=0 )    ======================
1-4       section4Length = 34
5         numberOfSection = 4
6-7       NV = 0
8-9       productDefinitionTemplateNumber = 0 [Analysis or forecast at a horizontal level or in a horizontal layer at a point in time (grib2/tables/5/4.0.table) ]
10        parameterCategory = 193 [Unknown code table entry (grib2/tables/5/4.1.0.table) ]
11        parameterNumber = 0 [Unknown code table entry () ]
12        typeOfGeneratingProcess = 0 [Analysis (grib2/tables/5/4.3.table) ]
13        backgroundProcess = 153
14        generatingProcessIdentifier = 255
15-16     hoursAfterDataCutoff = 0
17        minutesAfterDataCutoff = 0
18        indicatorOfUnitOfTimeRange = 0 [Minute (grib2/tables/5/4.4.table) ]
19-22     forecastTime = 0
23        typeOfFirstFixedSurface = 1 [Ground or water surface (grib2/tables/5/4.5.table) ]
24        scaleFactorOfFirstFixedSurface = MISSING
25-28     scaledValueOfFirstFixedSurface = MISSING
29        typeOfSecondFixedSurface = 255 [Missing (grib2/tables/5/4.5.table) ]
30        scaleFactorOfSecondFixedSurface = MISSING
31-34     scaledValueOfSecondFixedSurface = MISSING
======================   SECTION_5 ( length=23, padding=12 )   ======================
1-4       section5Length = 23
5         numberOfSection = 5
6-9       numberOfValues = 86016
10-11     dataRepresentationTemplateNumber = 200 [Unknown code table entry (grib2/tables/5/5.0.table) ]
```

## デコードで得られた値の確認

もっと詳しく、`grib_dump` と同様のコマンドラインツールである `grib_get_data` で、各格子点値を対応する経緯度と併せて表示してみます。比較のため、筆者がRustで開発している [grib-rs](https://github.com/noritada/grib-rs) というGRIBパーサライブラリの gribber というコマンドラインツールの出力と比べてみます。

なお、`grib_get_data` はデフォルトでは欠損値を飛ばして表示するため、`-m nan` で、欠損値を `nan` として表示するようにしています。

まずは先頭4点のデータです。

```shellsession
% # compare values of data decoded from the first submessage using eccodes with those of data decoded using gribber from grib-rs, which I am developing, and wgrib2
% ./install/bin/grib_get_data -w count=1 -m nan ${TNOWC_PATH} | head -5
Latitude Longitude Value
   47.958  118.062 nan
   47.958  118.188 nan
   47.958  118.312 nan
   47.958  118.438 nan
% gribber decode ${TNOWC_PATH} 0.0 | head -5
 Latitude Longitude     Value
47.958332 118.06249       NaN
47.958332 118.18749       NaN
47.958332 118.31249       NaN
47.958332 118.43749       NaN
% wgrib2 -d 1.1 -order raw -no_header -text - Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin | head -5
1.1:09.999e+20
9.999e+20
9.999e+20
9.999e+20
9.999e+20
```

続けて、値の頻度の比較です。

```shellsession
% # compare value counts of data decoded from the first submessage using eccodes with those of data decoded using gribber
% ./install/bin/grib_get_data -w count=1 -m nan ${TNOWC_PATH} | awk '{print $3;}' | sort | uniq -c | sort -nr
71493 nan
14383 1.0000000000e+00
  76 3.0000000000e+00
  64 2.0000000000e+00
   1 Value
% gribber decode ${TNOWC_PATH} 0.0 | awk '{print $3;}' | sort | uniq -c | sort -nr
71493 NaN
14383 1
  76 3
  64 2
   1 Value
```

基本的に同じデコード結果になっていそうです。

（本記事を最初に書いた段階では、欠損値がすべて `0` という値になってしまう問題がありましたが、修正されました。問題の原因であるレベル値 `0` という概念も含め、[おまけ](#おまけ：レベル値0に対応する代表値が欠損値ではなく0となっていた問題)として記載しました。）

## レイヤー一覧の確認

値のデコードに直接は影響はありませんが、データに含まれるレイヤーの一覧で圧縮方法の情報も表示されるので、一応確認しておきましょう。レイヤーの一覧を見るには `grib_ls` コマンドを使います^[表示される項目に違いはありますが、wgrib2の `wgrib2 ${input_file_name}` や `wgrib2 -V ${input_file_name}`、gribber の `gribber list ${input_file_name}` に対応する操作になります。]。

```shellsession
% # check the submessage listing
% ./install/bin/grib_ls ${TNOWC_PATH}
Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin
edition      centre       date         dataType     gridType     stepRange    typeOfLevel  level        shortName    packingType
2            rjtd         20160822     af           regular_ll   0            surface      0            unknown      grid_run_length
2            rjtd         20160822     af           regular_ll   10           surface      0            unknown      grid_run_length
2            rjtd         20160822     af           regular_ll   20           surface      0            unknown      grid_run_length
2            rjtd         20160822     af           regular_ll   30           surface      0            unknown      grid_run_length
2            rjtd         20160822     af           regular_ll   40           surface      0            unknown      grid_run_length
2            rjtd         20160822     af           regular_ll   50           surface      0            unknown      grid_run_length
2            rjtd         20160822     af           regular_ll   1            surface      0            unknown      grid_run_length
7 of 7 messages in Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin

7 of 7 total messages in 1 files
```

圧縮方法を表す `packingType` が `grid_run_length` となっており、Template 5.200、7.200 (run length packing with level values packing) であることがわかります。

（本記事を最初に書いた段階では、`unknown` になっていましたが、修正されました。[おまけ](#おまけ：packingtypeがunknownとなっていた問題)として記載しました。）

# pygrib でデータを読む

続いて pygrib でデータにアクセスしてみます。

## 環境準備

ecCodesの場合と同様、pygribについても今回の作業専用のディレクトリに開発版をインストールします。単なる筆者の好みで [poetry](https://python-poetry.org/) を用いて Python の仮想環境を構築していますが、ほかのツールでも問題ないでしょう。

以下のようなディレクトリ構成を前提とします（`${WORK_DIR}` が作業用ディレクトリです）。

- `${WORK_DIR}/pygrib`
  https://github.com/jswhit/pygrib のクローン。ビルドにも使用（書き込みあり）。
- `${WORK_DIR}/install`
  これまでの作業でecCodesをインストールした用ディレクトリ（読み取りのみ）。
- `${WORK_DIR}/Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin`
  竜巻発生確度ナウキャストGPV。
- `${WORK_DIR}/Z__C_RJTD_20230101000000_RDR_JMAGPV_Ggis1km_Prr10lv_ANAL_grib2.bin`
  全国合成レーダーGPV。[京都大学生存圏研究所の生存圏データベース](http://database.rish.kyoto-u.ac.jp/)からダウンロードしたもの。
- （poetry の仮想環境）

## pygrib インストール

```shellsession
% # install pygrib which uses eccodes installed above
% poetry init
% poetry add numpy pyproj cython
% cd pygrib
% mv pyproject.toml pyproject.toml.tmp  # use env associated with the parent directory
% ECCODES_DIR=../install poetry run python setup.py install
% cd ..
```

実は、上記作業でインストールしたところ、`${WORK_DIR}/install` 内の ecCodes を用いてビルドしたにも関わらず、インストールして実行するとシステムの ecCodes を参照してしまう問題が発生しました。環境に依存する問題で、本筋ともずれますので、この問題については一旦スキップし、[おまけ](#おまけ：pygribをインストールして実行するとシステムのeccodesを参照してしまう問題)として記載するのみとします。

## 竜巻発生確度ナウキャストGPVを用いたTemplate 5.200/7.200サポートの確認

竜巻発生確度ナウキャストGPVの格子点値（と経緯度）にアクセスしてみましょう。

```python
>>> import pygrib
>>> grib = pygrib.open("Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin")
>>> message = grib.message(1)
>>> message.latlons()
(array([[47.958333, 47.958333, 47.958333, ..., 47.958333, 47.958333,
        47.958333],
       [47.875   , 47.875   , 47.875   , ..., 47.875   , 47.875   ,
        47.875   ],
       [47.791667, 47.791667, 47.791667, ..., 47.791667, 47.791667,
        47.791667],
       ...,
       [20.208444, 20.208444, 20.208444, ..., 20.208444, 20.208444,
        20.208444],
       [20.125111, 20.125111, 20.125111, ..., 20.125111, 20.125111,
        20.125111],
       [20.041667, 20.041667, 20.041667, ..., 20.041667, 20.041667,
        20.041667]]), array([[118.0625, 118.1875, 118.3125, ..., 149.6875, 149.8125, 149.9375],
       [118.0625, 118.1875, 118.3125, ..., 149.6875, 149.8125, 149.9375],
       [118.0625, 118.1875, 118.3125, ..., 149.6875, 149.8125, 149.9375],
       ...,
       [118.0625, 118.1875, 118.3125, ..., 149.6875, 149.8125, 149.9375],
       [118.0625, 118.1875, 118.3125, ..., 149.6875, 149.8125, 149.9375],
       [118.0625, 118.1875, 118.3125, ..., 149.6875, 149.8125, 149.9375]]))
>>> message.values
masked_array(
  data=[[--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --],
        ...,
        [--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --]],
  mask=[[ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True],
        ...,
        [ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True]],
  fill_value=9999.0)
```

問題なくNumPy配列として読めています。

なお、このTemplate 5.200/7.200サポートが入るまでは、以下のようなエラーになっていました。

```python
>>> grib = pygrib.open("Z__C_RJTD_20160822020000_NOWC_GPV_Ggis10km_Pphw10_FH0000-0100_grib2.bin")
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template dataRepresentation from grib2/template.5.200.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
<pygrib._pygrib.open object at 0x113d9f100>
```

## 全国合成レーダーGPVの格子点値へのアクセス

続けて、全国合成レーダーGPVの格子点値にアクセスしてみましょう。

```python
>>> import pygrib
>>> grib = pygrib.open("Z__C_RJTD_20230101000000_RDR_JMAGPV_Ggis1km_Prr10lv_ANAL_grib2.bin")
ECCODES ERROR   :  Unable to find template productDefinition from grib2/template.4.50008.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template productDefinition from grib2/template.4.50008.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
>>> message = grib.message(1)
ECCODES ERROR   :  Unable to find template productDefinition from grib2/template.4.50008.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
ECCODES ERROR   :  Unable to find template productDefinition from grib2/template.4.50008.def 
ECCODES ERROR   :  grib_handle_new_from_message: No final 7777 in message!
>>> message.latlons()
ECCODES ERROR   :  latitudes: Unable to get size of values
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "src/pygrib/_pygrib.pyx", line 1539, in pygrib._pygrib.gribmessage.latlons
  File "src/pygrib/_pygrib.pyx", line 1173, in pygrib._pygrib.gribmessage.__getitem__
RuntimeError: Key/value not found
>>> message.values
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "src/pygrib/_pygrib.pyx", line 803, in pygrib._pygrib.gribmessage.__getattr__
  File "src/pygrib/_pygrib.pyx", line 1173, in pygrib._pygrib.gribmessage.__getitem__
RuntimeError: Key/value not found
```

エラーとなりました。Template 4.32768〜4.65534は各地域の機関での使用のために予約されており（"Reserved for Local Use" と定義されている）、全国合成レーダーGPVでは気象庁定義のTemplate 4.50008を使用しているためです^[Section 4なので、Section 3を用いた経緯度算出やSection 5/6/7を用いたデコードには影響しないのではと思って、本文のとおり `latlons()` や `values` を叩いてみたのですが、ダメでした。]。

幸い、ecCodesでは、ライブラリ側で用意していないTemplateについても、ユーザが自由に追加できるようになっています。Template 4.50008を追加してしまいましょう^[Template 4.Xは気象データの意味を定義するものなので、自分で追加すれば簡単に対応できます。しかし、新たな投影法を伴う格子系をTemplate 3.Xとして追加したり、新たな圧縮方法をTemplate 5.X/7.Xとして追加しても、実際のアルゴリズムのサポートが必要になるので、「Templateファイルの追加」だけでは解決できないことのほうが多いでしょう。]。

気象庁の[技術情報第162号 (PDF)](https://www.data.jma.go.jp/suishin/jyouhou/pdf/162.pdf) を眺めると、Template 4.50008は[Template 4.8](https://apps.ecmwf.int/codes/grib/format/grib2/templates/4/8)を拡張したものであることがわかるので、とりあえず次のようなファイルを作成しておけば問題ありません。

```:install/share/eccodes/definitions/grib2/template.4.50008.def
include "grib2/template.4.8.def"
```

改めて、全国合成レーダーGPVの格子点値にアクセスしてみましょう。

```python
>>> import pygrib
>>> grib = pygrib.open("Z__C_RJTD_20230101000000_RDR_JMAGPV_Ggis1km_Prr10lv_ANAL_grib2.bin")
>>> message = grib.message(1)
>>> message.latlons()
(array([[47.995833, 47.995833, 47.995833, ..., 47.995833, 47.995833,
        47.995833],
       [47.9875  , 47.9875  , 47.9875  , ..., 47.9875  , 47.9875  ,
        47.9875  ],
       [47.979167, 47.979167, 47.979167, ..., 47.979167, 47.979167,
        47.979167],
       ...,
       [20.021952, 20.021952, 20.021952, ..., 20.021952, 20.021952,
        20.021952],
       [20.013619, 20.013619, 20.013619, ..., 20.013619, 20.013619,
        20.013619],
       [20.004167, 20.004167, 20.004167, ..., 20.004167, 20.004167,
        20.004167]]), array([[118.00625, 118.01875, 118.03125, ..., 149.96875, 149.98125,
        149.99375],
       [118.00625, 118.01875, 118.03125, ..., 149.96875, 149.98125,
        149.99375],
       [118.00625, 118.01875, 118.03125, ..., 149.96875, 149.98125,
        149.99375],
       ...,
       [118.00625, 118.01875, 118.03125, ..., 149.96875, 149.98125,
        149.99375],
       [118.00625, 118.01875, 118.03125, ..., 149.96875, 149.98125,
        149.99375],
       [118.00625, 118.01875, 118.03125, ..., 149.96875, 149.98125,
        149.99375]]))
>>> values = message.values
>>> values
masked_array(
  data=[[--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --],
        ...,
        [--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --],
        [--, --, --, ..., --, --, --]],
  mask=[[ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True],
        ...,
        [ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True],
        [ True,  True,  True, ...,  True,  True,  True]],
  fill_value=9999.0)
>>> import numpy as np
>>> np.nanmax(values)
51.5
>>> np.unique(values, return_counts=True)
(masked_array(data=[0.0, 0.1, 0.25, 0.35000000000000003, 0.45, 0.55, 0.65,
                   0.75, 0.85, 0.9500000000000001, 1.05,
                   1.1500000000000001, 1.25, 1.35, 1.45, 1.55,
                   1.6500000000000001, 1.75, 1.85, 1.95, 2.13, 2.38, 2.63,
                   2.88, 3.13, 3.38, 3.63, 3.88, 4.13, 4.38, 4.63, 4.88,
                   5.25, 5.75, 6.25, 6.75, 7.25, 7.75, 8.25, 8.75, 9.25,
                   9.75, 10.5, 11.5, 12.5, 13.5, 14.5, 15.5, 16.5, 17.5,
                   18.5, 19.5, 20.5, 21.5, 22.5, 23.5, 24.5, 25.5, 26.5,
                   28.5, 29.5, 30.5, 31.5, 32.5, 33.5, 34.5, 36.5, 37.5,
                   38.5, 39.5, 40.5, 41.5, 42.5, 43.5, 46.5, 48.5, 50.5,
                   51.5, --],
             mask=[False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False, False, False,
                   False, False, False, False, False, False,  True],
       fill_value=9999.0), array([2292219,    2130,    3075,    2950,    3160,    3501,    3179,
          4292,    4106,    3112,    2888,    2718,    2027,    2036,
          2051,    1441,    1398,    1530,    1125,    1023,    1985,
          1942,    1499,    1174,     827,     876,     654,     586,
           460,     393,     365,     286,     402,     320,     214,
           172,     158,     117,      90,      67,      90,      59,
            63,      72,      42,      38,      35,      22,      17,
            18,      16,      13,      18,      16,      16,       9,
            14,       2,       8,       9,       6,       5,       3,
             4,       1,       5,       4,       1,       1,       1,
             1,       1,       1,       2,       1,       1,       2,
             1, 6248434]))
```

無事にデータにアクセスでき、このGPVに含まれる降水強度の格子点値のうち、最も強いのは 51.5 mm/h（1格子点）、次が50.5 mm/h（2格子点）であることがわかりました^[なお、51.5 mm/hや50.5 mm/hのような中途半端な値になっているのは、代表値という特性によるものです。前述の技術情報第162号のレベル値の表を見るとわかるはずですが、「51.5 mm/h」は「51.0 mm/h以上52.0 mm/h未満」の意味であり、「50.5 mm/h」は「50.0 mm/h以上51.0 mm/h未満」の意味です。]。また、欠損値の格子点が 6,248,434 個あることもわかります。

# おまけ：GRIB2のバージョンとコード

本記事冒頭の GRIB2 の Section 4 の説明のところで、以下のように記載しました。

> 含まれる値の意味に関する定義。例えば前述の気象要素、高度レベル、予報時間など。さまざまなパターンを網羅できるよう、Templateが存在。気象要素についてはテーブルでコードが（単位と一緒に）定義されており、そのコードで表現される。

実はこういったコードはあらゆる Section で使われており、使える「コード」をまとめた表である Code Table が大量に存在します。そして、それらの Code Table は半年に一度のペースでアップデートされ、GRIB2 のバージョンとして管理されています。

本文で用いた竜巻発生確度ナウキャスト GPV の場合、ダンプの結果を見ると、基づいている GRIB2 のバージョンは 5 でした。

```
10        tablesVersion = 5 [Version implemented on 4 November 2009 (grib2/tables/1.0.table) ]
```

しかし、Section 5 のテンプレートの意味が記載されている Code Table 5.0 には、GRIB2 のバージョン 5 の時点では、実は、Template 5.200 に相当する 200 番のエントリがまだ存在しません。Code Table 5.0 に 200 番のエントリが入るのは、GRIB2 のバージョン 8 からになります。

ecCodes の場合、定義ファイルとして各バージョンの Code Table もインストールされるため、以下のように、GRIB2 バージョン 7 までの Code Table 5.0 には 200 番が存在せず、GRIB2 バージョン 8 の Code Table 5.0 には 200 番が存在することを確認できます。

https://github.com/ecmwf/eccodes/blob/83821a501b06ae849bf120ab6eaf1f904c24c84f/definitions/grib2/tables/7/5.0.table

https://github.com/ecmwf/eccodes/blob/83821a501b06ae849bf120ab6eaf1f904c24c84f/definitions/grib2/tables/8/5.0.table

したがって、「使用しているコードがまだ GRIB2 の Code Table に登録されていなかった」というのが、竜巻発生確度ナウキャスト GPV のダンプ結果において、`Unknown code table entry (grib2/tables/5/5.0.table)` となっていた理由です。

# おまけ：レベル値0に対応する代表値が欠損値ではなく0となっていた問題

本記事を書くにあたって試した最初のバージョン（ecCodes のリビジョン `f677e67b8`）では、`grib_get_data` の出力において、欠損値が `nan` ではなく `0.0000000000e+00` すなわち `0` として表示されてしまう問題がありました。

Template 5.200 では、もともとのデータ（データ代表値）がレベル値に変換され、その代表値とレベル値のマッピングが Section 5 に埋め込まれます。竜巻発生確度ナウキャストでは、レベル値 `1` = 代表値 `1`、レベル値 `2` = 代表値 `2`、レベル値 `3` = 代表値 `3` という単純なマッピングになっています^[マッピングについては、[「Template 5.200/7.200 サポートの簡単な確認」](#template-5.200%2F7.200サポートの簡単な確認)で出力した Template 5.200 のダンプ表示を見れば、埋め込まれているのがわかるでしょう。]。このとき、レベル値 `0` に対応する代表値は定義されていないため、欠損値（`NaN`）として表現するのが適切なのですが、当初の実装では、以下の PR で意図的に `0` にされていました。

https://github.com/ecmwf/eccodes/pull/73

これは、筆者が問題を報告するとともに、サンプルとなる JMA の GRIB2 データを先方に共有したところ、修正していただけました。

https://github.com/ecmwf/eccodes/commit/e85de787cea0284c1c344b4fac94e5b07bc004a6

https://github.com/ecmwf/eccodes/commit/83821a501b06ae849bf120ab6eaf1f904c24c84f

なお、`0` となっていても竜巻発生確度ナウキャスト GPV の場合は区別できるので問題ありませんが、全国合成レーダー GPV などでは問題が発生します。全国合成レーダー GPV の場合は、[仕様 (PDF)](https://www.data.jma.go.jp/suishin/jyouhou/pdf/162.pdf) を見るとわかるように、レベル値 `0` = 欠損値、レベル値 `1` = 代表値 `0`（降水強度 0 mm/h）というマッピングになっています。

# おまけ：packingTypeがunknownとなっていた問題

本記事を書くにあたって試した最初のバージョン（ecCodes のリビジョン `f677e67b8`）では、`grib_ls` の出力において、`packingType` が `unknown` となっていました。これは、次のように `install/share/eccodes/definitions/grib2/section.5.def` に Template 5.200 の記載がないのが原因なので、ここに `dataRepresentationTemplateNumber = 200;` のエントリを加えてあげれば、ユーザ側で簡単に解決できます。

https://github.com/ecmwf/eccodes/blob/0e6d6a138fb6ed1ea375363d8a6dfe32b87222ce/definitions/grib2/section.5.def#L22-L51

（上記はリポジトリのコードですが、対応するコードが `install/share/eccodes/definitions/grib2/section.5.def` にもあるはずです。）

私のほうで PR を送り、（マージされずにcloseされたかたちとなっていますが）マージされたようです。

https://github.com/ecmwf/eccodes/pull/74

https://github.com/ecmwf/eccodes/commit/a1d55b5b5c590248f8a580e61fa39f01e81b041b

# おまけ：pygribをインストールして実行するとシステムのecCodesを参照してしまう問題

`ECCODES_DIR=../install poetry run python setup.py install` として、`${WORK_DIR}/install` 内の ecCodes を指定してビルド、インストールしたにも関わらず、インストールして実行するとシステムの ecCodes を参照してしまう問題が発生しました。次のように、筆者の用いている macOS 環境においてビルドすると、インストールされた pygrib の .so から libeccodes.dylib への参照が `@rpath` になってしまうのが原因です。

```shellsession
% otool -L ~/Library/Caches/pypoetry/virtualenvs/eccodes-template5-200-pygrib-XxMU_syS-py3.10/lib/python3.10/site-packages/pygrib-2.1.4-py3.10-macosx-10.15-x86_64.egg/pygrib/_pygrib.cpython-310-darwin.so
/Users/noritada/Library/Caches/pypoetry/virtualenvs/eccodes-template5-200-pygrib-XxMU_syS-py3.10/lib/python3.10/site-packages/pygrib-2.1.4-py3.10-macosx-10.15-x86_64.egg/pygrib/_pygrib.cpython-310-darwin.so:
	@rpath/libeccodes.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
```

次のように、リンクを `${WORKDIR}/install/lib/libeccodes.dylib` への参照に置き換えてあげることで解決しました。

```shellsession
% install_name_tool -change @rpath/libeccodes.dylib ${WORKDIR}/install/lib/libeccodes.dylib src/pygrib/_pygrib.cpython-310-darwin.so
```

# 参考文献

- [WMO（世界気象機関）の格子データ形式GRIB2について](https://qiita.com/e_toyoda/items/ce7497e1a633b16f1ff1) …… 気象庁の豊田英司さんによるGRIB2の解説。歴史的な経緯なども書かれており、非常に詳しい。
