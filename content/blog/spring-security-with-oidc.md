+++
author = "陈 sir"
categories = ["安全"]
tags = ["spring security","oauth2","oidc"]
date = "2019-09-25"
description = "oidc spring security"
featured = "main/title.png"
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "spring security 与 keyclaok 集成，使用 oidc 登录"
type = "post"

+++
# 核心代码
securityConfiguration.java 文件
``` java
package com.mycompany.gateway.config;

import com.mycompany.gateway.security.*;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import com.mycompany.gateway.security.oauth2.AudienceValidator;
import com.mycompany.gateway.security.SecurityUtils;
import org.springframework.security.oauth2.core.DelegatingOAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2TokenValidator;
import org.springframework.security.oauth2.jwt.*;
import com.mycompany.gateway.security.oauth2.AuthorizationHeaderFilter;
import com.mycompany.gateway.security.oauth2.AuthorizationHeaderUtil;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.mapping.GrantedAuthoritiesMapper;
import org.springframework.security.oauth2.core.oidc.user.OidcUserAuthority;
import java.util.*;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;
import org.springframework.security.web.csrf.CsrfFilter;
import com.mycompany.gateway.security.oauth2.JwtAuthorityExtractor;
import org.springframework.security.web.header.writers.ReferrerPolicyHeaderWriter;
import org.springframework.web.filter.CorsFilter;
import org.zalando.problem.spring.web.advice.security.SecurityProblemSupport;

@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
@Import(SecurityProblemSupport.class)
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    private final CorsFilter corsFilter;
// 这里的key，需要和application.yaml 中的 issuer-uri 的key对应
    @Value("${spring.security.oauth2.client.provider.keycloak.issuer-uri}")
    private String issuerUri;

    private final JwtAuthorityExtractor jwtAuthorityExtractor;
    private final SecurityProblemSupport problemSupport;

    public SecurityConfiguration(CorsFilter corsFilter, JwtAuthorityExtractor jwtAuthorityExtractor, SecurityProblemSupport problemSupport) {
        this.corsFilter = corsFilter;
        this.problemSupport = problemSupport;
        this.jwtAuthorityExtractor = jwtAuthorityExtractor;
    }

    @Override
    public void configure(WebSecurity web) {
        web.ignoring()
            .antMatchers(HttpMethod.OPTIONS, "/**")
            .antMatchers("/app/**/*.{js,html}")
            .antMatchers("/i18n/**")
            .antMatchers("/content/**")
            .antMatchers("/h2-console/**")
            .antMatchers("/swagger-ui/index.html")
            .antMatchers("/test/**");
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        http
            .csrf()
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        .and()
            .addFilterBefore(corsFilter, CsrfFilter.class)
            .exceptionHandling()
            .accessDeniedHandler(problemSupport)
        .and()
            .headers()
            .contentSecurityPolicy("default-src 'self'; frame-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://storage.googleapis.com; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self' data:")
        .and()
            .referrerPolicy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
        .and()
            .featurePolicy("geolocation 'none'; midi 'none'; sync-xhr 'none'; microphone 'none'; camera 'none'; magnetometer 'none'; gyroscope 'none'; speaker 'none'; fullscreen 'self'; payment 'none'")
        .and()
            .frameOptions()
            .deny()
        .and()
            .authorizeRequests()
            .antMatchers("/api/auth-info").permitAll()
            .antMatchers("/api/**").authenticated()
            .antMatchers("/management/health").permitAll()
            .antMatchers("/management/info").permitAll()
            .antMatchers("/management/prometheus").permitAll()
            .antMatchers("/management/**").hasAuthority(AuthoritiesConstants.ADMIN)
        .and()
//            启用oauth2
            .oauth2Login()
        .and()
            .oauth2ResourceServer()
                .jwt()
                .jwtAuthenticationConverter(jwtAuthorityExtractor)
                .and()
            .and()
                .oauth2Client();
        // @formatter:on
    }

    /**
     * Map authorities from "groups" or "roles" claim in ID Token.
     *
     * @return a {@link GrantedAuthoritiesMapper} that maps groups from
     * the IdP to Spring Security Authorities.
     */
    @Bean
    public GrantedAuthoritiesMapper userAuthoritiesMapper() {
        return (authorities) -> {
            Set<GrantedAuthority> mappedAuthorities = new HashSet<>();

            authorities.forEach(authority -> {
                OidcUserAuthority oidcUserAuthority = (OidcUserAuthority) authority;
                mappedAuthorities.addAll(SecurityUtils.extractAuthorityFromClaims(oidcUserAuthority.getUserInfo().getClaims()));
            });
            return mappedAuthorities;
        };
    }

    @Bean
    JwtDecoder jwtDecoder() {
        NimbusJwtDecoderJwkSupport jwtDecoder = (NimbusJwtDecoderJwkSupport)
            JwtDecoders.fromOidcIssuerLocation(issuerUri);

        OAuth2TokenValidator<Jwt> audienceValidator = new AudienceValidator();
        OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer(issuerUri);
        OAuth2TokenValidator<Jwt> withAudience = new DelegatingOAuth2TokenValidator<>(withIssuer, audienceValidator);

        jwtDecoder.setJwtValidator(withAudience);

        return jwtDecoder;
    }

    @Bean
    public AuthorizationHeaderFilter authHeaderFilter(AuthorizationHeaderUtil headerUtil) {
        return new AuthorizationHeaderFilter(headerUtil);
    }
}
```


application.yaml

