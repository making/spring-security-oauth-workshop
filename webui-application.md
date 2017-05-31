## Web UIアプリケーション (Authorization Code)の作成

いよいよResource Serverと連携したWeb UIを作成しましょう。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/6e64bbf3-7426-1346-cb6b-2fe78edccd5a.png)

### プロジェクトの作成

次のコマンドを実行すると、`tweeter-webui`フォルダに雛形プロジェクトが生成されます

```
curl start.spring.io/starter.tgz \
       -d artifactId=tweeter-webui \
       -d baseDir=tweeter-webui \
       -d dependencies=web,thymeleaf,cloud-oauth2 \
       -d applicationName=TweeterWebuiApplication | tar -xzvf -
```

### Webアプリケーションの作成

のちに作成するファイルの空ファイルを次のコマンドで作成してください。

```
cd tweeter-webui
touch src/main/java/com/example/TweeterController.java
touch src/main/resources/templates/tweets.html
```

まずは画面遷移のためのControllerクラスを作成します。

`src/main/java/com/example/TweeterController.java`に、次の内容を記述してください。

``` java
package com.example;

import java.net.URI;
import java.util.Collections;
import java.util.Date;
import java.util.List;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.RequestEntity;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

@Controller
public class TweeterController {
	private final RestTemplate restTemplate;
	private final URI tweeterApiUri;

	public TweeterController(RestTemplate restTemplate,
			@Value("${tweeter.api.uri}") URI tweeterApiUri) {
		this.restTemplate = restTemplate;
		this.tweeterApiUri = tweeterApiUri;
	}

	@GetMapping("/")
	String timelines(Model model) {
		RequestEntity<?> request = RequestEntity.get(UriComponentsBuilder
				.fromUri(tweeterApiUri).pathSegment("timelines").build().toUri()).build();
		List<Tweet> tweets = restTemplate
				.exchange(request, new ParameterizedTypeReference<List<Tweet>>() {
				}).getBody();
		model.addAttribute("tweets", tweets);
		return "tweets";
	}

	@GetMapping("/tweets")
	String tweets(Model model) {
		RequestEntity<?> request = RequestEntity.get(UriComponentsBuilder
				.fromUri(tweeterApiUri).pathSegment("tweets").build().toUri()).build();
		List<Tweet> tweets = restTemplate
				.exchange(request, new ParameterizedTypeReference<List<Tweet>>() {
				}).getBody();
		model.addAttribute("tweets", tweets);
		return "tweets";
	}

	@PostMapping("/tweets")
	String tweets(@RequestParam String text) {
		RequestEntity<?> request = RequestEntity.post(UriComponentsBuilder
				.fromUri(tweeterApiUri).pathSegment("tweets").build().toUri())
				.body(Collections.singletonMap("text", text));
		restTemplate.exchange(request, Void.class);
		return "redirect:/";
	}

	public static class Tweet {
		public String tweetId;
		public String username;
		public String text;
		public Date createdAt;
	}
}
```

次に`src/main/resources/templates/tweets.html`に次の内容を記述してください。

``` html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title>Tweeter</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/uikit/3.0.0-beta.11/css/uikit.min.css"/>
</head>

<body>

<div class="uk-grid">
    <div class="uk-width-1-5"></div>
    <div class="uk-width-3-5">
        <nav class="uk-navbar-container" uk-navbar="">
            <div class="uk-navbar-left">
                <ul class="uk-navbar-nav">
                    <li th:classappend="${#httpServletRequest.requestURI == '/'} ? 'uk-active'"><a
                            th:href="@{/}">Home</a></li>
                    <li th:classappend="${#httpServletRequest.requestURI == '/tweets'} ? 'uk-active'"><a
                            th:href="@{/tweets}">Tweets</a>
                    </li>
                </ul>
            </div>
        </nav>
        <h1>Tweeter</h1>
        <form th:action="@{/tweets}" class="uk-panel uk-panel-box uk-form" method="post">
            <span th:text="${#httpServletRequest.remoteUser}">user</span>
            <input class="uk-input uk-form-width-large" type="text" name="text" placeholder="Tweet"
                   required="required"/>
            <button class="uk-button uk-button-primary">Send</button>
        </form>
        <br/>
        <div class="uk-panel uk-panel-box" th:each="tweet : ${tweets}">
            <h3 class="uk-panel-title" th:text="${tweet.text}">Hello</h3>
            <span th:text="${tweet.username}">user</span> @ <span th:text="${tweet.createdAt}">2017-02-01</span>
            <hr class="uk-divider-icon"/>
        </div>
    </div>
    <div class="uk-width-1-5"></div>
</div>
</body>
</html>
```

