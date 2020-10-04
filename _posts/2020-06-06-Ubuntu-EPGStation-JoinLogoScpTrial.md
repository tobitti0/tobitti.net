---
layout: post
title: UbuntuでEPGStationとJoinLogoScpTrialをつかってCMカットする。
---
Linux環境で動作するすばらしいTV録画システムは多くある。  
私は以前WindowsでEDCBを用いてTVを録画していた。  
しかしながら、社会人となるにあたって研修などで自宅を離れることが予定されていた。   
自宅を離れている間も録画システムを運用しつつ、録画番組を自宅外から見れるようにしたいと考えた。   
（コロナ禍によって予定されていた研修等はすべて流れてしまったが…）  
いわゆるロケフリと呼ばれるシステムである。 そこで部屋に余っていたi7-4770Kとマザボや電源を使って新しい録画専用機を組んだ。   
TV録画用のシステムにはEPGStationを採用した。    
採用理由はスマートフォンから利用することを考慮されたUIであることや、Dockerのファイルが提供されていたことが理由である。    
また、TV録画用のシステムのOSにWindowsを利用するのは無駄であると考えたためにUbuntuを採用した。   
そして、無事に環境を構築でき提供されているDockerに、リバースプロキシやOauth2-proxyを用いた認証システムを加えて、通常のTV録画環境をロケフリで動作させることができた。   
加えてエンコードファイルをGoogleDriveに保存し、自分だけのSlackに通知するなどして便利に利用している。（DriveはG Suiteを契約しているので無限なのである。）

通常の録画システムが構築できれば、次はCMカットである。   
Windows環境ではJoinLogoScpTrialという素晴らしいCMカットツールがあった。   
これは非常に素晴らしいツールで偉大なるDTVの先人達が制作された物である。   
無音検出をし、ロゴ区間を検出などの情報を用いてかなり高い精度でCMカットを行う事ができる。    
ただし、Linux環境で動作する全自動でCMカットできるツールが私の調べる限りでは存在しなかった。   

そこで、JoinLogoScpTirialをLinux環境でも利用できれば最高だと考えた。  
ちょうどJoinLogoScp関係で重要な役割を果たしているAvisynth+が3.5でLinuxに正式に対応した。    
それにより、chapter\_exeやlogoframeに幾つかの改変を実施し、Linux環境でJoinLogoScpTrialを動作させることができた。   

移植したJoin\_logo\_scp\_trial\_setとdelogoはGitHubにおいてある。
- <a href="https://github.com/tobitti0/JoinLogoScpTrialSetLinux" target="_blank">JoinLogoScpTrialSetLinux</a>
- <a href="https://github.com/tobitti0/delogo-AviSynthPlus-Linux" target="_blank">delogo-AviSynthPlus-Linux</a>

そこで、それを用いたEPGStationでのCMカットエンコードの環境構築情報をここに記しておく。  
（というか自分用のメモだが。）  
もし、Linux環境でCMカットしたいという人がいれば参考にしていただければ幸いである。   
ただし、私の利用しているDockerを用いた環境であるため、直接導入している人は異なると思われるので、参考に頑張ってもらいたい。    

どうでもいい前置きが長くなったが、導入手順を記載していく。  
とりあえず、<a href="https://github.com/l3tnun/docker-mirakurun-epgstation" target="_blank">docker-mirakurun-epgstation</a>を動作できるようにしておく。  

動作できたことが確認できたら、一度downしておく。  
`docker-mirakurun-epgstation/epgstation/Dockerfile`を次の内容に置き換える。

```` docker
FROM ubuntu18.04
EXPOSE 8888

ARG NASM_VER="2.14"
ARG YASM_VER="1.3.0"
ARG LAME_VER="3.100"

RUN set -xe && \
    apt-get update && \
    apt-get -y install software-properties-common && \
    add-apt-repository ppa:ubuntu-toolchain-r/test && \
    apt-get update && \
    apt-get install -y aptitude && \
    aptitude update && \
    aptitude install -y \
        wget build-essential automake autoconf git libtool libvorbis-dev \
        libass-dev libfreetype6-dev libsdl2-dev libva-dev libvdpau-dev \
        libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev \
        mercurial libnuma-dev texinfo zlib1g-dev \
        cmake qtbase5-dev checkinstall gcc-9 g++-9 curl \
        python3-pip ninja-build \
    &&\
    pip3 install meson

