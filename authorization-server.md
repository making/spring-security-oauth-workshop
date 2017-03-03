## Authorization Serverの作成

次にAuthorization Serverを作成します。今回はユーザー、クライアントともにインメモリなものを使用します。

(DBを使用するバージョンは今後、追加予定・・・)



![image](https://qiita-image-store.s3.amazonaws.com/0/1852/56b57ccb-19e8-46ab-5da4-d2ac508344e0.png)


> **注意**
>
> 本ワークショップではローカル起動時にAuthorization ServerをHTTPで受け付けますが、OAuth2の仕様上、実運用時には**必ずHTTPSで受け付けてください**。
> 
> 「[Cloud Foundryにデプロイ](deploy-to-cloud-foundry.md)」ではHTTPSを使用します。ローカル開発の後、試してみてください。


### プロジェクトの作成

次のコマンドを実行すると、`tweeter-auth`フォルダに雛形プロジェクトが生成されます

```
curl start.spring.io/starter.tgz \
       -d artifactId=tweeter-auth \
       -d baseDir=tweeter-auth \
       -d dependencies=web,cloud-oauth2 \
       -d applicationName=TweeterAuthApplication | tar -xzvf -
```

### AuthorizationServerの実装

Authorization Serverは`@EnableAuthorizationServer`を付けるだけで有効になり、OAuth2トークンのエンドポイントが追加されます。

`src/main/java/com/example/TweeterAuthApplication.java`を次のように記述してください。

``` java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;

@SpringBootApplication
@EnableAuthorizationServer
public class TweeterAuthApplication {

	public static void main(String[] args) {
		SpringApplication.run(TweeterAuthApplication.class, args);
	}
}
```

インメモリのユーザー、クライアントの定義、およびAuthorization Serverの設定を`src/main/resources/application.properties`に行います。

``` properties
# [1]
security.oauth2.client.client-id=demo
security.oauth2.client.client-secret=demo
security.oauth2.client.scope=openid,tweet.read,tweet.write
security.oauth2.client.authorized-grant-types=password,authorization_code
# [2]
security.user.password=password
# [3]
server.context-path=/uaa
# [4]
security.oauth2.authorization.check-token-access=isAuthenticated()
server.port=9999
```

* [1] ... インメモリクライアントの定義。このクライアントに`openid`,`tweet.read`,`tweet.write`の3つのスコープと`password`,`authorization_code`の2つのGrant Typeを許可する。
* [2] ... インメモリユーザーの定義。
* [3] ... localhostなど同じホスト名を使用する場合、のちにWeb UIと連携する場合にセッションが混同してエラーが発生するため、Authorization Serverにはcontext-pathを設定する。異なるホスト名を使用する場合はこの設定は不要。
* [4] ... 次にResource Serverがトークンをチェックするためエンドポイントへのアクセス認可設定。デフォルトでは`denyAll()`になっており、アクセスできないため、設定を変更する。


次のコマンドでこのAuthorization Serverのjarファイルを作成してください。

```
cd tweeter-auth
./mvnw clean package -DskipTests=true
```

### Authorization Serverの起動

次のコマンドでAuthorization Serverを起動してください。

```
java -jar target/tweeter-auth-0.0.1-SNAPSHOT.jar
```

9999番ポートでAuthorization Serverが起動します。

```
...
2017-02-19 03:01:16.953  INFO 81867 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 0
2017-02-19 03:01:17.125  INFO 81867 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 9999 (http)
2017-02-19 03:01:17.130  INFO 81867 --- [           main] com.example.TweeterAuthApplication       : Started TweeterAuthApplication in 6.256 seconds (JVM running for 6.737)
```

また次のログの通り、OAuth2に関するエンドポイントが追加されていることがわかります。

```
2017-02-19 03:01:14.563  INFO 81867 --- [           main] .s.o.p.e.FrameworkEndpointHandlerMapping : Mapped "{[/oauth/authorize]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.security.oauth2.provider.endpoint.AuthorizationEndpoint.authorize(java.util.Map<java.lang.String, java.lang.Object>,java.util.Map<java.lang.String, java.lang.String>,org.springframework.web.bind.support.SessionStatus,java.security.Principal)
2017-02-19 03:01:14.564  INFO 81867 --- [           main] .s.o.p.e.FrameworkEndpointHandlerMapping : Mapped "{[/oauth/authorize],methods=[POST],params=[user_oauth_approval]}" onto public org.springframework.web.servlet.View org.springframework.security.oauth2.provider.endpoint.AuthorizationEndpoint.approveOrDeny(java.util.Map<java.lang.String, java.lang.String>,java.util.Map<java.lang.String, ?>,org.springframework.web.bind.support.SessionStatus,java.security.Principal)
2017-02-19 03:01:14.565  INFO 81867 --- [           main] .s.o.p.e.FrameworkEndpointHandlerMapping : Mapped "{[/oauth/token],methods=[GET]}" onto public org.springframework.http.ResponseEntity<org.springframework.security.oauth2.common.OAuth2AccessToken> org.springframework.security.oauth2.provider.endpoint.TokenEndpoint.getAccessToken(java.security.Principal,java.util.Map<java.lang.String, java.lang.String>) throws org.springframework.web.HttpRequestMethodNotSupportedException
2017-02-19 03:01:14.566  INFO 81867 --- [           main] .s.o.p.e.FrameworkEndpointHandlerMapping : Mapped "{[/oauth/token],methods=[POST]}" onto public org.springframework.http.ResponseEntity<org.springframework.security.oauth2.common.OAuth2AccessToken> org.springframework.security.oauth2.provider.endpoint.TokenEndpoint.postAccessToken(java.security.Principal,java.util.Map<java.lang.String, java.lang.String>) throws org.springframework.web.HttpRequestMethodNotSupportedException
2017-02-19 03:01:14.566  INFO 81867 --- [           main] .s.o.p.e.FrameworkEndpointHandlerMapping : Mapped "{[/oauth/check_token]}" onto public java.util.Map<java.lang.String, ?> org.springframework.security.oauth2.provider.endpoint.CheckTokenEndpoint.checkToken(java.lang.String)
2017-02-19 03:01:14.567  INFO 81867 --- [           main] .s.o.p.e.FrameworkEndpointHandlerMapping : Mapped "{[/oauth/confirm_access]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.security.oauth2.provider.endpoint.WhitelabelApprovalEndpoint.getAccessConfirmation(java.util.Map<java.lang.String, java.lang.Object>,javax.servlet.http.HttpServletRequest) throws java.lang.Exception
2017-02-19 03:01:14.568  INFO 81867 --- [           main] .s.o.p.e.FrameworkEndpointHandlerMapping : Mapped "{[/oauth/error]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.security.oauth2.provider.endpoint.WhitelabelErrorEndpoint.handleError(javax.servlet.http.HttpServletRequest)
```

###  Authorization Serverの動作確認

* アクセストークンの発行(`grant_type=password`)

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password http://localhost:9999/uaa/oauth/token
{"access_token":"4fc79a8d-8ac0-44db-8fb7-4a8b69c92d29","token_type":"bearer","expires_in":43199,"scope":"openid tweet.read tweet.write"}
```

* アクセストークンの発行(`grant_type=password`)、スコープ指定(`tweet.read`のみ)

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password -d scope=tweet.read http://localhost:9999/uaa/oauth/token
{"access_token":"ae49ce6a-d899-47ef-b782-008377e3b07f","token_type":"bearer","expires_in":43199,"scope":"tweet.read"}
```

* クライアント認証失敗

```
$ curl -X POST -u demo:secret -d grant_type=password -d username=user -d password=password http://localhost:9999/uaa/oauth/token
{"timestamp":1487441862663,"status":401,"error":"Unauthorized","message":"Bad credentials","path":"/uaa/oauth/token"}
```

* ユーザー認証失敗

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password2 http://localhost:9999/uaa/oauth/token
{"error":"invalid_grant","error_description":"Bad credentials"}
```

* 許可されていないスコープの指定

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password -d scope=tweet.admin http://localhost:9999/uaa/oauth/token
{"error":"invalid_scope","error_description":"Invalid scope: tweet.admin","scope":"openid tweet.read tweet.write"}
```

* 許可されていないGrant Typeの指定

```
$ curl -X POST -u demo:demo -d grant_type=client_credentials http://localhost:9999/uaa/oauth/token
{"error":"invalid_client","error_description":"Unauthorized grant type: client_credentials"}
```

* トークンのチェック(存在する)

```
$ curl -X GET -u demo:demo http://localhost:9999/uaa/oauth/check_token?token=4fc79a8d-8ac0-44db-8fb7-4a8b69c92d29
{"exp":1487484898,"user_name":"user","authorities":["ROLE_USER","ROLE_ACTUATOR"],"client_id":"demo","scope":["openid","tweet.read","tweet.write"]}
```


* トークンのチェック(存在しない)

```
$ curl -X GET -u demo:demo http://localhost:9999/uaa/oauth/check_token?token=ae49ce6a-d899-47ef-b782-008377e3b07a
{"error":"invalid_token","error_description":"Token was not recognised"}
```

### Resource Serverの認可制御

このAuthorization Serverを使ってResource Serverへのアクセスの認可制御を行います。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/8be2ab91-4db3-f838-90f6-fcdbd6d02ace.png)