``` yaml
spring:
  security:
    oauth2:
      client:
        provider:
        # 可以配置多个provider
          keycloak:
          # issuer-uri 是keycloak realm 的地址，模式为http://localhost:9000/auth/realms/<realm-name>
            issuer-uri: http://localhost:9000/auth/realms/jhipster
        registration:
        # registration 下的 keyclaok 会生成一个 http://<yourserviceip>:<port>/oauth2/authorization/keycloak 的地址,
        #浏览器访问这个地址会被重定向到 keycloak 的登录地址。如果把keycloak 改为oidc 那么生成的地址就为 http://<yourserviceip>:<port>/oauth2/authorization/keycloak 
        # 可以配置多个 registration，对应的配置类 OAuth2ClientProperties
          keycloak:
            provider: keycloak
            client-id: web_app
            client-secret: web_app
```


# 认证流程


如果你访问一个未授权的资源，spring securety 会强制重定向到
`http://localhost:8080/oauth2/authorization/keycloak`（假设应用在8080端口启动）
`OAuth2LoginAuthenticationFilter` 会监听 `oauth2/authorization` 这个 url 模式，并重定向到对应的 identity provider，在本例中就是配置的 `keycloak` 所以完整的地址就是`http://localhost:8080/oauth2/authorization/keycloak`

> 如果你配置了多个identity provider，并且想要选择使用哪个identity provider 进行登录，spring security 在启用 oauth 2 之后会在在 `http://<serverip>:<port>/login` URL 上生成一个默认页面，该页面会列出所有的可用 oauth2 provider

{{< fancybox path="/img/main/" file="loginpage.jpg" caption="loginpage.jpg" gallery="Gallery Name" >}}

列出的identity provider 会显示 identity provider的 `issuer-uri` 地址,而实际的连接地址是
`http://<yourserviceip>:<port>/oauth2/authorization/<provider>` 这个地址是在跳转到keycloak 注册的realm的登录地址。

当在 keycloak 完成登录后，keycloak 会重定向到 client 配置的redirect url(此处配置为localhost:*)，并在后面追加 `/login/oauth2/code/<provider>` 因为认证流程中的过滤器`OAuth2LoginAuthenticationFilter` 监听 `/login/oauth2/code/*` 这个地址

``` java
public class OAuth2LoginAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    public static final String DEFAULT_FILTER_PROCESSES_URI = "/login/oauth2/code/*";
```

`OAuth2LoginAuthenticationFilter` 会使用 `OidcAuthorizationCodeAuthenticationProvider` 作为认证代理进行相关认证。
`OidcAuthorizationCodeAuthenticationProvider` 主要做了以下三件事情
1. 将 AuthorizationCode 换成 token
2. 验证id token
3. 通过 user info endpoint 获取用户信息， user info endpoint 可以通过 identity provider 的 well known configuration 地址查看 对keycloak 的 realm 来说，就是
`http://localhost:9000/auth/realms/<realm-name>/.well-known/openid-configuration`

# 定制化
## 1. 修改 OAuth2LoginAuthenticationFilter 的监听地址

十分常见的一个场景是，修改 `OAuth2LoginAuthenticationFilter` 的redirect 监听地址。比如想要将 `/login/oauth2/code/*` 这个地址，修改为 `/oauth/callback/*` 修改代码如下

``` java
 @Override
    public void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .oauth2Login()
                .redirectionEndpoint()
                .baseUri("/oauth/callback/*")

    }
```
## 2. 获取当前账户信息
``` java
(OAuth2AuthenticationToken) SecurityContextHolder.getContext().getAuthentication()
```
或者
``` java
 @GetMapping("/me")
 public ResponseEntity<OAuth2AuthenticationToken> hello(currentUser: OAuth2AuthenticationToken)  {
        return ResponseEntity.ok(currentUser)
}
```
## 3. 获取 token 相关信息
在openid connect 认证流程中，应用将获得三种token `access token`，`id token`，`refresh token`，`OAuth2AuthorizedClientService` 保存了三种token的信息，可以通过如下方式获取。
``` java
 val currentUser = SecurityContextHolder.getContext().authentication as OAuth2AuthenticationToken
 val currentUserClientConfig = oAuth2AuthorizedClientService.loadAuthorizedClient(
                authorizedClientRegistrationId,
                currentUser.name)
  println("AccessToken: ${currentUserClientConfig.accessToken.tokenValue}")
  println("RefreshToken: ${currentUserClientConfig.refreshToken.tokenValue}")
```
## 4 刷新token
当token 过期 resource server 会返回401，最简单的处理方法就是抛出 `OAuth2AuthorizationException`，这样会自动触发登录流程。但是我们也可以通过程序自动刷新token



## 5. 使用 curl 命令 从 keycloak 获取 token
前置条件，client 必须启用 direct access grant

![image.png](https://upload-images.jianshu.io/upload_images/5120230-66ee56ab0306d906.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---



``` bash
curl -X POST -u "<client_id>:<client_secret>" http://localhost:9000/auth/realms/<realm-name>/protocol/openid-connect/token -d "grant_type=password&username=<keycloak-username-default-use-admin>&password=<openstack>&scope=openid email microprofile-jwt offline_access phone"
```

---

> 当 client type 为 public 和 bear token 的时候，是没有client secret的，此时 -u 参数只填写client id 即可。 例如 -u "web_app:"，冒号不能省略

[参考文档 官方](https://www.keycloak.org/docs/latest/server_admin/index.html#_oidc-auth-flows)
[参考文档 curl 获取token](https://backstage.forgerock.com/knowledge/kb/article/a45882528)

[参考文档 Spring boot + Spring Security 5 + OAuth2/OIDC Client - Basics](https://dev.to/shyamala_u/spring-boot--spring-security-5--oauth2oidc-client---basics-4ibo)
[参考文档 Spring boot + Spring Security 5 + OAuth2/OIDC Client - Deep Dive](https://dev.to/shyamala_u/spring-boot--spring-security-5--oauth2oidc-client---deep-dive-261l)