# nasm
RUN set -xe && \
    DIR=/tmp/nasm && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    curl -sL https://www.nasm.us/pub/nasm/releasebuilds/${NASM_VER}/nasm-${NASM_VER}.tar.bz2 | \
    tar -jx --strip-components 1 && \
    ./autogen.sh && \
    ./configure --prefix=/usr/local && \
    make -j$(nproc) && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# yasm
RUN set -xe && \
    DIR=/tmp/yasm && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    curl -sL https://www.tortall.net/projects/yasm/releases/yasm-${YASM_VER}.tar.gz | \
    tar -zx --strip-components 1 && \
    ./configure \
      --prefix=/usr/local && \
    make -j$(nproc) && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# l-smash
RUN set -xe && \
    DIR=/tmp/l-smash && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone https://github.com/l-smash/l-smash.git && \
    cd l-smash && \
    ./configure \
      --enable-shared \
      && \
    make && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# libx264
RUN set -xe && \
    DIR=/tmp/x-264 && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone --depth 1 https://code.videolan.org/videolan/x264.git && \
    cd x264 && \
    ./configure \
      --prefix=/usr/local \
      --enable-shared \
      --enable-pic \
    && \
    make && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# libvpx
RUN set -xe && \
    DIR=/tmp/libvpx && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone --depth 1 https://chromium.googlesource.com/webm/libvpx.git && \
    cd libvpx && \
    ./configure \
      --disable-examples \
      --disable-unit-tests \
      --enable-vp9-highbitdepth \
      --as=yasm \
    && \
    make -j$(nproc) && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# libfdk-aac
RUN set -xe && \
    DIR=/tmp/libfdk-aac && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone --depth 1 https://github.com/mstorsjo/fdk-aac && \
    cd fdk-aac && \
    autoreconf -fiv && \
    ./configure \
    && \
    make -j$(nproc) && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# libmp3lame
RUN set -xe && \
    DIR=/tmp/lame && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    curl -sL https://versaweb.dl.sourceforge.net/project/lame/lame/$(echo ${LAME_VER} | \
    sed -e 's/[^0-9]*\([0-9]*\)[.]\([0-9]*\)[.]\([0-9]*\)\([0-9A-Za-z-]*\)/\1.\2/')/lame-${LAME_VER}.tar.gz | \
    tar -zx --strip-components=1 && \
    ./configure \
      --enable-shared \
      --enable-nasm \
    && \
    make -j$(nproc) && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# libopus
RUN set -xe && \
    DIR=/tmp/libopus && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone --depth 1 https://github.com/xiph/opus && \
    cd opus && \
    ./autogen.sh && \
    ./configure \
      --enable-shared \
    && \
    make -j$(nproc) && \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

#AviSynth+
RUN set -xe && \
    DIR=/tmp/AvisynthPlus && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone --depth 1 git://github.com/AviSynth/AviSynthPlus.git && \
    cd AviSynthPlus && \
    mkdir avisynth-build && \
    cd avisynth-build && \
    CC=gcc-9 CXX=gcc-9 LD=gcc-9 cmake ../ -G Ninja && \
    ninja && \
    checkinstall \
      --pkgname=avisynth \
      --pkgversion="$(grep -r \
      Version avs_core/avisynth.pc | cut -f2 -d " ")-$(date --rfc-3339=date | \
      sed 's/-//g')-git" \
      --backup=no \
      --deldoc=yes \
      --delspec=yes \
      --deldesc=yes \
      --strip=yes \
      --stripso=yes \
      --addso=yes \
      --fstrans=no \
      --default ninja install \
      && \
    rm -rf ${DIR} && \
    ldconfig

# FFmpeg
RUN set -xe && \
    DIR=/tmp/ffmpeg && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    sed -Ei 's/^# deb-src /deb-src /' /etc/apt/sources.list && \
    apt-get update && \
    apt-get -y build-dep ffmpeg && \
    git clone --depth 1 git://git.ffmpeg.org/ffmpeg.git && \
    cd ffmpeg && \
    ./configure \
      --cc=gcc-9 \
      --cxx=gcc-9 \
      --enable-gpl\
      --enable-version3 \
      --disable-doc \
      --disable-debug \
      --enable-pic \
      --enable-avisynth \
      --enable-libx264 \
      --enable-libfdk-aac \
      --enable-libfreetype \
      --enable-libmp3lame \
      --enable-libopus \
      --enable-libvorbis \
      --enable-libvpx \
      --enable-nonfree \
      && \
    make -j$(nproc) && \
    checkinstall --pkgname=ffmpeg --pkgversion="7:$(git rev-list \
    --count HEAD)-g$(git rev-parse --short HEAD)" --backup=no --deldoc=yes \
    --delspec=yes --deldesc=yes --strip=yes --stripso=yes --addso=yes \
    --fstrans=no --default \
    && \
    rm -rf ${DIR}