Authorization Serverと連携してAuthorization Code Flowを有効にします。これにより、Authorization Server側で認証を行いAuthorization Codeを発行し、これとアクセストークンを交換し、Web UI側のHTTPセッションに保存できます。

Authorization Code Flowを有効にするためは、`@EnableOAuth2Sso`を付与すれば良いです。またHTTPセッション上のアクセストークンを使ってHTTPアクセスできるように`OAuth2RestTemplate`を定義します。

`src/main/java/com/example/TweeterWebuiApplication.java`を次のように記述してください。

``` java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.security.oauth2.client.EnableOAuth2Sso;
import org.springframework.context.annotation.Bean;
import org.springframework.security.oauth2.client.OAuth2ClientContext;
import org.springframework.security.oauth2.client.OAuth2RestTemplate;
import org.springframework.security.oauth2.client.resource.OAuth2ProtectedResourceDetails;

@SpringBootApplication
@EnableOAuth2Sso
public class TweeterWebuiApplication {

	public static void main(String[] args) {
		SpringApplication.run(TweeterWebuiApplication.class, args);
	}

	@Bean
	OAuth2RestTemplate oauth2RestTemplate(OAuth2ClientContext oauth2ClientContext,
										  OAuth2ProtectedResourceDetails details) {
		return new OAuth2RestTemplate(details, oauth2ClientContext);
	}
}
```

最後にOAuth2クライアントの情報を定義するため、`src/main/resources/application.properties`を次のように記述してください。

``` properties
tweeter.api.uri=http://localhost:8082/v1
tweeter.auth.uri=http://localhost:9999/uaa
security.oauth2.client.client-id=demo
security.oauth2.client.client-secret=demo
security.oauth2.client.scope=tweet.read,tweet.write
security.oauth2.client.access-token-uri=${tweeter.auth.uri}/oauth/token
security.oauth2.client.user-authorization-uri=${tweeter.auth.uri}/oauth/authorize
security.oauth2.resource.token-info-uri=${tweeter.auth.uri}/oauth/check_token
```

次のコマンドでこのWebアプリケーションのjarファイルを作成してください。

```
cd tweeter-webui
./mvnw clean package -DskipTests=true
```

### Webアプリケーションの動作確認

```
java -jar target/tweeter-webui-0.0.1-SNAPSHOT.jar
```

ブラウザで[http://localhost:8080](http://localhost:8080)にアクセスしてください。

ブラウザ側では認証が済んでいないため、Authorization Serverにリダイレクトされます。Authorization Server側では認証に関する設定を特にしていないためデフォルトのBasic認証が使われます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/73e833a4-da49-3798-4260-5e40f5cd648a.png)

ユーザー名とパスワードを入力してください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/f62068c1-b8f0-e298-a61f-94af78c87d07.png)


初回はWeb UIに`tweet.read`及び`tweet.write`スコープを許可するかどうかを選択する画面(OAuth Approval)が表示されます。
両方を許可(Approve)して、Authorizeボタンをクリックしてください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/7a97fceb-831f-eadd-7850-091a259572b5.png)


その後、Web UIにリダイレクトされ、Tweet一覧が表示されます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/74612f3f-f074-d605-5e8c-b4589fc466aa.png)


