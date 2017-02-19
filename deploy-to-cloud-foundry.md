## Cloud Foundryにデプロイ

Authorization Server、Resource Server、Web UIをCloud Foundryにデプロイします。

ここでは[Pivotal Web Services](https://run.pivotal.io)を使用します。Cloud Foundryを使うことで各サーバーのHTTPS対応も自動で行われます。

アカウントの作成方法及び、ログイン方法は[こちら](https://github.com/Pivotal-Japan/cf-workshop/blob/master/pivotal-web-services.md)を確認してください。


以下の`manifest.yml`中の全ての`tmaki`を自分のイニシャルに変更してください。


### Authorization Serverをデプロイ

`tweeter-auth`フォルダの直下で、`manifest.yml`に次の内容を記述してください。`tmaki`の部分を自分のイニシャルに変更してください。

``` yaml
---
applications:
- name: tweeter-auth-tmaki # 要変更
  buildpack: java_buildpack
  memory: 512m
  path: target/tweeter-auth-0.0.1-SNAPSHOT.jar
  env:
    server.context-path: ""
    security.oauth2.client.auto-approve-scopes: openid,tweet.read,tweet.write
```

各サーバーで異なるホスト名を使用するので`server.context-path`を`/uaa`を設定する必要はありません。

次のコマンドでデプロイしてください。

```
../tweeter-auth
cf push
```

次のURLでアクセス可能です

https://tweeter-auth-tmaki.cfapps.io/

アクセストークンを発行してみましょう。

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password https://tweeter-auth-tmaki.cfapps.io/oauth/token
{"access_token":"3728ca1e-e6c0-4eb2-990b-f2cd8abcb61f","token_type":"bearer","expires_in":35153,"scope":"openid tweet.read tweet.write"}
```

### Resource Serverのデプロイ

Resource ServerのバックエンドDBとして、MySQLのサーボスインスタンスを作成します。

```
cf create-service cleardb spark tweeter-db
```

`tweeter-api-master`フォルダの直下で、`manifest.yml`に次の内容を記述してください。`tmaki`の部分を自分のイニシャルに変更してください。

``` yaml
---
applications:
- name: tweeter-api-tmakii # 要変更
  buildpack: java_buildpack
  memory: 512m
  path: target/tweeter-api-0.0.1-SNAPSHOT.jar
  env:
    security.oauth2.resource.token-info-uri: https://tweeter-auth-tmaki.cfapps.io/oauth/check_tokeni # 要変更(HTTPSにすること)
  services:
  - tweeter-db
```

次のコマンドでデプロイしてください。アプリケーション側にMySQLの設定は不要です。JDBCドライバもステージング中に自動で組み込まれます。

```
cd ../tweeter-api-master
cf push
```

次のURLでアクセス可能です。

https://tweeter-api-tmaki.cfapps.io/

### Web UIのデプロイ

`tweeter-webui`フォルダの直下で、`manifest.yml`に次の内容を記述してください。`tmaki`の部分を自分のイニシャルに変更してください。

``` yaml
---
applications:
- name: tweeter-webui-tmaki # 要変更
  buildpack: java_buildpack
  memory: 512m
  path: target/tweeter-webui-0.0.1-SNAPSHOT.jar
  env:
    tweeter.api.uri: https://tweeter-api-tmaki.cfapps.io/v1 # 要変更(HTTPSにすること)
    tweeter.auth.uri: https://tweeter-auth-tmaki.cfapps.io # 要変更(HTTPSにすること)
```

次のコマンドでデプロイしてください。

```
cd ../tweeter-webui
cf push
```

次のURLでアクセス可能です

https://tweeter-webui-tmaki.cfapps.io/