# l-smash-source
RUN set -xe && \
    DIR=/tmp/l-smash-source && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone https://github.com/HolyWu/L-SMASH-Works.git && \
    cd L-SMASH-Works/AviSynth && \
    git checkout 72d3eac802eebcfc9080009c1a8d47a747e3a306 &&\
    CC=gcc-9 CXX=gcc-9 LD=gcc-9 LDFLAGS="-Wl,-Bsymbolic" meson build && \
    cd build && \
    ninja -v && \
    ninja install && \
    rm -rf ${DIR} && \
    ldconfig

# delogo for Avisynth+ and Linux
RUN set -xe && \
    DIR=/tmp/delogo && \
    mkdir -p ${DIR} && \
    cd ${DIR} && \
    git clone https://github.com/tobitti0/delogo-AviSynthPlus-Linux.git && \
    cd delogo-AviSynthPlus-Linux/src && \
    make CC=gcc-9 CXX=gcc-9 LD=gcc-9&& \
    make install && \
    rm -rf ${DIR} && \
    ldconfig

# install nodejs
RUN curl -sL https://deb.nodesource.com/setup_10.x | bash - && \
    apt-get install -y nodejs

# install EPGStation
RUN cd /usr/local/ && \
    git clone https://github.com/l3tnun/EPGStation.git && \
    cd /usr/local/EPGStation && \
    npm install && \
    npm install googleapis@39 --save && \
    npm install async && \
    npm run build

# join logo scp trial for Linux
ADD JoinLogoScpTrialSetLinux/modules /tmp/modules
RUN set -xe && \
    DIR=/tmp/modules && \
    cd /tmp/modules/chapter_exe/src && \
    make && \
    mv chapter_exe /tmp/modules/join_logo_scp_trial/bin/ && \
    cd /tmp/modules/logoframe/src && \
    make && \
    mv logoframe /tmp/modules/join_logo_scp_trial/bin/ && \
    cd /tmp/modules/join_logo_scp/src && \
    make && \
    mv join_logo_scp /tmp/modules/join_logo_scp_trial/bin/ && \    
    mv /tmp/modules/join_logo_scp_trial /usr/local/EPGStation/join_logo_scp_trial && \
    cd /usr/local/EPGStation/join_logo_scp_trial && \
    rm -rf ${DIR} && \
    npm install && \
    npm link && \
    jlse --help

WORKDIR /usr/local/EPGStation

ENTRYPOINT npm start
````
次に、`docker-mirakurun-epgstation/epgstation`の中で`git clone --recursive https://github.com/tobitti0/JoinLogoScpTrialSetLinux.git`を実行する。  
`docker-mirakurun-epgstation`の中で次のコマンドを実行する。  
````
mkdir join_logo_scp_trial
cp -r epgstation/JoinLogoScpTrialSetLinux/modules/join_logo_scp_trial/JL join_logo_scp_trial
cp -r epgstation/JoinLogoScpTrialSetLinux/modules/join_logo_scp_trial/setting join_logo_scp_trial
cp -r epgstation/JoinLogoScpTrialSetLinux/modules/join_logo_scp_trial/src join_logo_scp_trial
````

`docker-mirakurun-epgstation/docker-compose.yml`にはepgstationのvolumesへ次の内容を追記する。

````
- ./join_logo_scp_trial/JL:/usr/local/EPGStation/join_logo_scp_trial/JL
- ./join_logo_scp_trial/logo:/usr/local/EPGStation/join_logo_scp_trial/logo
- ./join_logo_scp_trial/result:/usr/local/EPGStation/join_logo_scp_trial/result
- ./join_logo_scp_trial/setting:/usr/local/EPGStation/join_logo_scp_trial/setting
- ./join_logo_scp_trial/src:/usr/local/EPGStation/join_logo_scp_trial/src
````