`tweeter-api`の`src/main/java/com/example/TweeterApiApplication.java`を次の内容に変更してください。

> **メモ**
>
> 変更点は[こちら](https://github.com/tweeter-service/tweeter-api/compare/master...enable-resource-server?diff=split)から確認できます。


``` java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
// ここから追加 (1/2)
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
// ここまで (1/2)

@SpringBootApplication
public class TweeterApiApplication {

	public static void main(String[] args) {
		SpringApplication.run(TweeterApiApplication.class, args);
	}

	// ここから追加 (2/2)
	@Configuration
	@EnableResourceServer // [1]
	static class ResourceServerConfig extends ResourceServerConfigurerAdapter {
		@Override
		public void configure(HttpSecurity http) throws Exception {
			// [2]
			http.authorizeRequests()
					.antMatchers(HttpMethod.GET, "/v1/**")
					.access("#oauth2.hasScope('tweet.read')")
					.antMatchers(HttpMethod.POST, "/v1/**")
					.access("#oauth2.hasScope('tweet.write')")
					.antMatchers(HttpMethod.PUT, "/v1/**")
					.access("#oauth2.hasScope('tweet.write')")
					.antMatchers(HttpMethod.DELETE, "/v1/**")
					.access("#oauth2.hasScope('tweet.write')")
					.antMatchers(HttpMethod.PATCH, "/v1/**")
					.access("#oauth2.hasScope('tweet.write')");
		}
	}
	// ここまで (2/2)
}
```

* [1] ... `@EnableResourceServer`をつけることでアクセストークンを使った認可が有効になる。
* [2] ... Tweeter APIのうち、GETメソッドには`tweet.read`スコープ、POST, PUT, DELETE, PATCHメソッドには`tweet.write`スコープが必要とする。

また、


``` properties
# [1]
security.basic.enabled=false
spring.jackson.serialization.write-date-as-timestamps=false
server.port=8082
# [2]
security.oauth2.resource.token-info-uri=http://localhost:9999/uaa/oauth/check_token
security.oauth2.client.client-id=demo
security.oauth2.client.client-secret=demo
```

* [1] ... デフォルトで有効になっているBasic認証は不要なので、無効にする
* [2] ... アクセストークンをチェックするためのエンドポイントの設定

`tweeter-api-master`ディレクトリに移動して、次のコマンドで再度ビルドしてください。

```
./mvnw clean package -DskipTests=true
```

ビルドできたら再度Resource Serverを起動してください。

```
java -jar target/tweeter-api-0.0.1-SNAPSHOT.jar
```

### Resource Serverの連携動作確認

まずはAuthorization Serverからアクセストークンを発行します。

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password http://localhost:9999/uaa/oauth/token
{"access_token":"4fc79a8d-8ac0-44db-8fb7-4a8b69c92d29","token_type":"bearer","expires_in":40939,"scope":"openid tweet.read tweet.write"}
```

このアクセストークンを使ってResource Serverにアクセスします。アクセストークンは`Authorization`ヘッダーに`Bearer <Token>`の形式で設定します。
Tweetを投稿しましょう。

```
$ curl -X POST -H 'Authorization: Bearer 4fc79a8d-8ac0-44db-8fb7-4a8b69c92d29' -H 'Content-Type: application/json' -d '{"text":"Hello World!"}' http://localhost:8082/v1/tweets
{"tweetId":"1ea78b32-5d78-4f6e-84c9-3c678daa49b6","text":"Hello World!","username":"user","createdAt":"2017-02-18T18:55:13Z"}
```

Tweetの取得も可能です。

```
$ curl -X GET -H 'Authorization: Bearer 4fc79a8d-8ac0-44db-8fb7-4a8b69c92d29' http://localhost:8082/v1/tweets
[{"tweetId":"1ea78b32-5d78-4f6e-84c9-3c678daa49b6","text":"Hello World!","username":"user","createdAt":"2017-02-18T18:55:13Z"}]
```

次に`tweet.read`スコープのみを持つアクセストークンを使ってResource Serverにアクセスしましょう。

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password -d scope=tweet.read http://localhost:9999/uaa/oauth/token
{"access_token":"ae49ce6a-d899-47ef-b782-008377e3b07f","token_type":"bearer","expires_in":40719,"scope":"tweet.read"}
```

このトークンを使った場合、Tweetの取得は可能ですが、

```
$ curl -X GET -H 'Authorization: Bearer ae49ce6a-d899-47ef-b782-008377e3b07f' http://localhost:8082/v1/tweets
[{"tweetId":"1ea78b32-5d78-4f6e-84c9-3c678daa49b6","text":"Hello World!","username":"user","createdAt":"2017-02-18T18:55:13Z"}]
```

Tweetの投稿はできません。

```
$ curl -X POST -H 'Authorization: Bearer ae49ce6a-d899-47ef-b782-008377e3b07f' -H 'Content-Type: application/json' -d '{"text":"Hello World!"}' http://localhost:8082/v1/tweets
{"error":"insufficient_scope","error_description":"Insufficient scope for this resource","scope":"tweet.write"}
```

---

curlコマンドを使ってOAuth2対応したTweeter APIの動作確認をしました。
次にこのAPIを使ったクライアントアプリケーションを作成しましょう。
