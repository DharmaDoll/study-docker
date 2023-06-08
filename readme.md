- [MERN Docker Compose DEMO www](#mern-docker-compose-demo-www)
- [基本 build\&run](#基本-buildrun)
  - [-d はデタッチでコンテナをバックグランドで動かすイメージ。これ理解してないとハマる。](#-d-はデタッチでコンテナをバックグランドで動かすイメージこれ理解してないとハマる)
- [DockerfileのCMDとENTRYPOINT](#dockerfileのcmdとentrypoint)
  - [ENTRYPOINTの有無でCMDの役割が変わる（分かりやすい例）](#entrypointの有無でcmdの役割が変わる分かりやすい例)
  - [ベストプラクティスとしては…](#ベストプラクティスとしては)
  - [コマンド指定の仕方（文字列（シェル形式） or 配列（exec形式））](#コマンド指定の仕方文字列シェル形式-or-配列exec形式)
- [Docker上でcronを動かす時のDockerfile sample](#docker上でcronを動かす時のdockerfile-sample)
- [Docker-compose](#docker-compose)
  - [Syntax](#syntax)



## MERN Docker Compose DEMO www
 Mongodb, Express.js, React, Node.js シンプルで良い。 Dockerizeの参考に。
- https://github.com/usagihitsuji/shellscripts/blob/master/Dockerfiles/mern-docker-compose/
- https://github.com/sidpalas/mern-docker-compose
 
## 基本 build&run
```bash
docker build --no-cache=true -t IMAGE_NAME .

# これで動かす。
docker run -d --name CONTAINER_ID IMAGE_NAME

#ホストの12345で待ち受けて22に転送、ホストのexportフォルダをコンテナのtmpフォルダに公開。
# Docker for macの場合、ホスト側のdirはの設定で追加しとく必要がある
docker run -it --rm --name {name} -p 12345:22 -v /usr/export:/コンテナのdir/tmp {IMAGE_NAME testtag} bin/bash
```
### -d はデタッチでコンテナをバックグランドで動かすイメージ。これ理解してないとハマる。
*
*Docker Container上ではProcessがForegroundで動いていないとContainerは終了する』というContainerの制限がある中で、** 1つのContainer上で複数のプロセス (例えばNode + MongoやFluentd + Elasticsearchなど)を動かしたい、永続化させたい時にSupervisorというToolが使われます。



## [DockerfileのCMDとENTRYPOINT](https://qiita.com/uehaj/items/e6dd013e28593c26372d#%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%81%AE%E5%AE%9F%E8%A1%8C%E3%81%A8%E8%B5%B7%E5%8B%95%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9%E3%81%AE%E6%8C%87%E5%AE%9A)
そもそも「コンテナ」は、構築されたDockerイメージに対して、コンテナを生成し、その中で 何か1つのプロセスを起動・実行 することで、意味のある動作をさせることができるVM(サーバとして継続的に実行したり、コマンドラインから1ショットで処理をするなど)。  

コンテナ内で起動する(唯一の)プロセスの指定方法には以下の2つがある。

- `docker run` の後にコマンドを記述
  - run時にイメージ内の任意のコマンドからプロセスを実行できる、自由度が高いコンテナ
  - 「docker run コマンド指定(+引数)」でプロセスを起動する
- Dockerfile内の`ENTRYPOINT`項目で指定
  - あらかじめビルド時に起動するプロセスが特定され、run時にはそのプロセスに対する引数だけが指定できる自由度の低いコンテナ
  - この場合でも実行するにはdocker runでイメージを起動する
  - この場合にdocker runの後に指定するのはプロセス指定ではなく`ENTRYPOINT`指定されるプロセスに与える引数   

これら2つの指定方法は基本的に排他。つまり、
- `ENTRYPOINT`を指定したときは、runの後の記述は起動するプロセスを特定する指示としては機能しないし、
- `docker run`で起動するプロセスを指定できるのは、Dockerfileで`ENTRYPOINT`を指定せずに構築されたコンテナに対してのみである。 

ENTRYPOINTでの指定は、ビルド時に確定してイメージに封入されてしまうことも留意。(必要なら`run --entrypoint=""`で上書きすることもできるとのこと)   

 - あくまで`CMD`は単なるrunの後に続ける引数のデフォルト値指定であると言える。
**Entrypointの有無でコマンドか引数かの役割が変わる**

### ENTRYPOINTの有無でCMDの役割が変わる（分かりやすい例） 
**`CMD` のみの場合**は、`CMD` の第一引数がコマンドとみなされます。docker run の引数により上書きされます。

```Dockerfile
CMD ["/bin/cmd", "arg1", "arg2"]
```

```bash
docker run image1                       #/bin/cmd arg1 arg2 が実行される
docker run image1 /bin/cmd2 arg3 arg4   #/bin/cmd2 arg3 arg4 が実行される
```

**`ENTRYPOINT` のみ**の場合は`ENTRYPOINT`run引数が実行されます。

```Dockerfile
ENTRYPOINT ["/bin/cmd", "arg1", "arg2"]
```

```bash
docker run image1            # /bin/cmd arg1 arg2 が実行される
docker run image1 arg3 arg4  # /bin/cmd arg1 arg2 arg3 arg4 が実行される
```
- http://www.tohoho-web.com/docker/dockerfile.html#cmd

`ENTRYPOINT` と `CMD` が exec形式で指定された場合、
- docker run を引数無しで実行すると `ENTRYPOINT` ＋ `CMD` が、
- 引数有りで実行すると `ENTRYPOINT` ＋ `run`引数が実行されます。

```Dockerfile

ENTRYPOINT ["/bin/cmd", "arg1", "arg2"]
CMD ["arg3", "arg4"]

```

```bash
docker run image1            # /bin/cmd arg1 arg2 arg3 arg4 が実行される
docker run image1 arg5 arg6  # /bin/cmd arg1 arg2 arg5 arg6 が実行される
```

### ベストプラクティスとしては…
- サービスの起動が固定されている場合（自由度が低い）は`ENTORYPOINT`のみで
- それ以外は`ENTRYPOINT` + `CMD`(デフォルトパラメタ用) を基本つけよう。基本的にdockerなので起動させたいサービスやアプリはあるでしょう。
  - `CMD`はデフォルト値またはオプションの為に利用
    - この場合は、必ず配列形式(exec system call)で指定する
  - 適宜パラメタを変える場合は`docker run`で上書きする
  - 最悪 `--entrypoint` オプションで変えてあげれば良いだけ
- 使う時点ではそんなに目的もなくただコンテナを立ち上げてるだけなら`ENTORYPOINT`なしで`docker run`時に指定すればよい
```Dockerfile
ENTRYPOINT ["npm"]
CMD ["run", "start"] 

#これだと起動すらできない。第３引数以降に"/tail -f ..."が渡る
ENTRYPOINT ["/etc/init.d/cron", "start"]
CMD ["tail", "-f", "/dev/null"]
```

- 明確なサービスとかを起動する必要がない場合は`CMD`/`ENTRYPOINT`を何も付けずに`docker run`で指定してやれば良い


### コマンド指定の仕方（文字列（シェル形式） or 配列（exec形式））
`ENTRYPOINT`、`CMD`ともに、
- 「文字列の配列」で指定するとexecシステムコールがプロセスの起動に使用される。
- 「空白区切りの文字列」で指定すると「/bin/sh -c」が付与されてシェル経由で起動。

```
ENTRYPOINT ["/bin/cmd", "arg1", "arg2"]   # exec形式
ENTRYPOINT /bin/cmd arg1 arg2             # シェル形式
```
注意点として、
`CMD`に関しては、「/bin/sh -c」の付与は`ENTRYPOINT`「追加引数」として使用される場合も区別されずに適用され、「["/bin/sh","-c"]」が引数に含まれるようになる。
この動作は意図しないものである可能性が高いので、一般には`ENTRYPOINT`対する`CMD`は文字列の配列形式で指定することが多い。

そして、この場合、`CMD`はデフォルト引数になり、docker runする時に上書き可能。

## Docker上でcronを動かす時のDockerfile sample
そもそもcronって環境変数読み込まない？のでPATHとか明示的に読み込ませないとうまく動かないので注意 cronの設定の先頭に必要な環境変数exportとかして対応

```Dockerfile
#COPYするファイルに定期実行したい.shなど入っててcron.configの設定読み込ませてcron指定するだけ
#ただ、そもそもcronって環境変数読み込まないのでPATHとか明示的に読み込ませないとうまく動かないので注意 sed のところ
FROM perl:5.20

#for cpan depandency
RUN apt-get update &&\
apt-get install debian-keyring debian-archive-keyring &&\
 apt-get install -y git &&\
 apt-get install -y vim &&\
 apt-get install -y cron &&\
 apt-get install -y libgd2-noxpm-dev &&\
 apt-get install -y libgd2-xpm-dev

COPY ./201610/ /root

#update cpan mod
WORKDIR /root/app/
RUN git clone https://github.com/miyagawa/cpanminus.git &&\
 cpanm --installdeps .

RUN TZ=Asia/Tokyo &&\
 ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone &&\
 sed -i '1iPATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\n' cron.config &&\
 crontab /root/cron.config

#とりあえず起動させたいだけ(cronだけ動かしてたい)
CMD /etc/init.d/cron start && tail -f /dev/null
#ENTRYPOINT /etc/init.d/cron start && tail -f /dev/null #これでも良さげ？

#これだと駄目起動できない、、そりゃそうだ。第３引数に"/tail -f ..."が渡っちゃう
ENTRYPOINT ["/etc/init.d/cron", "start"]
CMD ["tail -f /dev/null"]
```



## [Docker-compose](https://docs.docker.jp/v17.06/compose/index.html)
- 複数のコンテナを定義し実行する Docker アプリケーションのためのツールです。Compose においては YAML ファイルを使ってアプリケーションサービスの設定を行います。コマンドを１つ実行するだけで、設定内容に基づいたアプリケーションサービスの生成、起動を行います。 
- dockerfile を使い、docker run することでも動かすことは可能ですが、起動時の設定を覚えるのが大変なので docker-compose.yaml でその設定を書いてしまう方が良い場合も
docker run のコマンドが複雑で、複数コンテナの場合は docker-compose 使う方がシンプル。
- コンテナ内でカスタマイズのアプリを使う場合、当然imageは事前に`docker build`で作っとく必要あり。ただのミドルウェア使いたいだけとかは、勝手にpullしてくると思う。(build . みたいな感じで手元のDockerfileでbuildしてくれる設定もある)   
 - https://qiita.com/Gucchiy/items/8a4421dd087a99b44c78   
 - [その二　完](https://qiita.com/Gucchiy/items/8d078dde6d890fdd8144)
 - [デフォルトのnaming ruleについて。変更方法なども](https://qiita.com/satodoc/items/188a387f7439e4ec394f)
 - https://github.com/usagihitsuji/shellscripts/tree/master/Dockerfiles/mern-docker-compose
### Syntax
Docker Composeのdocker-compose.ymlファイルでは`entrypoint`はentrypointのまま、`CMD`はcommand項目に対応する。
- http://docs.docker.jp/compose/compose-file.html#entrypoint
- http://docs.docker.jp/compose/compose-file.html#command


