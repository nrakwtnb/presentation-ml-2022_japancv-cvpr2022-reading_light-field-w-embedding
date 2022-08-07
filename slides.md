---
theme: seriph
#theme: academic
#theme: light-icons
#theme: purplin
#theme: penguin
#theme: unicorn
#theme: light-icons
#class: 'text-center'
highlighter: prism
#highlighter: shiki
lineNumbers: false
---

## Learning Neural Light Fields with Ray-Space Embedding

### written by B. Attal, J. Huang, M. Zollhoefer, J. Kopf, C. Kim

<br><br>

### @ 第 11 回全日本コンピュータビジョン勉強会 CVPR 2022 読み会

<br>

## N. Watanabe.

<footer class="absolute bottom-0 left-0 right-0 p-3">
<div>
version: 1.0
<div></div>
暫定公開版：https://temporal-nw-slide-japancv-cvpr2022-reading-20220807.netlify.app/
<div></div>
引用元が明記されていない画像の著作権は、スライド作成者に帰属します。
</div>


</footer>

---

# 発表者のバックグラウンドについて

- epoch1: 物理学（場の理論・弦理論、やや数理物理寄り）を専攻（博士号取得まで）
  - 理論的な話も大好きですが、簡単な計算機実験もしていました
- epoch2: いわゆる受諾分析会社でデータサイエンティスト（未定義）として従事
- 技術領域は、画像認識、自然言語処理が主
    - ただ画像解析系はここ1年ほど離れています（最近の流れについていけていない）
    - Comptuer Graphicsもド素人
  - 小規模 R&D 寄りを担当する事が多め
    - 小規模だが、API 開発・運用設計まで行うため、論文調査のウェイトはかなり小さく、たくさん読んでるわけではない

<style>
li{
  line-height: 20px;
  margin-top: 3px;
  margin-bottom: 3px;
  padding-top: 4px;
  padding-bottom: 4px;
}
</style>

---

# 紹介論文概要

<div class="grid grid-cols-[70%,30%] gap-4"><div>

* 主著者がMetaへインターンに行った時の研究結果
* フォトリアルなView synthesisにおいて、Light Fieldを直接モデル化し、以下を達成（Light Field自体はView synthesisの標準的な手法の1つであるため、全く目新しくない）
  * **NeRF系に比べてレンダリングが高速** （精度は同程度）
  * MPI系(NeXなど)と比べてメモリ効率的な手法（精度はやはり同程度）
  * radiance fieldでは不十分な、屈折や反射などもよく表現できる（との主張）
* Light Fieldの汎化困難さに対し、以下2つの技法を提案し、その有効性を評価
  * Ray-Space Embeddingと呼ばれる埋め込みを、Positional Embeddingの前に挿入する事で、サンプル間の補間をしやすい構造へマッピング
  * 複雑なシーンについても、Subdivisionという空間分割を行う事で精度を担保
    * 分割数で、解像度（計算コスト）と精度のトレードオフを制御

</div><div>
  <video src="https://neural-light-fields.github.io/videos/shiny_cd_ours.mp4" controls loop></video>
  <video src="https://neural-light-fields.github.io/videos/shiny_lab_ours.mp4" controls loop></video>
</div></div>

---

# 選定理由？

#### 客観的理由

- NeRFとその派生研究が流行っている（Computer Graphics $\times$ 機械学習）
  - 免責：発表者は 3D Computer Graphics に詳しい訳ではないため（つい1 ヶ月前まで NeRF 原論文＋顔認識系の 3D 構成数本くらいしか読んだ事なかった）、やや理解が浅い箇所もあるかもしれません。気になった箇所は、気兼ねなくご指摘・ご質問ください。

- Light Field自体は、Novel View Synthesisに対し、NeRFと異なるアプローチをとっている

#### 主観的理由

- (Novel View Synthesisの文脈で) "Light Field" なる単語をたまに見かけるが、何なのか全く知らない
  - "Light", "Field"と物理学畑で育った人間に刺さるキーワード
  - 数学的観点からも興味深い

- 今回幾つか読んだ中で、初見では彼らの提案内容（のアイディア）をよく理解できなかった
  - せっかく自分なりに解読・再整理したので、発表します

