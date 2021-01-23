---
layout: post
title: UbuntuのEPGStation_v2でCMカットエンコードする。
---
## はじめに
EPGStation v2が先日公開された。  
EPGSstationのおかげでTVをいつでもどこからでも視聴できる上、気が向いたときに録画予約を入れる事ができ大変便利である。  
私はEPGSstation v2が公開された日にアップグレードした。  
すると、エンコード進捗を表示できる機能が新たに実装されており、感動した。  
そこで私の移植したLinuxで動くJoinLogoScpTrialSetLinuxを用いてエンコード進捗を表示できるようにした。下記のように表示される。

<blockquote class="twitter-tweet" data-conversation="none" data-theme="dark"><p lang="ja" dir="ltr">EPGStation v2のエンコード進捗にてjoin logo scp trial linuxの進捗をとりあえず満足いく感じにすることができた。<br>来週末くらいにはGitHubの連携例を更新しようかな。 <a href="https://t.co/joTnUcCTJ8">pic.twitter.com/joTnUcCTJ8</a></p>&mdash; tobitti0 (@tobitti0) <a href="https://twitter.com/tobitti0/status/1350809160963690501?ref_src=twsrc%5Etfw">January 17, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


そこで、環境構築情報をここに記しておく。  
（というか自分用のメモだが。）  
もし、Linux環境にてEPGStation v2でCMカットしたいという人がいれば参考にしていただければ幸いである。   
ただし、私の利用しているのはDockerを用いた環境であるため、直接導入している人は異なると思われるので、参考に頑張ってもらいたい。    

## EPGStationのDockerにJoinLogoScpTrialSetLinuxの導入
どうでもいい前置きが長くなったが、導入手順を記載していく。  
とりあえず、<a href="https://github.com/l3tnun/docker-mirakurun-epgstation" target="_blank">docker-mirakurun-epgstation</a>を動作できるようにしておく。  

動作できたことが確認できたら、一度downしておく。  
`docker-mirakurun-epgstation/epgstation/Dockerfile`を次の内容に置き換える。

<script src="https://gist.github.com/tobitti0/0a1dc30b6a94534e69f80c5602b26333.js"></script>

Dockerfileの中で使っている移植したJoin\_logo\_scp\_trial\_setとdelogoはGitHubにおいてある。
- <a href="https://github.com/tobitti0/JoinLogoScpTrialSetLinux" target="_blank">JoinLogoScpTrialSetLinux</a>
- <a href="https://github.com/tobitti0/delogo-AviSynthPlus-Linux" target="_blank">delogo-AviSynthPlus-Linux</a>

次に、ホスト側のファイルを用意する。  
以前からJoinLogoScpTrialSetLinuxを使っている方はlogoやJLやSettingを一旦避難させて、`docker-mirakurun-epgstation/join_logo_scp_trial`を削除する。  
つかってない人や、過去のものを削除したら`docker-mirakurun-epgstation`の中で`git clone https://github.com/tobitti0/join_logo_scp_trial`を実行する。  
以前から使っていた人は実行したら避難させたものを戻す。

次にdocker-composeを修正する。
`docker-mirakurun-epgstation/docker-compose.yml`内のepgstationのvolumesへ次の内容を追記する。

````
- ./join_logo_scp_trial/JL:/join_logo_scp_trial/JL
- ./join_logo_scp_trial/logo:/join_logo_scp_trial/logo
- ./join_logo_scp_trial/result:/join_logo_scp_trial/result
- ./join_logo_scp_trial/setting:/join_logo_scp_trial/setting
- ./join_logo_scp_trial/src:/join_logo_scp_trial/src 
````

ここまでできたら`docker-compose up -d --build`でビルドする。  
以前の解説と違い、FFmpegやl-smashなどを含んだdocker-avisynthplusというPackageをGithubにて公開したのでFFmpegなどのビルド時間がかからずに構築できるようになった。
- <a href="https://github.com/users/tobitti0/packages/container/package/docker-avisynthplus" target="_blank">docker-avisynthplus</a>  

nvencのためのタグもあるのでGPUエンコードにしたい人はベースイメージのタグをそれに変更し、nvidia-container-runtime等を入れて構築することもできる。
必要であれば解説を追加したいと思うが、とりあえずは書かないでおく。

ビルドできたらjoin_logo_scp_trialのlogoフォルダにロゴデータを入れておく。

## エンコードスクリプトの設定 
次に`docker-mirakurun-epgstation/epgstation/config`にjlse.jsを作成する。中身は次の通り。
<script src="https://gist.github.com/tobitti0/bba90f8d82d1756df6dc14bd3c3e011c.js"></script>
FFmpeのオプションなどは各自好きなように設定してほしい。

`docker-mirakurun-epgstation/epgstation/config/config.json`のencodeには次のような感じで記載する。
````
encode:
    - name: jlse
      cmd: '%NODE% %ROOT%/config/jlse.js'
      suffix: .mp4
      rate: 4.0
````
これでエンコードのところにjlsの選択が現れるので、選択しエンコードすれば問題なく利用できると思う。   
進捗も表示されるはずだ。

注意としては以前からJoinLogoScpTrialSetLinuxを利用していた場合は`docker-mirakurun-epgstation/join_logo_scp_trial`の中身を最新のものにしなければ、進捗は表示されない。  
jlsの設定などはwindowsとかわらないのでwindowsの記事を参考に設定すれば問題ないはず。
jlsの設定ファイルなどは`docker-mirakurun-epgstation/join_logo_scp_trial/`の中をいじればよい。   

## おわりに
雑ですがとりあえず一通り導入方法を記載した。  
導入方法に万一問題があればTwitterまで。  
移植版に問題があればGitHubのissueに。    
現状私が使用できているので、問題なく使用できるはずですが、暇があれば問題には対応します。たぶん。。。

## 謝辞
EPGStationやJoinLogoScpTrialなどを制作された様々な偉大なる先人達に感謝いたします。
