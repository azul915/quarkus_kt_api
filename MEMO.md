# GraalVMのコンテナでKotlinをネイティブコンパイルする

## GraalVMのコンテナを立ち上げてみる
- https://www.graalvm.org/docs/getting-started/
```sh
docker run -it oracle/graalvm-ce:20.0.0 /bin/bash
```

## wgetでKotlin/Nativeをインストールする

- https://nowokay.hatenablog.com/entry/20181125/1543170183
- https://qiita.com/kurun_pan/items/7c37a92ecbd037dfda6f

```sh
# yumのアップデート
bash-4.2# yum update -y

# wgetをインストール
bash-4.2# yum install -y wget
# Kotlin/Nativeをインストール
bash-4.2# wget https://github.com/JetBrains/kotlin/releases/download/v1.3.61/kotlin-native-linux-1.3.61.tar.gz

# tar.gzの解凍
bash-4.2# tar xzf kotlin-native-linux-1.3.61.tar.gz

# => 実行した場所に、kotlin-native-linux-1.3.61ができる
```

## Kotlin/Nativeでネイティブコンパイルする

- 適当にKotlinのファイルを作る

```kotlin
fun main(args: Array<String>) {
    println("Hello World")
}
```

```sh
# kotlin-native-linux-1.3.61配下にbin/kotlinc-nativeがあるので [Kotlinのファイル.kt] -o [出力ファイル]ででネイティブイメージを作る
kotlin-native-linux-1.3.61/bin/kotlinc-native HelloWorld.kt -o hellokt

# 上の例だと「hellokt.kexe」のバイナリファイルができる
# 実行する
./hellokt.kexe
# => Hello World

```

## GraalVMでネイティブコード生成する

