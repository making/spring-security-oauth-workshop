# Spring Security OAuth Workshop

Twitter風サービス(Tweeter)を作成して、OAuth 2の基本、および[Spring Security OAuth](https://projects.spring.io/spring-security-oauth/)を[Spring Bootで使う方法](http://docs.spring.io/spring-boot/docs/1.5.1.RELEASE/reference/html/boot-features-security.html#boot-features-security-oauth2)を学びます。


![image](https://qiita-image-store.s3.amazonaws.com/0/1852/a3d0d4e0-f22b-2f01-5b4b-36df828630f2.png)

1. [Resource Server (REST API)の準備](resource-server.md)
1. [Authorization Serverの作成](authorization-server.md)
1. [CUIアプリケーション (Resource Owner Password Credentials)の作成](cli-application.md)
1. [Web UIアプリケーション (Authorization Code)の作成](webui-application.md)
1. Web UIアプリケーション (Implicit)の作成 ([サンプルコードのみ](https://github.com/tweeter-service/tweeter-spa/blob/master/src/main/resources/static/index.html))
1. [Cloud Foundryにデプロイ](deploy-to-cloud-foundry.md)
1. Authorization ServerのUserをデータベースで管理 (TODO)
1. Authorization ServerのClientをデータベースで管理 (TODO)
1. JWTに対応 (TODO)
1. Zuul連携 (TODO)

> **注意**
>
> Spring SecurityのOAuth対応はSpring Security 5でをコアに含まれる予定です([spring-security#3907](https://github.com/spring-projects/spring-security/issues/3907))。
> 
> Spring Security OAuth自体には今後積極的な機能追加(OpenID Connect対応など)は行われないと思われます。

### 利用規約

無断で本ドキュメントの一部または全部を改変したり、本ドキュメントを用いた二次的著作物を作成することを禁止します。ただし、ドキュメント修正のためのPull Requestは大歓迎です。