フォームにTweet本文を入力してSENDボタンを押してください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/e07ab0b0-ee35-d5a6-3622-491944c6b86e.png)

入力した内容が表示されます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/b3e55272-32d6-c8f0-64b6-469e804f81bd.png)

### Authorization Serverのログインフォームを作成


Authorization ServerがBasic認証しかサポートしていないのは使い勝手が悪いのでフォーム認証を追加します。

> **メモ**
> 
> 変更点は[こちら](https://github.com/tweeter-service/tweeter-auth/compare/master...login-form?diff=split)で確認できます。


![image](https://qiita-image-store.s3.amazonaws.com/0/1852/56b57ccb-19e8-46ab-5da4-d2ac508344e0.png)



`tweeter-auth`プロジェクトでこれから実装するファイルの空ファイルを次のコマンドで作成してください。

```
cd ../tweeter-auth/
touch src/main/java/com/example/AuthController.java
touch src/main/resources/templates/index.html
touch src/main/resources/templates/login.html
```

まずは`pom.xml`にThymeleafの依存関係を追加します。


``` xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

ログインフォームとログイン後のデフォルトページに遷移するためのControllerを実装します。
`src/main/java/com/example/AuthController.java`に次の内容を記述してください。

``` java
package com.example;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class AuthController {
	@GetMapping("/")
	String index() {
		return "index";
	}

	@GetMapping("login")
	String login() {
		return "login";
	}
}
```

ログインフォーム画面を作成します。
`src/main/resources/templates/login.html`に次の内容を記述してください。


``` html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title>Tweeter Auth</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/uikit/3.0.0-beta.11/css/uikit.min.css"/>
</head>
<body>
<div class="uk-grid">
    <div class="uk-width-1-5"></div>
    <div class="uk-width-3-5">
        <h1>Tweeter Login</h1>

        <p class="uk-text-danger" th:if="${param.error}">
            Login failed ...
        </p>

        <p class="uk-text-success" th:if="${param.logout}">
            Logout succeeded.
        </p>

        <form class="uk-form-stacked" method="post" th:action="@{/login}">
            <div class="uk-margin">
                <label class="uk-form-label" for="username">Username</label>
                <div class="uk-form-controls">
                    <input class="uk-input" id="username" type="text" name="username" value="user"/>
                </div>
            </div>
            <div class="uk-margin">
                <label class="uk-form-label" for="password">Password</label>
                <div class="uk-form-controls">
                    <input class="uk-input" id="password" type="password" name="password" value="password"/>
                </div>
            </div>
            <div class="uk-margin">
                <button class="uk-width-1-1 uk-button uk-button-primary uk-button-large">Login</button>
            </div>
        </form>
    </div>
    <div class="uk-width-1-5"></div>
</div>
</body>
</html>
```


ログイン後のデフォルト画面を作成します。
`src/main/resources/templates/index.html`に次の内容を記述してください。


``` html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title>Tweeter Auth</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/uikit/3.0.0-beta.11/css/uikit.min.css"/>
</head>
<body>
<div class="uk-grid">
    <div class="uk-width-1-5"></div>
    <div class="uk-width-3-5">
        <h1>Tweeter Auth</h1>
        <form th:action="@{/logout}" method="post">
            <button class="uk-button uk-button-default">Logout</button>
        </form>
    </div>
    <div class="uk-width-1-5"></div>
</div>
</body>
</html>
```

最後に、ログインフォームに関する認可情報を設定します。これはAuthorization Serverの認可設定とは別の設定です。

``` java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
// ここから(1/2)
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
// ここまで
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;

@SpringBootApplication
@EnableAuthorizationServer
public class TweeterAuthApplication {

	public static void main(String[] args) {
		SpringApplication.run(TweeterAuthApplication.class, args);
	}

	// ここから(2/2)
	@Configuration
	static class LoginConfig extends WebSecurityConfigurerAdapter {
		@Override
		protected void configure(HttpSecurity http) throws Exception {
			http.formLogin().loginPage("/login").permitAll().and().requestMatchers()
					.antMatchers("/", "/login", "/oauth/authorize",
							"/oauth/confirm_access") // [1]
					.and().authorizeRequests().anyRequest().authenticated();
		}
	}
	// ここまで
}
```

* [1] ... `/`, `/login`, `/oauth/authorize`, `/oauth/confirm_access`に対するアクセスのみ以下の認可制御を適用する


以上で`tweeter-auth`プロジェクトの変更は完了です。ビルドする前に次のチェックリストを確認してください。

- [ ] pom.xmlにThymeleafを追加した
- [ ] AuthControllerを実装した
- [ ] login.htmlを作成した
- [ ] index.htmlを作成した
- [ ] TweeterAuthApplication.javaを修正した


次のコマンドでこのAuthorization Serverのjarファイルを作成してください。

```
./mvnw clean package -DskipTests=true
```


次のコマンドでAuthorization Serverを起動してください。

```
java -jar target/tweeter-auth-0.0.1-SNAPSHOT.jar
```

ここで再度Web UI([http://localhost:8080](http://localhost:8080))にアクセスしてください。

今回はログインフォームにリダイレクトされます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/2f82aeac-ee3e-cadc-703b-d0293927b227.png)

OAuth Approval画面で`tweet.read`と`tweet.write`をApproveしてAuthorizeボタンを押してください（初回のみ）。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/3ba09aec-75e7-d53a-321c-ee864f754937.png)

> **メモ**
>
> OAuth Approval画面を上書きしたい場合は`/oauth/confirm_access`に対するControllerを作成してください。
> また特定のクライアントに対して特定のスコープを自動でApproveさせることができます。
>
> インメモリクライアントの場合、`tweeter-auth`プロジェクトの`application.properties`に次の設定を追加すれば良いです。
>
> ``` properties
> security.oauth2.client.client-id=demo
> security.oauth2.client.client-secret=demo
> security.oauth2.client.scope=openid,tweet.read,tweet.write
> security.oauth2.client.authorized-grant-types=password,authorization_code
> ## ↓この設定を追加
> security.oauth2.client.auto-approve-scopes=openid,tweet.read,tweet.write
> ```

これでログインできました。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/c2bcf606-9c86-d240-b3dd-b94f9bc97eb5.png)

Authorization Serverのログイン後画面([http://localhost:9999/uaa](http://localhost:9999/uaa))にアクセスすると、Authorization Serverからログアウトするボタンがあります。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/0f9b1938-743a-a33d-b563-81bc67015e54.png)

このログアウトはAuthorization ServerのHTTPセッションからログアウトすることを意味し、Web UIのHTTPセッションはログアウトしないことに注意してください。
Web UIからログアウトする場合は、Web UI側でもログアウトボタンを用意する必要があります。

Web UIからログアウトした場合に、Authorization ServerのHTTPセッションが残っていれば、次にWeb UIにログインしようとした際にログインフォームは表示されず、そのままWeb UIのTOPページにリダイレクトされます。この仕組みを利用すると別のWeb UI(OAuth2クライアント)が増えた場合でも毎回ログインフォームを入力することなくログインできてSingle Sign-Onのように見えます。これが`@EnableOAuth2Sso`アノテーションの名前につながっています。


### ログインユーザー情報の詳細を取得するエンドポイントを追加

Authorization ServerとWeb UIの連携に

``` properties
security.oauth2.resource.token-info-uri=${tweeter.auth.uri}/oauth/check_token
```
が使用されていました。`/check_token`エンドポントでは次のような情報を取得できます。

```
$ curl -X GET -u demo:demo http://localhost:9999/uaa/oauth/check_token?token=4fc79a8d-8ac0-44db-8fb7-4a8b69c92d29
{"exp":1487484898,"user_name":"user","authorities":["ROLE_USER","ROLE_ACTUATOR"],"client_id":"demo","scope":["openid","tweet.read","tweet.write"]}
```

今回はインメモリなユーザーを使用しているため、この情報でも十分ですが、通常はユーザー情報をカスタマイズしますので、この情報だと不十分なことが多いです。
その場合は任意のユーザー情報取得エンドポイントを用意して、そのエンドポントと連携するようにすれば良いです。

ここではAuthorization Serverに`/userinfo`エンドポントを用意して、返却する情報を増やします。

> **メモ**
>
> 変更点は[こちら](https://github.com/tweeter-service/tweeter-auth/compare/login-form...userinfo?diff=split)で確認できます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/56b57ccb-19e8-46ab-5da4-d2ac508344e0.png)

作成するファイルの空ファイルを次のコマンドで作成してください。

```
touch src/main/java/com/example/UserinfoController.java
```

まずは`/userinfo`エンドポントに対するエンドポントを作成します。`name`、`authorities`フィールドに加えて、`firstName`、`lastName`、`email`フィールドを追加します。この情報はインメモリなユーザーには含まれないので、一時的にハードコードします。実際はデータベースからユーザー(`UserDetails`)を取得する場合はこれらの情報が含まれている想定です。

`src/main/java/com/example/UserinfoController.java`に次の内容を記述してください。


``` java
package com.example;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.stream.Collectors;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserinfoController {

	@GetMapping("userinfo")
	Object userinfo(Authentication authentication) {
		Map<String, Object> userinfo = new LinkedHashMap<>();
		userinfo.put("name", authentication.getName());
		userinfo.put("authorities", authentication.getAuthorities().stream()
				.map(GrantedAuthority::getAuthority).collect(Collectors.toList()));
		userinfo.put("firstName", "John");
		userinfo.put("lastName", "Doe");
		userinfo.put("email", "jdoe@example.com");
		return userinfo;
	}
}
```


このエンドポントにアクセスするための認可設定をAuthorization Server側に設定します。このエンドポントにはアクセストークンを用いてアクセスするため、Autorization Server自体もResource Serverになります。`src/main/java/com/example/TweeterAuthApplication.java`に次の内容を記述してください。


``` java

package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Configuration;
// ここから追加(1/4)
import org.springframework.http.HttpMethod;
// ここまで追加
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
// ここから追加(2/4)
import org.springframework.security.config.http.SessionCreationPolicy;
// ここまで追加
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
// ここから追加(3/4)
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
// ここまで追加

@SpringBootApplication
@EnableAuthorizationServer
public class TweeterAuthApplication {

	public static void main(String[] args) {
		SpringApplication.run(TweeterAuthApplication.class, args);
	}

	@Configuration
	static class LoginConfig extends WebSecurityConfigurerAdapter {
		@Override
		protected void configure(HttpSecurity http) throws Exception {
			http.formLogin().loginPage("/login").permitAll().and().requestMatchers()
					.antMatchers("/", "/login", "/oauth/authorize",
							"/oauth/confirm_access")
					.and().authorizeRequests().anyRequest().authenticated();
		}
	}

	// ここから追加(4/4)
	@Configuration
	@EnableResourceServer // [1]
	static class ResourceServerConfig extends ResourceServerConfigurerAdapter {
		@Override
		public void configure(HttpSecurity http) throws Exception {
			http.sessionManagement()
					.sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
					.authorizeRequests().antMatchers(HttpMethod.GET, "/userinfo")
					.access("#oauth2.hasScope('openid')"); // [2]
		}
	}
	// ここまで追加
}
```

* [1] ... `/userinfo`へのアクセスを守るために`@EnableResourceServer`を付与する。 
* [2] ... `/userinfo`へのアクセスには`openid`スコープが必要とする。



次のコマンドでこのAuthorization Serverのjarファイルを作成してください。

```
./mvnw clean package -DskipTests=true
```

次のコマンドでAuthorization Serverを起動してください。

```
java -jar target/tweeter-auth-0.0.1-SNAPSHOT.jar
```

動作確認しましょう。

`openid`スコープを含むアクセストークンを発行してください。

```
$ curl -X POST -u demo:demo -d grant_type=password -d username=user -d password=password http://localhost:9999/uaa/oauth/token
{"access_token":"d271c238-f841-45e8-8c13-b46a3d00cce1","token_type":"bearer","expires_in":43199,"scope":"openid tweet.read tweet.write"}
```

このアクセストークンを使って`/userinfo`エンドポントにアクセスしてください。

```
$ curl -X GET -H 'Authorization: Bearer d271c238-f841-45e8-8c13-b46a3d00cce1' http://localhost:9999/uaa/userinfo
{"name":"user","authorities":["ROLE_ACTUATOR","ROLE_USER"],"firstName":"John","lastName":"Doe","email":"jdoe@example.com"}
```

ログインユーザーの詳細を取得できました。

次にWeb UIの設定を変更してこの`/userinfo`をアクセスするようにします。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/6e64bbf3-7426-1346-cb6b-2fe78edccd5a.png)


> **メモ**
>
> 変更点は[こちら](https://github.com/tweeter-service/tweeter-webui/compare/master...userinfo?diff=split)で確認できます。

`tweeter-webui`の`application.properties`を次のように設定してください。

``` properties

tweeter.api.uri=http://localhost:8082/v1
tweeter.auth.uri=http://localhost:9999/uaa
security.oauth2.client.client-id=demo
security.oauth2.client.client-secret=demo
## ↓ openidスコープを追加
security.oauth2.client.scope=tweet.read,tweet.write,openid
security.oauth2.client.access-token-uri=${tweeter.auth.uri}/oauth/token
security.oauth2.client.user-authorization-uri=${tweeter.auth.uri}/oauth/authorize
## ↓ token-info-uriからuser-info-uriに変更
security.oauth2.resource.user-info-uri=${tweeter.auth.uri}/userinfo
```

次のコマンドでWeb UIのjarファイルを作成してください。

```
cd ../tweeter-webui
./mvnw clean package -DskipTests=true
```

次のコマンドでWeb UIを起動してください。

```
java -jar target/tweeter-webui-0.0.1-SNAPSHOT.jar
```

Web UI([http://localhost:8080](http://localhost:8080))にアクセスしてください

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/4c2b1e70-1707-5262-98f4-0d216020c8c4.png)

これまでと同様にWeb UIとAuthorization Serverが連携できています。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/96e817ff-838c-190a-1f18-f8e6ac069b87.png)


実はこの設定だけではログインユーザー情報は`/oauth/check_token`とほぼ変わりません(むしろ、`scope`が取れなくなります)。`security.oauth2.resource.user-info-uri`を指定した場合はユーザー情報の取得に[`PrincipalExtractor`](https://github.com/spring-projects/spring-boot/blob/v1.5.1.RELEASE/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/security/oauth2/resource/PrincipalExtractor.java)、権限情報の取得に[`AuthoritiesExtractor.`](https://github.com/spring-projects/spring-boot/blob/v1.5.1.RELEASE/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/security/oauth2/resource/AuthoritiesExtractor.java)が使用されます。
デフォルトでは次の実装が使用されます。

* [`FixedPrincipalExtractor`](https://github.com/spring-projects/spring-boot/blob/v1.5.1.RELEASE/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/security/oauth2/resource/FixedPrincipalExtractor.java)
* [`FixedAuthoritiesExtractor`](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/security/oauth2/resource/FixedAuthoritiesExtractor.java)

`FixedPrincipalExtractor`は`/userinfo`のレスポンスJSON中に`name`フィールドを見つけるとそれを認証ユーザー情報として返却します。

> **メモ**
>
> `/oauth/check_token`を使う場合は[`org.springframework.security.oauth2.provider.token.RemoteTokenServices`](https://github.com/spring-projects/spring-security-oauth/blob/2.0.12.RELEASE/spring-security-oauth2/src/main/java/org/springframework/security/oauth2/provider/token/RemoteTokenServices.java)が使用され、`/usefinfo`を使う場合は[`org.springframework.boot.autoconfigure.security.oauth2.resource.UserInfoTokenServices`](https://github.com/spring-projects/spring-boot/blob/v1.5.1.RELEASE/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/security/oauth2/resource/UserInfoTokenServices.java)が使用されます。


### `PrincipalExtractor`の実装

`/userinfo`のレスポンスを利用してWeb UI側で任意のログインユーザーを作成したい場合は`PrincipalExtractor`を実装する必要があります。

`/userinfo`のレスポンスに対応する`TweeterUser`クラスを作成し、`TweeterUser`インスタンスを返却する`TweeterPrincipalExtractor`を実装しましょう。


![image](https://qiita-image-store.s3.amazonaws.com/0/1852/6e64bbf3-7426-1346-cb6b-2fe78edccd5a.png)

> **メモ**
>
> 変更点は[こちら](https://github.com/tweeter-service/tweeter-webui/compare/userinfo...extractor?diff=split)で確認できます。

作成するファイルの空ファイルを次のコマンドで作成してください。

```
touch src/main/java/com/example/TweeterUser.java
touch src/main/java/com/example/TweeterPrincipalExtractor.java
```

まずはログインユーザークラスを作成します。`src/main/java/com/example/TweeterUser.java`に次の内容を記述してください。

``` java
package com.example;

import java.io.Serializable;

public class TweeterUser implements Serializable {
	private final String username;
	private final String firstName;
	private final String lastName;
	private final String email;

	public TweeterUser(String username, String firstName, String lastName, String email) {
		this.username = username;
		this.firstName = firstName;
		this.lastName = lastName;
		this.email = email;
	}

	public String getUsername() {
		return username;
	}

	public String getFirstName() {
		return firstName;
	}

	public String getLastName() {
		return lastName;
	}

	public String getEmail() {
		return email;
	}
}
```

次にこの`TweeterUser`インスタンスを返却する`TweeterPrincipalExtractor`を実装します。

``` java

package com.example;

import java.util.Map;

import org.springframework.boot.autoconfigure.security.oauth2.resource.PrincipalExtractor;
import org.springframework.stereotype.Component;

@Component
public class TweeterPrincipalExtractor implements PrincipalExtractor {

	@Override
	public Object extractPrincipal(Map<String, Object> map) {
		return new TweeterUser(map.get("name").toString(),
				map.get("firstName").toString(), map.get("lastName").toString(),
				map.get("email").toString());
	}
}
```

このログインユーザー情報を画面に反映しましょう。Contollerの引数に`@AuthenticationPrincipal`アノテーションをつけることでログインユーザーを取得できます。
このユーザー情報を`Model`に追加してください。

`src/main/java/com/example/TweeterController.java`に、次の内容を記述してください。

``` java

package com.example;

import java.net.URI;
import java.util.Collections;
import java.util.Date;
import java.util.List;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.RequestEntity;
// ここから追加(1/5)
import org.springframework.security.core.annotation.AuthenticationPrincipal;
// ここまで
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

@Controller
public class TweeterController {
	private final RestTemplate restTemplate;
	private final URI tweeterApiUri;

	public TweeterController(RestTemplate restTemplate,
			@Value("${tweeter.api.uri}") URI tweeterApiUri) {
		this.restTemplate = restTemplate;
		this.tweeterApiUri = tweeterApiUri;
	}

	@GetMapping("/")
	// ここから追加(2/5)
	String timelines(Model model, @AuthenticationPrincipal TweeterUser user) {
	// ここまで
		RequestEntity<?> request = RequestEntity.get(UriComponentsBuilder
				.fromUri(tweeterApiUri).pathSegment("timelines").build().toUri()).build();
		List<Tweet> tweets = restTemplate
				.exchange(request, new ParameterizedTypeReference<List<Tweet>>() {
				}).getBody();
		model.addAttribute("tweets", tweets);
		// ここから追加(3/5)
		model.addAttribute("user", user);
		// ここまで
		return "tweets";
	}

	@GetMapping("/tweets")
	// ここから追加(4/5)
	String tweets(Model model, @AuthenticationPrincipal TweeterUser user) {
	// ここまで
		RequestEntity<?> request = RequestEntity.get(UriComponentsBuilder
				.fromUri(tweeterApiUri).pathSegment("tweets").build().toUri()).build();
		List<Tweet> tweets = restTemplate
				.exchange(request, new ParameterizedTypeReference<List<Tweet>>() {
				}).getBody();
		model.addAttribute("tweets", tweets);
		// ここから追加(5/5)
		model.addAttribute("user", user);
		// ここまで
		return "tweets";
	}

	@PostMapping("/tweets")
	String tweets(@RequestParam String text) {
		RequestEntity<?> request = RequestEntity.post(UriComponentsBuilder
				.fromUri(tweeterApiUri).pathSegment("tweets").build().toUri())
				.body(Collections.singletonMap("text", text));
		restTemplate.exchange(request, Void.class);
		return "redirect:/";
	}

	public static class Tweet {
		public String tweetId;
		public String username;
		public String text;
		public Date createdAt;
	}
}
```

最後に画面でこのユーザー情報を表示しましょう。

`src/main/resources/templates/tweets.html`に次の内容を記述してください。

``` html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title>Tweeter</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/uikit/3.0.0-beta.11/css/uikit.min.css"/>
</head>

<body>

<div class="uk-grid">
    <div class="uk-width-1-5"></div>
    <div class="uk-width-3-5">
        <nav class="uk-navbar-container" uk-navbar="">
            <div class="uk-navbar-left">
                <ul class="uk-navbar-nav">
                    <li th:classappend="${#httpServletRequest.requestURI == '/'} ? 'uk-active'"><a
                            th:href="@{/}">Home</a></li>
                    <li th:classappend="${#httpServletRequest.requestURI == '/tweets'} ? 'uk-active'"><a
                            th:href="@{/tweets}">Tweets</a>
                    </li>
                </ul>
            </div>
        </nav>
        <h1>Tweeter</h1>
        <form th:action="@{/tweets}" class="uk-panel uk-panel-box uk-form" method="post">
            <!-- ここから変更 -->
            <span th:text="${user.firstName + ' ' + user.lastName + '@' + user.username}">user</span>
            <!-- ここまで -->
            <input class="uk-input uk-form-width-large" type="text" name="text" placeholder="Tweet"
                   required="required"/>
            <button class="uk-button uk-button-primary">Send</button>
        </form>
        <br/>
        <div class="uk-panel uk-panel-box" th:each="tweet : ${tweets}">
            <h3 class="uk-panel-title" th:text="${tweet.text}">Hello</h3>
            <span th:text="${tweet.username}">user</span> @ <span th:text="${tweet.createdAt}">2017-02-01</span>
            <hr class="uk-divider-icon"/>
        </div>
    </div>
    <div class="uk-width-1-5"></div>
</div>
</body>
</html>
```

以上で`tweeter-webui`プロジェクトの変更は完了です。ビルドする前に次のチェックリストを確認してください。

- [ ] TweeterUserを実装した
- [ ] TweeterPrincipalExtractorを実装した
- [ ] TweeterControllerを変更した
- [ ] tweets.htmlを変更した

次のコマンドでWeb UIのjarファイルを作成してください。

```
./mvnw clean package -DskipTests=true
```

次のコマンドでWeb UIを起動してください。

```
java -jar target/tweeter-webui-0.0.1-SNAPSHOT.jar
```

Web UI([http://localhost:8080](http://localhost:8080))にアクセスしてください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/5fe92717-e803-c8bf-0e64-8ddb2f8c8105.png)

投稿フォームの左にFirst NameとLast Nameが表示されていることが確認できます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/92c98498-2f65-f720-6c10-f0b608c3f58c.png)


以上でWeb UIの作成は完了です。