- [GraalVM 19.0.0 Docker image does not contain `native-image`](https://github.com/oracle/docker-images/issues/1276)
  - DockerのGraalVMイメージは「native-image」が含まれないので自分で入れないといけないみたい

```sh
# native-imageのインストール
bash-4.2# gu install native-image
```

このあと、`native-image`コマンドでclassファイルがjarファイルを元にnativeイメージを作りたいが...

## (多分) Kotlinのファイルからjarを作るにはKotlinのコンパイラが必要なのでコンパイラをインストールする

```sh
# wgetする 
bash-4.2# wget https://github.com/JetBrains/kotlin/releases/download/v1.3.61/kotlin-compiler-1.3.61.zip

# zipの解凍
## unzipのインストール
bash-4.2# yum install -y unzip

## 解凍
bash-4.2# unzip kotlin-compiler-1.3.61.zip
# => koltincができる
```

## ktファイルをコンパイルして動かしてみる

```sh
# 普通にコンパイル
bash-4.2# kotlinc/bin/kotlinc HelloWorld.kt
# => HelloWorldKt.classができる

# 実行
bash-4.2# kotlinc/bin/kotlin HelloWorldKt
# => HelloWorld



# jarにしてjavaとして動かす
# jarを作る
bash-4.2# kotlinc/bin/kotlinc HellowWorld.kt -include-runtime -d HelloWorld.jar

# jarから実行
bash-4.2# java -jar HelloWorld.jar
# => HelloWorld
```

# jarをnative-imageでネイティブコンパイルする
```sh
bash-4.2# native-image -jar HelloWorld.jar

=> # 実行結果
# Build on Server(pid: 436, port: 34327)*
# [HelloWorld:436]    classlist:  14,316.01 ms,  0.75 GB
# [HelloWorld:436]        (cap):   2,035.84 ms,  0.75 GB
# [HelloWorld:436]        setup:   8,863.30 ms,  0.75 GB
# [HelloWorld:436]   (typeflow):  11,955.95 ms,  0.70 GB
# [HelloWorld:436]    (objects):   5,828.34 ms,  0.70 GB
# [HelloWorld:436]   (features):     443.61 ms,  0.70 GB
# [HelloWorld:436]     analysis:  18,529.35 ms,  0.70 GB
# [HelloWorld:436]     (clinit):     263.57 ms,  0.71 GB
# [HelloWorld:436]     universe:     963.85 ms,  0.71 GB
# [HelloWorld:436]      (parse):   4,157.14 ms,  0.70 GB
# [HelloWorld:436]     (inline):   4,406.00 ms,  0.69 GB
# [HelloWorld:436]    (compile):  22,283.94 ms,  0.70 GB
# [HelloWorld:436]      compile:  31,359.42 ms,  0.70 GB
# [HelloWorld:436]        image:     994.65 ms,  0.72 GB
# [HelloWorld:436]        write:     364.53 ms,  0.72 GB
# [HelloWorld:436]      [total]:  76,932.95 ms,  0.72 GB

# => HelloWorldが作成される

# 作成されたネイティブイメージを実行する
bash-4.2# ./HelloWorld
# => HelloWorld
```

```sh
# ベンチマーク
# bash-4.2# time ./hellokt.kexe 
# Hello World
# 
# real	0m0.004s
# user	0m0.000s
# sys	0m0.000s
# bash-4.2# 
# bash-4.2# 
# bash-4.2# 
# bash-4.2# time ./HelloWorld
# Hello World
# 
# real	0m0.006s
# user	0m0.000s
# sys	0m0.000s
# bash-4.2# 

# => 若干GraalVMが遅い
```

ktファイルからjarを手軽に作りたい

qurkusで`./gradlew quarkusDev`して作ったjarをnative-imageコマンドでバイナリ化してあげれば動きそうな雰囲気


# GradleでKotlinをGraalVM上で動かす
- javaが動くことが前提として必要
```sh
bash-4.2# /usr/bin/java -version
```
## Gradleをバイナリで取ってくる
```sh
bash4.2# wget https://services.gradle.org/distributions/gradle-6.2-bin.zip --no-check-certificate
```

## unzipする
```sh
# unzipのインストールはなければ行う
bash-4.2# yum install -y unzip

# 解凍
bash-4.2# unzip gradle-6.2-bin.zip
# => gradle-6.2ができる

bash-4.2# gradle-6.2/bin/gradle --version

# => Kotlin 1.3.61に対応している模様

# Welcome to Gradle 6.2!
# 
# Here are the highlights of this release:
#  - Dependency checksum and signature verification
#  - Documentation links in deprecation messages
#  - Shareable read-only dependency cache
# 
# For more details see https://docs.gradle.org/6.2/release-notes.html
# 
# 
# ------------------------------------------------------------
# Gradle 6.2
# ------------------------------------------------------------
# 
# Build time:   2020-02-17 08:32:01 UTC
# Revision:     61d3320259a1a0d31519bf208eb13741679a742f
# 
# Kotlin:       1.3.61
# Groovy:       2.5.8
# Ant:          Apache Ant(TM) version 1.10.7 compiled on September 1 # 2019
# JVM:          1.8.0_242 (Oracle Corporation 25.242-b06-jvmci-20.# 0-b02)
# OS:           Linux 4.9.184-linuxkit amd64
```

# ネイティブコンパイル
## GraalVMを使ってネイティブバイナリを作成する
* ローカルにGraalVMが入っていれば、その環境を使いバイナリをビルドすることもできるが、Dockerでビルドする
* Dockerでビルドする場合は、`--docker-build`オプションを指定してビルドする（Dockerがインストールされていること）
* ネイティブコンパイルされたバイナリは、build配下に`quarku-1.0.0-SNAPSHOT-runner`のように拡張子なしで作られる
## ネイティブバイナリの実行
* src/main/docker/Dockerfile.native を使用して、コンテナでバイナリを実行する
* イメージを作成する
```shell script
docker build -f src/main/docker/Dockerfile.native -t quarkus/quarkus-kt-api .
```
* コンテナを起動する
```shell script
docker run --rm -p 8080:8080 quarkus/quarkus_kt_api
```
* docker-composeで起動する
```yaml
# docker-compose.yml
version: '3.4'
services:
  api:
    build: 
      context: .
      dockerfile: ./src/main/docker/Dockerfile.native
    ports:
    - 8080:8080
    container_name: kt_api_native
```
```shell script
# execute
$ docker-compose up
```
# ローカルで開発する
* intelliJの内蔵実行環境でGradle実行した方が良さそう...