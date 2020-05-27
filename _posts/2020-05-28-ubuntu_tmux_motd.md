---
layout: post
title: UbuntuへSSHするとtmuxが立ち上がる場合にmotdを表示する
image: /images/2020-05/28/2020-05-28.png
---

誰得って話ですが、メモ書き程度に。  
UbuntuにSSHで接続すると特に設定をいじってなければmotdが表示される。  
motdってのはmessage of the dayの略で今日のメッセージらしい。  
Ubuntuでは通常次のような表示。  

``` console
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 5.3.0-53-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 * MicroK8s passes 9 million downloads. Thank you to all our contributors!

     https://microk8s.io/

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

0 個のパッケージがアップデート可能です。
0 個のアップデートはセキュリティアップデートです。

Your Hardware Enablement Stack (HWE) is supported until April 2023.
Last login: Wed May 27 20:30:11 2020 from 192.168.1.1
```

私は、UbuntuをTV録画サーバとしていて、ちょくちょくメンテや移植しているCMカットツールの開発のためにSSHで接続する。  
というか、画面をつないでないので、SSHでしか操作できない。  
鯖にはzshとtmuxが入っていて接続すると、tmuxが立ち上がる。  
しかし、接続するとmotdが一瞬表示されるが、tmuxが表示されるので消えてしまう。  
ログアウトすると見えるが、パッケージのアップデート情報や、再起動要求などが表示されるのでできればみたいなと思い、今の環境でも最初に表示されるようにすることにした。  
<img src="{{ site.baseurl }}/images/2020-05/28/motd-clear.gif" alt="motd clear"/>


前提として、SSHでつなぐとtmuxが立ち上がるようにしているので、.zshrcにつぎの記載がある。bashを使ってる人は.bashrcに書いてあるのかな。

``` shell
# SHELL LOGIN WITH TMUX / If not running interactively, do not do anything
[[ $- != *i* ]] && return
[[ -z "$TMUX" ]] && exec tmux
```

早速だが、motdを表示するようにするには.zshrcに次の内容を記載すればいい。
記載してからリロードすれば動作すると思う。

``` shell
# Display MOTD in the first pane when connected to SSH
if [ -n "$SSH_CONNECTION" ] ; then
  case "$(tmux display-message -p  "#{pane_index}")" in
    1)  cat /run/motd.dynamic
        last $USER --time-format full | grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' | perl -lane '!/still logged in/ && print "Last login: @F[3..7] from $F[2]" and last'
        ;;
    *)  ;;
  esac
fi
```
こんなふうに動作する。最初に消えるmotdがあるので少しちらついて見える。デフォルトのmotdを無効にしてもいいかもしれない。  
<img src="{{ site.baseurl }}/images/2020-05/28/motd-fix.gif" alt="motd show"/>

不要かもしれないが、少し解説をする。  
`if [ -n "$SSH_CONNECTION" ] ; then`の部分が、SSH接続かどうかを判定している。SSH接続ならばtrueでifの中身が実行される。  

`"$(tmux display-message -p  "#{pane_index})"`この部分は、tmuxのウィンドウのペイン番号を取得している。  
ちなみにtmuxには`${TMUX_PANE}`という使えそうなenvがあるが、これはtmuxサーバー全体のペイン番号が入っている。そのため、0のウィンドウでペインを2個開いていた場合、新規にSSHで接続すると3が返ってくる。なので、SSHで接続し初めて表示する画面かどうかがわからない。  
そもそも、ペイン番号取得とかいるのか？って話ではあるが、なにも判定をしないと、ペインを開く(分割する)たびに表示されて非常にうざいことになるので必要であると思われる。  
あと、通常のmotdはSSHで別の接続をすると表示されるので、そのへんも似せたかった。なのでtmuxサーバー全体ではなくウインドウのペイン数を取得している。  

`1)  cat /run/motd.dynamic`の部分は、先に取得したウインドウのペイン番号が1のとき（1からはじまるようだ）にmotdを表示する処理である。 
私はこれで、やったぜ、完成だわって思ったのですが、よくよく見るとLast loginの表示がなかったんですよね… 

`last $USER --time-format full | grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' | perl -lane '!/still logged in/ && print "Last login: @F[3..7] from $F[2]" and last'`  
せっかくだし表示するかってことで追加したこの行がLast loginを表示する処理です。  
タイトルにUbuntuって書いてるのですが、それはだいたいこのコマンドが原因です。通常lastは日付周りが略されています。(略かはしらんけど通常は)秒と年がありません。Ubuntuは`--time-format full`で表示できたのですが、他がわからないのでそういう風に書いています。(motdの位置も他のは違うかもしれないが…)  
grepしているのはIPを表示するためです。tmuxのペインが立ち上がるとログのに`tmux(xxxx).%1`とかが記録されるようです。%のあとの数字はtmuxサーバー全体のペイン番号みたいだ。今回はIPの最後のログを表示したいので、grepで抽出しています。

まぁ、本当に誰得って話ではあるけれどメモ代わりに残しておくとする。  
もっとスマートなやり方があるのかもしれないが、今はこれでもとと同じmotdが表示できているので満足している。  
<img src="{{ site.baseurl }}/images/2020-05/28/2020-05-28.png" alt="motd"/>