<style>
li :not(p){
  font-size: 11pt;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
</style>

<!--
  - 別のLight Fieldを使った論文は、Best paper finalistであったが、こっちと迷ったが、
他にもLight Field関連　数本ある中で、最も Light Field についてその特性を考察しているように見えた
  - こっちはそのうち誰か発表しそうor解説書きそうだったので。
-->

---

# そもそもNovel View Synthesisとは何か？

<div class="grid grid-cols-[70%,30%] gap-4"><div>

Novel View Synthesisを訳すと「新規視点の光景を合成」となる。もう少し言葉を補ってやると、これは
- 3 次元世界における、特定の対象に関する撮影・視覚情報が与えられた時に、
  - 新しく指定した視点（Novel）からの
  - 見え方（View）を
  - 特定・合成（Synthesis）する

タスクである。特にここで考えている入出力としては、右図にも示したような

- 入力：幾つかの方向から捉えた既知の光景
- 出力：ある方向から捉えた場合の未知の光景

という形式である。

なお、古くから存在する定番のタスクだと思われるが、
最近のneural networkによるシーン表現の研究と、
2022年のNeRF論文によって、耳目を集めている。

</div><div>

<img class="h-100" src="/novel_view_synthesis.png" />

<font size="1pt">ドラムの各画像は、NeRFの公開データから拝借した。</font>

</div></div>

---

# 言葉の定義

- **『シーン(scene)』** : 3 次元世界・環境を表す情報の内、視覚情報のみに関わる状態を表す
  - 多くの場合、どんな色合い＆反射特性を持つオブジェクト（表面）が、空間中のどこに存在するか、で指定される
- **『放射輝度(radiance)』** : ある領域から各方向へ、光が放たれる際のその強度
  - 3次元空間内の位置に依存するだけでなく、方向・角度依存性も持つ事に注意
  - 特にNeRF-familyでは、RGBそれぞれの成分 + 物質密度の形に分解する
    - 波数依存性を考慮したものは、『分光放射輝度(spectral radiance)』と呼ばれるが、こちらに近い

<div class="grid grid-cols-[70%,30%] gap-4"><div>

- **『場(field)』** : 空間の各点に何かしらの（物理的な）値が付随している状態、あるいはその表現
- **『Radiance field』** : 放射輝度に値を取る場
  - 3次元空間の各点に対して付随するため、"物質"が存在しない位置でも値が定義されている
  - このradiance fieldから、様々な視点から見える画像を生成するプロセスは、 **『ボリュームレンダリング(Volume Rendering)』** と呼ばれている

</div><div>

<img class="w-62" src="/scene_and_radiance.png" />

<font size="1pt">レゴの画像は、NeRFの公開データから拝借した。</font>

</div></div>

---


# Radiance Field を用いたボリュームレンダリングの問題点

#### レンダリングプロセス

<div class="grid grid-cols-[70%,30%] gap-4"><div>

* 通常Radiance Fieldを用いたボリュームレンダリングに於いては、各視線上、放射輝度の値の分布が利用される
* 言い換えると、viewの1つのピクセルの色を決定するために、そのピクセルを通る視線上の放射輝度の値を、十分なサンプリング数で計算する必要がある

</div><div>

<center>
<img class="w-100" src="volume_rendering_process_w_radiance_field.png" />
<font size="2">論文[7]より借用</font>
</center>

</div></div>

<br>

<v-click>

#### 計算コストの見積もり

$H \times W$ピクセル程度の新規視点画像を合成する事を考える。
放射輝度分布を知るための、典型的な必要サンプリング数を$M$とおくと、計算コストのオーダーは$H \times W \times M$（要するに、3次元の解像度レベル）になる。

とくに$W \sim 300$、$H \sim 300$、$M \sim 128 \sim 10^2$とおくと、
$H \times W \times M \sim 10^7$の放射輝度を評価する必要がある

NeRF-family においては、radiance field をMLP (Multi-Layer Perceptron)でモデル化しているが、
1枚の画像生成のために、このMLPに対する$10^7$回のforwarding計算となる。

そのため、一般的に **radiance fieldを用いたレンダリングプロセスは高コスト** で、リアルタイム計算が難しい。

</v-click>

<!-->
 の radiance field を表す MLP を、回評価する必要がある
カメラ焦点を通る視線上、
リアルタイムに評価するのは到底厳しい
-->

---

# 解決のためのアイディアとLight Fieldの導入

- レンダリングにおいては、シーン中心から外れた、光線上に付随する放射輝度積算結果のみに興味がある。
- よって、シーンから出てくる最終的な放射輝度（光が進んでも値がほぼ変化しない）を直接評価する。

このような、光線毎に付随する積算した放射輝度を表現するものとして、 **『Light Field』** と呼ばれるものが知られている。

<br>

<div class="grid grid-cols-[50%,50%] gap-4"><div>

<table>
  <tr><th>

  </th><th>
    入力
  </th><th>
    出力
  </th></tr>
  
  <tr style="border: none;"><td>
    Radiance Field (NeRF)
  </td><td>
    シーン空間の位置と方向
  </td><td>
    各点に付随する放射輝度と密度
  </td></tr>

  <tr style="border: none;"><td>
    Light Field
  </td><td>
    光線
  </td><td>
    光線に付随する放射輝度
  </td></tr>
</table>

</div><div>

<center>
<img class="w-110" src="/radiance_field_vs_light_field.png" />
<font size="2">論文[7]より借用</font>
</center>

</div></div>

<!--
* レンダリングのために、光線上各点の放射輝度を評価して合算するのではなく、
-->

<style>
th {
  border-right: 1px solid black;
  border-left: 1px solid black;
  border-bottom: 1.5px solid black;
}
td {
  border-right: 1px solid black;
  border-left: 1px solid black;
}
table {
  font-size: 11px;
  border-top: 1px solid black;
  border-bottom: 1px solid black;
}
</style>

---

# Light Fieldに関する言葉の定義と座標系の導入

* 3次元空間における、光線(ray)がなす集合（以降 $\;\mathcal{L}\;$ とかく）を考え、 **『光線空間(ray space)』** と呼ぶ $^{*1}$
  - この$\mathcal{L}$上の、放射輝度（色のみ）に値を取る場が、**Light Field** である $^{*2}$
* 特にこれをNNでモデル化したものを、**『Neural Light Field (NLF)』** と呼ぶ。

<br>

<v-click>

<div class="grid grid-cols-[70%,30%] gap-4"><div>

#### Light Slab：2平行平面を用いた座標系について

右図のように平行な2平面($\;\small \pi_A$, $\small \pi_B$とよぶ)
を置き、それぞれに2次元座標系($\;\scriptsize (x_i, y_i)_{i=A,B}\;$)
を設定しておく。
そして各光線 $\;\gamma\;$ と、2平面の交点 ($\;\scriptsize P_A^\gamma, P_B^\gamma\;$)
を考え、
この2交点の各2次元座標の値を集めた、4個の数値
（$\;\scriptsize (x_A^\gamma, y_A^\gamma, x_B^\gamma, y_B^\gamma)\;$
where
$\scriptsize w^\gamma_i := w_i(P_i^\gamma)$
for $\scriptsize w\!=\!x,y$ and $\scriptsize i\!=\!A,B$）
を、光線座標（$\;\mathcal{L}\;$の局所座標）とする。

逆に、光線座標の値を与えると、3次元空間中に光線を1つ定める事が分かる。

</div><div>

<img class="w-58" src="/light_slab.png" />

</div></div>

<footer class="absolute bottom-0 left-0 right-0 p-3">
<hr>

1. 全ての光線がなす集合は、4 次元の"幾何学的な空間"とみなせ、これはGrassmann 多様体 $\small {\rm Gr}(2,\mathbb{R}^4)$と呼ばれる対象になる
  - 本対象論文含む正面に近いviewのみ考える状況（forward-facing）では、この多様体の部分多様体を考える事が多い
  - 別文献[7]では、6 次元の Euclide 空間に埋め込む事（$\rm Pl\"ucker$埋め込み）もあるが、本文献では参考追加実験で使われている

2. より数学的に表すと、写像$\small \rm Map(\mathcal{L}, \mathcal{C})$（ただし$\small \mathcal{C} = [0,1]^3$は色を表す）の元である。なおNeRFでは、$\small \rm Map(\mathbb{R}^3 \times S^2, \mathcal{C} \times \mathbb{R}_{\ge 0})$（targetの第2成分は密度を表す。また方位成分$\small S^2$は、逆カリー化してある。）となり、どちらもシーンが **Neural Field** として表現される。[6]

</footer>

</v-click>

<!--
大域的な 4 次元の座標は貼れないが、後述のように
-->

---

# 2次元シーン空間に対する光線空間

光線空間$\;\mathcal{L}\;$とその上の座標を理解するために、ここでは2次元の世界（以降、シーン空間 $\mathcal{S}$と呼ぶ）を考える。この時、$\;\mathcal{L}\;$は2次元の空間をなすが、シーン空間と光線空間の間に以下の対応が存在する：

<table width="300">

  <tr><td class="left">

2次元シーン空間 $\mathcal{S}$

  </td><td class="center">

$\leftrightarrow$

  </td><td class="right">

光線空間 $\mathcal{L}$

  </td></tr>



  <tr><td class="left">
1つの光線（＝直線）
  </td><td class="center">

$\leftrightarrow$

  </td><td class="right">
1点
  </td></tr>


  <tr><td class="left">

特定の点を通る光線全体の集合（下図中央）

  </td><td class="center">

$\leftrightarrow$

  </td><td class="right">

直線（下図右） $^{*1}$

  </td></tr>


  <tr><td class="left">

特定の点から出る光線に対し、その角度を変える

  </td><td class="center">

$\leftrightarrow$

  </td><td class="right">

直線上を動く $^{**2}$

  </td></tr>

</table>

<style>
table {
  border-top: 1px solid black;
  border-bottom: 1px solid black;
}
td.left {
  width: 40%;
  text-align: right;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
td.right {
  width: 30%;
  text-align: left;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
td.center {
  width: 5%;
  text-align: center;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
td {
  font-size: 13px;
  line-height: 18px;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
  border-right: 1px solid black;
  border-left: 1px solid black;
  border-top: 1px solid black;
  border-bottom: 1px solid black;
}
td p {
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
</style>

<!--
定義より、2次元シーン空間内の1つの光線（直線）は、光線空間内の1点に対応する。
また、特定の点を通る光線全体の集合（下図中央）は、
光線空間上の直線（下図右）に一致する。$^{*1}$
言い換えると、特定の点から出る光線に対し、その角度を変える事は、光線空間内の直線上を動く事に対応する。$^{**2}$
-->


<div class="grid grid-cols-[30%,35%,35%] gap-4"><div style="font-size: 11pt;">

<br>

##### 2平行"線"座標系

<br>

- light slabで導入した2平面は、2次元シーン空間 $\mathcal{S}$ だと、2直線になる
- 各slab直線に、s座標とu座標を導入
  - 光線空間 $\mathcal{L}$ の座標となる

</div><div>

<center>

  2次元シーン空間 $\mathcal{S}$
  <img class="w-90" src="/NLF_explanation_2D_scene.png" />

</center>

</div><div>

<center>

  対応する光線空間  $\mathcal{L}$ (2次元)

  <img class="w-60" src="/NLF_explanation_2D_ray_space.png" />
</center>

<style>
div center {
  font-size: 12px;
}
</style>

</div></div>

<footer class="absolute bottom-0 left-0 right-0 p-1">
<hr>

1. 正確にはlight slabが直交座標を張る場合。3次元シーン空間の場合は、平面となる。
2. light slab座標系は光線空間全体をカバーしない。実際、光線がlight slab平面・直線に平行な場合、その交点は無限遠に飛ぶ。

</footer>

---

# シーン空間と光線空間の対応関係（続き）

1つの点オブジェクトが存在するシーンを考え、そこから出ていく光線全体についてもう少し考察する。

<div v-click="1">

1. <font color="magenta">オブジェクトが、slab直線に対し平行に（図面上、水平に）移動する</font>
  - この時、光線空間内では直線も平行移動、すなわち、s切片が変わる

</div>
<div v-click="2">

2. <font color="deepskyblue">オブジェクトが、slab直線に対し垂直に（図面上、鉛直に）移動する</font>
  - この時、光線空間内では、ある点を中心に回転移動、すなわち、直線の傾きが変わる
    - 今回の例だと、オブジェクト移動で不変なray $B$ が回転の中心

</div>
<div v-click="3">

3. <font color="red">オブジェクトが、 s 座標に対応するslab直線上にのる</font>
  - 傾きが0、すなわち光線空間において、$u$ 軸に平行な直線となる

</div>
<div v-click="1">

  <div class="grid grid-cols-[55%,45%] gap-4"><div>

<center>
  <font size="2">2次元シーン空間</font>
  <img class="w-70" src="/NLF_relation_2D_scene.png" />
</center>

  </div><div>

<left>
  <font size="2">対応する光線空間(2次元)</font>
  <img class="w-60" src="/NLF_relation_2D_ray_space.png" />
</left>

  </div></div>

</div>

<style>
ol li{
  font-size: 12pt;
}

ul li{
  font-size: 11pt;
  margin-left: 30px;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 5px;
}
</style>

---

# ふと我に返る（安直なLight Fieldの学習は困難では？）

これまで見てきたように、Light Field は、光線を入力とし、付随する輝度を返す関数である。
これは視点情報（1点を通る光線の集まり）を入力とすると、対応するview画像（1点に集約していく輝度の集まり）を出力する、とも言えるため、実質的には

<v-click>

<center>

**view synthesis = Light Field の計算 = Neural Light Field (NLF) の評価**

</center>

と言える。

</v-click><v-click>

特に、未知の視点(novel view)に対して推論可能である事に対しては、

<center>

**novel view synthesis がうまくいく = NLF が汎化 = 既知のviewからの正しい補間が定められる**

</center>

となる。

</v-click><v-click>

しかし実際問題は、光線空間の背後に隠れてしまった3 次元世界の事前情報を直接活用できていない事もあり、うまい補間を定めるのは難しい、と考えられる。

※そもそもこれが上手く行くなら、(単純な view synthesis において) NeRF なんて複雑なものは要らず、直接解いてしまえばよかったはずである。

</v-click>

<!--
実質的に  を解く事と等価である。

training sample は入力空間の中で sparse であり、
novel view synthesis が達成できる事は、
入力データからのうまい interpolation を定めることと等価

当然これは難しい


NLF は カメラ情報を入力とし、対応する view 画像を出力としており、
view synthesis を直接 NN (MLP)で解こうとしている
-->

---

# 2 次元シーン空間における View synthesis

説明を簡単化するため、引き続き、2 次元シーン世界の view synthesis を考える。

<div class="grid grid-cols-[60%,40%] gap-4"><div>

### セットアップ
<br>

<v-click>

#### "2"次元シーン空間内のオブジェクトとカメラ配置

* 十分小さな点オブジェクトP, Qが与えられる
* カメラ視点(viewpoint)もA, Bの2つを考える

</v-click><v-click>
<br>

#### 光線

* 輝度が非自明な光線としては、A-P, A-Q, B-P, B-Qの4つが存在

</v-click>

</div><div>

<center>
2次元空間内の配置
<img class="w-80" src="/NLF_explanation_3Dspace.png" />
</center>

</div></div>

<!--
#### 2平行"線"座標系

* s座標とu座標を導入
* 光線のなす集合$\mathcal{L}$も2次元

<br>
-->

---

<!--
* シーン空間内での各光線は、光線空間内では、点として表現される
  * A-P, A-Q, B-P, B-Qの各光線に対応する点をそれぞれ可視化
  * 光線空間上の座標は、光線と2つの平行線との2交点の座標で与えられる
-->

このセットアップでは、シーン空間と光線空間の対応関係は以下のようになる。

* A-P, A-Q, B-P, B-Qの各光線に対応する点をそれぞれ可視化

<br><br>

<div class="grid grid-cols-[36%,31%,33%] gap-4"><div>

<center>
  2次元シーン空間内の配置
  <img class="w-75" src="/NLF_explanation_3Dspace.png" />
</center>

</div><div>

<center>
  対応する光線空間(2次元)
  <img class="w-75" src="/NLF_explanation_ray_space.png" />
</center>

</div><div>

<!-- [TODO]: vertical-align -->
オブジェクトを起点とする場合と同様、
カメラ視点から出る全ての視線（光線）の集合も、光線空間上の直線となる。
左図では、それを点線で示した。

</div></div>

<!--
ここで、以下の事実に注意する：

* オブジェクトPがs平面に一致する時（$s_{AP}=s_{BP}$）、Aと任意の視点を通る光線のs座標は共通
* すなわち、視点を変えるに伴い、光線空間上のある水平線上を動く
  + 他の組み合わせでも同様
-->

<!--
<div class="grid grid-cols-[60%,40%] gap-4"><div>

</div><div>

</div></div>

* 点オブジェクトがs平面上直上にある事は、任意の光線のs座標が一定値、すなわち水平線である事と対応する
* より一般にs平面から話していくと、光線空間内で傾きを持つ
  - 傾きはs平面とu平面の距離の比のみで定まる

言い換えると、同一点からの放射輝度を特定する事は、
深度推定する事に他ならない

深度推定 = 傾き推定

この事実は、後述するように、「光線空間の埋め込み」なるものを考える動機付けとなる
論文で導入

光線空間の位置によって，同一オブジェクト由来の直線の傾きが異なる
-->

---

<div class="grid grid-cols-[70%,30%] gap-4"><div>

* この時、既知の各光景（訓練サンプル）は、光線空間内の線分上の色分布に他ならない(下図左)
<div v-click="1">

* 一方で、新規の光景を推論する事は、
  1. 異なる視点間で、同一オブジェクト由来の輝度の対応付けを行い、
  2. 視点間の輝度を補間し、新たな線分上の色分布を推定する
  
  事に相等する(下図中央)
</div>

<style>
li p {
  margin-top: 5px;
  margin-bottom: 5px;
}
</style>

</div><div>

<center>
<font size="3" align="center">2次元シーン空間内の配置</font>
<img class="h-50" src="/NLF_explanation_3Dspace.png" />
</center>

</div></div>

<table style="border-collapse: collapse; border: none;">
  <tr style="border: none; padding-bottom: 0px;"><td>
    既知サンプルが与えられた時
    <br>
    （点線が既知）
  </td><td>
    <div v-click="1">
    未知サンプル（光線）の輝度を推定
    </div>
  </td><td>
    <div v-click="2">
    正解(ground truth)
    <br>
    （実線が正しい輝度に対応し、他は0）
    </div>
  </td></tr>
  <tr style="border: none;"><td>
    <img class="w-55" src="/NLF_explanation_training_data_on_ray_space.png" />
  </td><td>
    <div v-click="1">
    <img class="w-55" src="/NLF_explanation_inference_on_ray_space.png" />
    </div>
  </td><td>
    <div v-click="2">
    <img class="w-55" src="/NLF_explanation_ground_truth_on_ray_space.png" />
    </div>
  </td></tr>
</table>

<style>
td {
  font-size: 14px;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 2px;
}
td img {
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
</style>

<!--
なお、以下の事実に着目する：

* 同一オブジェクト由来の光線は、光線空間内で直線となる
* その傾きは、オブジェクトの深度（シーン空間内の鉛直方向に関する位置）で決まる
-->


---

# 再び3次元写実的な世界へ

先の説明を、より現実的なケースとの対応と共に整理すると、

<style>
th {
  border-right: 1px solid black;
  border-left: 1px solid black;
  border-bottom: 2px solid black;
}
td {
  border-right: 1px solid black;
  border-left: 1px solid black;
  padding-top: 10px;
  padding-bottom: 10px;
}
table {
  font-size: 12px;
  border-top: 1px solid black;
  border-bottom: 1px solid black;
}
</style>

<table rules="cols">
  <tr><th>
    セットアップ
  </th><th>
    2点のオブジェクトからなる2次元シーン空間
  </th><th>
    多種多様なオブジェクトが存在する3次元シーン空間
  </th></tr>

  <tr><td>
    光線空間の座標と次元
  </td><td>
    2つの1次元直線座標　→　2次元
  </td><td>
    2つの2次元平面座標　→　4次元
  </td></tr>

  <tr><td>
    ある点を通る光線全体
  </td><td>
    2次元光線空間上の1次元直線
  </td><td>
    4次元光線空間上の2次元平面
  </td></tr>

  <tr><td>
    光線空間上、同一オブジェクト由来の輝度の対応付け
  </td><td>
    組み合わせがほとんどないため、簡単
  </td><td>
    オブジェクトの表面上、似たような輝度が連続して続くため、対応付け困難
  </td></tr>

  <tr><td>
    光線空間上、同一オブジェクト由来の視点補間
  </td><td>
    対応づけした2点間を通る直線を特定するだけ（余次元1）
  </td><td>
    対応づけした複数点から平面を保管する必要がある（余次元2）
  </td></tr>
</table>

<v-click>

となるが、結局、「Neural Light Field (NLF)の汎化」は、

<center>

<div style="text-align: left; margin-left: 100px;">

4次元光線空間 $\mathcal{L}$ 中、<br>
幾つかの2次元平面上での色分布（＝画像）が（既知サンプルとして）与えられた際に、<br>
それらとは異なる新しい2次元平面上の色分布（＝新規視点の画像）を補間（＝生成）する

</div>

</center>

事に他ならない。しかし、(3次元世界由来である、という)事前知識なしにこれは難しい。

</v-click><v-click>

これを成功させるため、論文では、次に説明するような2つの技法を提案している。

</v-click>

---

# 提案技法1. Ray Space Embedding

<div class="grid grid-cols-[50%,50%] gap-4"><div>

- 目的
  - 入力情報の効率的な圧縮
  - 補間可能な潜在空間

- お気持ち・アイディア
  - positional encodingは、成分毎に適用されるため、光線空間上の座標軸方向は特別な意味を持つ。
    - 2つは光線方向・角度成分、残り2つはテキスチャ成分を検出する。
  - 光線空間の各点における、同一点の方位変化方向（傾き）を推定し、これを軸と平行になるよう補正したい。そのためには回転行列を推定して、作用させれば良いが、ここでは一般的化したアファイン変換を推定する。
    - 先に述べたように、これは深度推定に相当する。
  - 特に光線空間の各点で傾きは異なる（オブジェクトごとに深度が異なる）ため、アファイン変換は局所的（＝光線空間上の関数）。
  - 推定候補自体は複数用意するため、埋め込み次元は4ではなく、一般に$\;4n\;$と表される。光線座標系を複数設定している事に相等する。

</div><div>

- 定式化：$\scriptsize L(\mathbb{r}) = F_\theta( \gamma( A(\mathbb{r})\mathbb{r} + b(\mathbb{r} ) ) )$ where $\scriptsize (A,b) = E_\phi(\mathbb{r})$
  - $r \in \mathbb{R}^4$: 入力。光線空間の座標値
  - $E_\phi$: 局所アファイン変換を推定するネットワーク
    - $A(\mathbb{r}) \in \mathbb{R}^{N \times 4}$: 光線空間を$\;N=4n\;$次元潜在空間へ埋め込むための線形変換パート
    - $b(\mathbb{r}) \in \mathbb{R}^N$: 平行移動パート
  - $\gamma$: positional encoding
  - $F_\theta$: 潜在空間を入力とし、色（輝度）を決めるネットワーク (color networkなどと呼ばれている)

<br>
<img class="w-100" src="/local_affine_trans_based_embedding.png" />
<font size="1">対象論文[1]から引用</font>

</div></div>

<style>
ul p {
  padding-bottom: 0px;
  margin-bottom: 0px;
}
li li {
  font-size: 13px;
  line-height: 22px;
}
</style>

---

# 提案技法2. Subdivision

<div class="grid grid-cols-[70%,30%] gap-4"><div>

- 目的
  - 視点に関して疎な状況においても、補間をうまくやりたい

- お気持ち・アイディア
  - 空間を分割（ボクセル化）し、それぞれにLight stabとLocal Light Fieldを割り当てる
    - NeRFとNLFを組み合わせる事に相等し、ボクセルの数分だけ光線空間の座標系が追加される
    - 各ボクセルにおいては、対象オブジェクト表面はより平面に近づくので扱いやすい
    - 特にボクセル断面と合致すると、傾き推定(深度推定)が自明となる
  - ボクセル化に伴って光線も分割され、各分割区間で積算した放射輝度をLight Fieldで評価する
    - NeRF同様、密度(＝光の放射強度と減衰率)も推定するが、あくまで光線に付随する(積算)密度である事に注意
    - レンダリングについても、視線と交差する各ボクセルからの積算放射輝度を、減衰込みで加算して算出

- 定式化
  - $i$: ボクセルインデックス
  - $\mathbb{r}_i$: ボクセル$i$に付随する光線空間の座標
  - NNモデルの変更点：双方の入力にボクセルの位置情報の埋め込みが追加
    - 局所affine変換の推定：$\scriptsize E_\phi: (\mathbb{r}_i, \gamma(i)) \rightarrow (A_i, b_i)$
    - Color network：$\scriptsize F_\theta: (\gamma(A_i \mathbb{r}_i+b_i), \gamma(i)) \rightarrow (c_i, \alpha_i)$
      - 輝度に加え、密度$\;\alpha_i\:$の推定が加わる
  - 光線$r$に付随する輝度の最終的な計算式: $\scriptsize c = \sum_{i \in \mathcal{V}(r)} \left( \prod_{j\in \mathcal{V}(r) \cap j \lt i} (1-\alpha_j) \right) \alpha_i c_i$
    - NeRFのボリュームレンダリングとほぼ同じ

</div><div>

<!--
  - 各ボクセル、その付近に存在するオブジェクトの輝度を推定する
  - またボクセルの位置情報は、positional encodingで統合する

(厳密な割り当てルールは、あまり確認していません)
-->

<center>
  <font size="2">グローバルLight Slabと局所Light Slabの対応</font>
  <img class="w-55" src="/Fig4_reparametrization_for_local_field.png" />
  <font size="1">対象論文[1]から引用</font>
</center>
<br>
<center>
  <font size="2">subdivisionを取り入れた際のアーキテクチャ</font>
  <img class="w-60" src="/Fig5_local_LF_via_spatila_subdivision.png" />
  <font size="1">対象論文[1]から引用</font>
</center>
<br>
<center>
  <font size="1">参考：取り入れる前のアーキテクチャ</font>
  <img class="w-60" src="/local_affine_trans_based_embedding.png" />
  <font size="1">対象論文[1]から引用</font>
</center>

</div></div>

<style>
ul li p {
  padding-top: 0px;
  padding-bottom: 0px;
  margin-top: 0px;
  margin-bottom: 0px;
}
li li {
  font-size: 13px;
  line-height: 22px;
}
</style>

---

# 実験概要と定量比較

<div class="grid grid-cols-[50%,50%] gap-4"><div>

- データセット
  - Real Forward-Facingとその変種(Undistorted RFF)
    - （視点に関し）疎なサンプルを含むデータセット
  - Shiny (NeX論文で提案) 
    -  viewの方向・角度依存性を評価するため、反射や屈折のある画像を含むデータセット
  - Stanford Light Field
    - （視点に関し）密なサンプルを含むデータセット
- 評価指標
  - 精度：PSNR, SSIMに加え、近年のNVSでよく使われるLPIPSを採用（定義は割愛）
  - 推論速度としてFPS

</div><div>

- 比較結果
  - 精度：既存手法と遜色ないレベル
  - 推論速度：NeRFの5〜100倍、X-Fieldを除き、疎な補間で数倍程度、密な補間だと100倍近い改善

<br>
<div>
  <center>
    <img class="w-90" src="/quantitative_comparison_w_marker.png" />
  </center>
</div>

<div>
  <center>
    <font size="1pt">対象論文[1]から引用（したものにマーカ入れ）</font>
  </center>
</div>

</div></div>

---

# 定性比較：デモ動画

以下は、著者らのHPに掲載されている比較である。
著者らは違いを赤い枠で囲み強調しているが、正直、見た目に違いはほとんどない...

<center>
<table style="border-collapse: collapse; border: none;">

  <tr style="border: none;"><td>
   <video width="250" src="https://neural-light-fields.github.io/videos/stanford_tarot_small_gt.mp4" loop controls></video>
  </td><td>
   <video width="250" src="https://neural-light-fields.github.io/videos/stanford_tarot_small_nerf.mp4" loop controls></video>
  </td><td>
    <video width="250" src="https://neural-light-fields.github.io/videos/stanford_tarot_small_xfields.mp4" loop controls></video>
  </td><td>
    <video width="250" src="https://neural-light-fields.github.io/videos/stanford_tarot_small_nerf.mp4" loop controls></video>
  </td></tr>

  <tr style="border: none;"><td>
    Ground Truth<br>(Nearest)
  </td><td>
    NeRF
  </td><td>
    X-Field
  </td><td>
    NLF
  </td></tr>

</table>

<!-- [TODO] centering tha cells at the last row -->

taken from https://neural-light-fields.github.io/
</center>

---

# 定性比較：view-dependence

著者らは、本手法による視点依存性の改善を強調しており、以下のような結果を提示している。
実際、専攻研究よりも鮮明さはあるように見える。（が、view-dependenceの改善については、この可視化からは分からないように見える）

<table style="border-collapse: collapse; border: none;">
  <tr style="border: none;"><td>
    <img class="w-90" src="/comparison_on_view_dependence_no1.png" />
  </td><td>
    <img class="w-90" src="/comparison_on_view_dependence_no2.png" />
  </td></tr>

  <tr style="border: none;"><td>
    <img class="w-90" src="/comparison_on_view_dependence_no3.png" />
    <center>
      <font size="2">いずれも対象論文[1]より拝借</font>
    </center>
  </td><td>
<div>

* RFF: T-Rex, Flower
* Shiny: Food, CD
* Stanford: Treasure, Flowers

</div>
  </td></tr>
</table>

---

# Ablation study

<div class="grid grid-cols-[50%,50%] gap-4"><div>


<center>
  <font size="3">ray space embedding</font>
  <img class="w-95" src="/result_of_ablation_study_on_ray_embedding.png" />
</center>


<div class="grid grid-cols-[45%,55%] gap-4"><div>

- "Feature"は光線空間座標をMLPで直接埋め込む場合
- "Local affine", "Feature", "Not Used"の順に良い
  - 特に"Not Used"は、Standford データセット（サンプルが視点間に対し密）では大きな差が出ている

</div><div>
<img class="w-60" src="/comparison_among_architecture.png" />
</div></div>


</div><div>


<center>
  <font size="3">subdivision</font>
  <img class="w-95" src="/result_of_ablation_study_on_subdivision.png" />
</center>

- 推論速度（FPS）は、グリッド分割数（細かさ）に反比例する
  - ボクセルが粗いほど、推論速度が速い
  - 放射輝度サンプル数〜グリッド分割数になる事から推察可能
- 細かいほど精度が上がるが、16->32だと伸びは小さい
- 推論速度と精度はトレードオフ

</div></div>

<style>
li {
  font-size: 14px;
}
</style>

---

# 細かい補足

#### 訓練時間

|モデル|詳細|データセット|訓練時間|
|-|-|-|-|
|NLF | subdivisionあり($32^3$ resolution) | Stanford（視点に関し密なデータセット） | 約20時間 |
|NLF | subdivisionなし | RFF, Shiny（視点に関し疎なデータセット） | 約10時間 |
|NeRF| | 各データセット | 約18時間 |
|NeX | | 各データセット | 約36時間・2GPU |

<br>

<div class="grid grid-cols-[50%,50%] gap-4"><div>

#### アーキテクチャ

* 光線空間の埋め込み次元：$N=32$
* positional encoding $\gamma$
  - subdivided version: $L=8$
  - no subdivision: $L=10$
* 埋め込みモデル$E_\phi$・LFネットワーク$F_\theta$：共に 8 layer, 256 hidden unit MLPで、1つのskip connectionをもつ

</div><div>

#### 制限

* 360$^\circ$ シーンで同様の事をしようとすると難しい
  - Light slabは使えず、代わりに$\small \rm Pl\"ucker$座標を使用
  - この座標系では、決まった点を通る光線集合は平面にならないため
* 現状、subdivisionと組み合わせないと、視点について疎なケースでは十分な精度が出ないが、元の問題であった推論速度低下につながる

</div></div>

<style>
tr, td {
  font-size: 12px;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}

li {
  font-size: 14px;
}
</style>

<!--
  margin-top: 0px;
  height: 8px;
-->

---

# まとめに代えて

### 所感・感想

- Light Field単体でも工夫すれば、思いの外学習ができる事は意外だった
- ただしオリジナルのNeRFを中心に比較しており、精度面でも速度面でも進化していると思われるので、どの程度優位性があるかは不明
  - 例えば、FasterNeRF(文献[10])では精度を落とさず、200FPS程度まで高速化している、との記載がある
- また提案技法の内容も「お気持ち・アイディア」から実際の「定式化」まではややギャップがあるように思われる
  - アーキテクチャは複雑になるが、同じCVPR2022のLF Neural Rendering(文献[2])の方が、補間時に、同一オブジェクトの情報を参照するためにepipolar line/pointsを利用していて、自然に感じる
- 全体としてややネガティブな印象となってしまったが、Light Fieldの可能性を感じさせてくれる論文ではあった

<!--
- またsubdivisionは実質NeRFを組み合わせているが、複雑な
- 提案技法は、NLFの学習がうまくいかない要因を取り除いているように見えるが、疎な視点が与えられた状況下で、NLFの学習がうまくいく理由は
-->

<br>

### 疑問点

- local affine 変換の推定結果から、実際に深度の情報が取り出せるのか？
- sceneの照明条件変更などの拡張はできるのか？
- 事前に NeRF で学習を行い、推論はLFN に蒸留してから行う手法があっても良さそう
  - と思っていたら、実は文献[11] (R2L)がまさにその研究であった
- 3 次元情報の取り込みを、local affine 変換の推定と、subdivision で吸収しているが、より explicit に 3 次元情報を取り込めないか？
  - Light FieldがGrassmann 多様体上の関数である事をうまく使えないか？

<style>
li {
  font-size: 14px;
}
</style>

---

# 参考文献

1. B.Attal, J-B.Huang, M.Zollhöfer, J.Kopf, and C.Kim, "Learning Neural Light Fields with Ray-Space Embedding". CVPR2022 (**今回の論文**)

1. B.Mildenhall, P.P.Srinivasan, M.Tancik, J.T.Barron, R.Ramamoorthi, and R.Ng. "Nerf:Representing scenes as neural radiance fields for view synthesis". ECCV2020

1. M.Suhail, C.Esteves, L.Sigal, and A.Makadia, "Neural Light Field Rendering". CVPR2022
  (**Best paper finalist**)

1. F.Dellaert and L.Yen-Chen, "Neural Volume Rendering: NeRF And Beyond". 2101.05204
  (**NeRFのサーベイ**)

1. A.Tewari, J.Thies, B.Mildenhall, P.Srinivasan, E.Tretschk, et.al. "Advances in Neural Rendering". 2111.05849, State of the Art Report at EUROGRAPHICS 2022
  (**Neural renderingのレビュー**)

1. Y.Xie, T.Takikawa, S.Saito, O.Litany, S.Yan, et.al. "Neural Fields in Visual Computing and Beyond". 2111.11426
  (**Neural Fieldsのレビュー**)

1. V.Sitzmann, S.Rezchikov, W.T.Freeman, J.B.Tenenbaum, and F.Durand, "Light field networks: Neural scene representations with single-evaluation rendering". NeurIPS2021

1. M.Levoy and P.Hanrahan, "Light field rendering". In Proceedings of the 23rd annual conference on Computer graphics and interactive techniques, 1996

1. M.Tancik, P.P.Srinivasan, B.Mildenhall, S.Fridovich-Keil, and N.Raghavan el.al. "Fourier featureslet networks learn high frequency functions in low dimen-sional domains". NeurIPS2020

1. S.J.Garbin, M.Kowalski, M.Johnson, J.Shotton, and J.Valentin, "Fastnerf: High-fidelity neuralrendering at 200fps". ICCV2021

1. H.Wang, J.Ren, Z.Huang, K.Olszewski, M.Chai, Y.Fu, and S.Tulyakov, "R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis". arXiv:2203.17261[cs.CV], ECCV2022

1. mebiusbox, 『基礎からはじめる物理ベースレンダリング』. https://zenn.dev/mebiusbox/books/619c81d2fbeafd, 2021
1. 山内, 『三次元空間のニューラルな表現とNeRF』. ALBERT Official Blog, https://blog.albert2005.co.jp/2020/05/08/nerf/, 2020
1. 金谷, 菅谷, 金澤, 『3次元コンピュータビジョン計算ハンドブック』. 森北出版, 2016
1. 加藤, 『微分可能レンダリング』. https://speakerdeck.com/hkato/wei-fen-ke-neng-rendaringu-cvimyan-jiu-hui-tiyutoriaru, CVIM&PRMU研究会チュートリアル, 2022

<!-- interpolation paper -->

<style>
div {
  font-size: 12px;
}
li{
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
li p{
  line-height: 20px;
  margin-top: 0px;
  margin-bottom: 0px;
  padding-top: 0px;
  padding-bottom: 0px;
}
</style>


