## CUIアプリケーション (Resource Owner Password Credentials)の作成

まずは簡単なOAuth2クライアントアプリケーションとしてCUIアプリケーションを作成します。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/92da7ca3-cca6-0a42-90d2-6f22f40018a3.png)

ここではResource Owner Password Credentials(`grant_type=password`)のフローを使用します。

> **メモ**
>
> Resource Owner Password Credentialsはユーザー名・パスワードとアクセストークンを交換する方式であるため
> クライアントアプリケーションが直接ユーザー名・パスワードに触れる機会があります。
> 通常、この方式はAuthorization Server提供元の公式アプリケーションで使用され、信用できないクライアントには許可すべきではありません。



### プロジェクトの作成

次のコマンドを実行すると、`tweeter-cli`フォルダに雛形プロジェクトが生成されます

```
curl start.spring.io/starter.tgz \
       -d artifactId=tweeter-cli \
       -d baseDir=tweeter-cli \
       -d dependencies=web,configuration-processor \
       -d applicationName=TweeterCliApplication | tar -xzvf -
```


### CUIアプリケーションの作成

すでにcurlコマンドを使って動作確認してきたように、OAuth2に対応したREST APIのアクセスは簡単なHTTPプロトコルで実現可能です。
今回のCUIアプリケーションでは特にSpring Securityは使用せず、HTTPクライアントである`RestTemplate`のみを使用します。

`src/main/java/com/example/TweeterCliApplication.java`を次のように記述してください。

``` java
package com.example;

import static org.springframework.http.RequestEntity.get;
import static org.springframework.http.RequestEntity.post;

import java.net.URI;
import java.util.Collections;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.util.Base64Utils;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.util.StringUtils;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

import com.fasterxml.jackson.databind.JsonNode;

@SpringBootApplication
@ConfigurationProperties(prefix = "cli")
public class TweeterCliApplication implements CommandLineRunner {
	private URI accessTokenUri;
	private String clientId;
	private String clientSecret;
	private URI tweetApiUri;
	private String username;
	private String password;
	private String text;

	public static void main(String[] args) {
		SpringApplication.run(TweeterCliApplication.class, args);
	}

	@Override
	public void run(String... args) throws Exception {
		RestTemplate restTemplate = new RestTemplateBuilder().build();
		if (getUsername() == null) {
			System.out.print("Enter username : ");
			setUsername(System.console().readLine());
		}
		if (getPassword() == null) {
			System.out.print("Enter password : ");
			setPassword(String.valueOf(System.console().readPassword()));
		}
		if (getText() == null) {
			System.out.print("Enter text : ");
			setText(System.console().readLine());
		}

		// Get Access Token
		MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
		body.add("grant_type", "password");
		body.add("username", getUsername());
		body.add("password", getPassword());
		body.add("scope", "tweet.read tweet.write");

		JsonNode token = restTemplate.exchange(post(getAccessTokenUri())
				.header("Authorization",
						"Basic " + Base64Utils.encodeToString(
								(getClientId() + ":" + getClientSecret()).getBytes()))
				.body(body), JsonNode.class).getBody();
		String accessToken = token.get("access_token").asText();

		if (!StringUtils.isEmpty(getText())) {
			// Post Tweet
			restTemplate
					.exchange(
							post(UriComponentsBuilder.fromUri(getTweetApiUri())
									.pathSegment("tweets").build().toUri())
											.header("Authorization",
													"Bearer " + accessToken)
											.body(Collections.singletonMap("text",
													getText())),
							JsonNode.class);
		}

		// Get Timelines
		JsonNode timelines = restTemplate.exchange(
				get(UriComponentsBuilder.fromUri(getTweetApiUri())
						.pathSegment("timelines").build().toUri())
								.header("Authorization", "Bearer " + accessToken).build(),
				JsonNode.class).getBody();

		timelines.forEach(tweet -> {
			System.out.println("=========");
			System.out.println("Name: " + tweet.get("username").asText());
			System.out.println("Text: " + tweet.get("text").asText());
			System.out.println("Date: " + tweet.get("createdAt").asText());
		});
	}

	public URI getAccessTokenUri() {
		return accessTokenUri;
	}

	public void setAccessTokenUri(URI accessTokenUri) {
		this.accessTokenUri = accessTokenUri;
	}

	public String getClientId() {
		return clientId;
	}

	public void setClientId(String clientId) {
		this.clientId = clientId;
	}

	public String getClientSecret() {
		return clientSecret;
	}

	public void setClientSecret(String clientSecret) {
		this.clientSecret = clientSecret;
	}

	public URI getTweetApiUri() {
		return tweetApiUri;
	}

	public void setTweetApiUri(URI tweetApiUri) {
		this.tweetApiUri = tweetApiUri;
	}

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public String getText() {
		return text;
	}

	public void setText(String text) {
		this.text = text;
	}
}
```

クライアント認証情報やAuthorization Serverの情報を`src/main/resources/application.properties`に定義します。

``` properties
cli.client-id=demo
cli.client-secret=demo
cli.access-token-uri=http://localhost:9999/uaa/oauth/token
cli.tweet-api-uri=http://localhost:8082/v1
spring.main.web-environment=false
spring.main.banner-mode=off
logging.level.root=off
```

次のコマンドでこのCLIアプリケーションのjarファイルを作成してください。

```
cd tweeter-cli
./mvnw clean package -DskipTests=true
```

### CLIアプリケーションの動作確認

次のコマンドでCLIアプリケーションを起動し、ユーザー名、パスワード、Tweet本文を入力してください。

```
$ java -jar target/tweeter-cli-0.0.1-SNAPSHOT.jar
Enter username : user
Enter password : password
Enter text : Hello from CLI
=========
Name: user
Text: Hello from CLI
Date: 2017-02-18T19:20:35Z
=========
Name: user
Text: Hello World!
Date: 2017-02-18T18:55:13Z
```

ユーザー名、パスワード、Tweet本文はプログラム引数でも指定可能です。

```
$ java -jar target/tweeter-cli-0.0.1-SNAPSHOT.jar --cli.username=user --cli.password=password --cli.text="Hello OAuth2"
=========
Name: user
Text: Hello OAuth2
Date: 2017-02-18T19:22:21Z
=========
Name: user
Text: Hello from CLI
Date: 2017-02-18T19:20:35Z
=========
Name: user
Text: Hello World!
Date: 2017-02-18T18:55:13Z
```

----

Tweetの投稿と一覧表示ができるCLIアプリケーションを作成しました。

次にWebアプリケーションを作成しましょう。
