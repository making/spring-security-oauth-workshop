## Resource Server (REST API)の準備

まずはResource Serverから作成します。本ワークショップではResource Serverは時間短縮のため作成済みのものを使用し、REST APIの実装は行いません。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/d1174c2c-9f55-cbcd-4e49-510647293a3a.png)

### Resource Serverプロジェクトのダウンロード

[Resource Serverプロジェクト](https://github.com/tweeter-service/tweeter-api/archive/master.zip)をダウンロードして展開ください。

以下のコマンドでプロジェクトを取得しても構いません。

```
wget https://github.com/tweeter-service/tweeter-api/archive/master.zip -O tweeter-api-master.zip
unzip tweeter-api-master.zip
rm -f tweeter-api-master.zip
```

または次のコマンドでも構いません。

```
git clone https://github.com/tweeter-service/tweeter-api.git -b master tweeter-api-master
```

展開したフォルダに移動して、ビルドしてください。

```
cd tweeter-api-master
./mvnw clean package -DskipTests=true
```


> **メモ**
> 
> Resource Serverを0から作成したい場合は、次のコマンドを実行して雛形プロジェクトを作成してください。
> 
> ```
> curl start.spring.io/starter.tgz \
>        -d artifactId=tweeter-api \
>        -d baseDir=tweeter-api \
>        -d dependencies=web,h2,jdbc,flyway,cloud-oauth2 \
>        -d applicationName=TweeterApiApplication | tar -xzvf -
> ```

### Resource Serverの起動

次のコマンドでResource Serverを起動してください。

```
java -jar target/tweeter-api-0.0.1-SNAPSHOT.jar
```

8082番ポートでResource Serverが起動します。

```
...
2017-02-19 01:46:12.747  INFO 68878 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 0
2017-02-19 01:46:12.902  INFO 68878 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8082 (http)
2017-02-19 01:46:12.907  INFO 68878 --- [           main] com.example.TweeterApiApplication        : Started TweeterApiApplication in 6.823 seconds (JVM running for 7.306)
```

### Resource Serverの動作確認

最初の実装ではResource ServerにはBasic認証が設定されており、組み込みユーザー`user:password`が利用可能です。


* Tweet全件取得

```
$ curl -X GET -u user:password http://localhost:8082/v1/timelines
[]
```

* ログインユーザーのTweet全件取得

```
$ curl -X GET -u user:password http://localhost:8082/v1/tweets
[]
```

* Tweet投稿

```
$ curl -X POST -u user:password -H 'Content-Type: application/json' -d '{"text":"Hello World!"}' http://localhost:8082/v1/tweets
{"tweetId":"6da25fa9-522a-4737-9623-053f48fc758f","text":"Hello World!","username":"user","createdAt":"2017-02-18T16:54:28Z"}
```

再度、Tweet全件取得

```
$ curl -X GET -u user:password http://localhost:8082/v1/tweets
[{"tweetId":"6da25fa9-522a-4737-9623-053f48fc758f","text":"Hello World!","username":"user","createdAt":"2017-02-18T16:54:28Z"}]
```

今後、このResource ServerをOAuth2対応するために修正していきます。