ここまでできたら`docker-compose up -d --build`でビルドする。  
FFmpegとその他をビルドするのでかなり時間がかかりますがゆっくり待ちましょう。   
ビルドできたらjoin_logo_scp_trialにlogoフォルダができてるのでロゴデータを入れておく。
次に`docker-mirakurun-epgstation/epgstation/config`にjlse.jsを作成する。中身は次の通り。

```` javascript
const spawn = require('child_process').spawn;
const ffmpeg = process.env.FFMPEG;

const input = process.env.INPUT;
const output = process.env.OUTPUT;
const analyzedurationSize = '10M'; // Mirakurun の設定に応じて変更すること
const probesizeSize = '32M'; // Mirakurun の設定に応じて変更すること
const maxMuxingQueueSize = 1024;
const dualMonoMode = 'main';
const videoHeight = parseInt(process.env.VIDEORESOLUTION, 10);
const isDualMono = parseInt(process.env.AUDIOCOMPONENTTYPE, 10) == 2;
const audioBitrate = videoHeight > 720 ? '192k' : '128k';
const preset = 'medium';
const codec = 'libx264'; 
const crf = 23;

const args = ['-y', '-analyzeduration', analyzedurationSize, '-probesize', probesizeSize];

const path = require('path');
const output_name = path.basename(output, path.extname(output));
const output_dir = path.dirname(output);

// dual mono 設定
if (isDualMono) {
    Array.prototype.push.apply(args, ['-dual_mono_mode', dualMonoMode]);
}

// メタ情報を先頭に置く
Array.prototype.push.apply(args,['-movflags', 'faststart']);

// 字幕データを含めたストリームをすべてマップ
//Array.prototype.push.apply(args, ['-map', '0', '-ignore_unknown', '-max_muxing_queue_size', maxMuxingQueueSize, '-sn']);

// video filter 設定
let videoFilter = 'yadif';
if (videoHeight > 720) {
    videoFilter += ',scale=-2:720'
}
Array.prototype.push.apply(args, ['-vf', videoFilter]);

// その他設定
Array.prototype.push.apply(args,[
    '-preset', preset,
    '-aspect', '16:9',
    '-c:v', codec,
    '-crf', crf,
    '-f', 'mp4',
    '-c:a', 'aac',
    '-ar', '48000',
    '-ab', audioBitrate,
    '-ac', '2'
]);

let str = '';
for (let i of args) {
    str += ` ${ i }`
}
console.error(str);

const jlse_args = ['-i', input, '-e', '-o', str,'-r', '-d', output_dir, '-n', output_name];
console.error(jlse_args);

var env = Object.create( process.env );
env.HOME = '/root';
console.error(env);
const child = spawn('jlse', jlse_args, {env:env});

child.stderr.on('data', (data) => { console.error(String(data)); });

child.on('error', (err) => {
    console.error(err);
    throw new Error(err);
});

process.on('SIGINT', () => {
    child.kill('SIGINT');
});
````
`docker-mirakurun-epgstation/epgstation/config/config.json`のencodeには次を追記する。
````
{
    "name": "jls",
    "cmd": "%NODE% %ROOT%/config/jlse.js",
    "suffix": ".mp4",
    "default": true
}
````
これでエンコードのところにjlsの選択が現れるので、選択しエンコードすれば問題なく利用できると思う。   

注意としてはFFmpegが4.2以上になるため、現在のEPGStationでは字幕が動作しない。  
詳しい方、字幕をうまく調整してEPGStationにぜひ投げてほしい。（私は詳しくないので無理…）   
あと、現在のJoinLogoScpTrialSetLinuxには保存先ディレクトリを変更する機能を有していないので、EPGStationでエンコードファイルの保存先を変更している場合は問題が発生すると思われる。  
時間があれば対応します。たぶん。。。

雑ですがとりあえず一通り導入方法を記載した。  
jlsの設定などはwindowsとかわらないのでwindowsの記事を参考に設定すれば問題ないはず。
jlsの設定ファイルなどは`docker-mirakurun-epgstation/join_logo_scp_trial/`の中をいじってください。   

導入方法に万一問題があればTwitterまで。  
移植版に問題があればGitHubのissueに。    
現状私が使用できているので、問題なく使用できるはずですが、暇があれば問題には対応します。たぶん。。。

最後に、EPGStationやJoinLogoScpTrialなどを制作された様々な偉大なる先人達に感謝いたします。
