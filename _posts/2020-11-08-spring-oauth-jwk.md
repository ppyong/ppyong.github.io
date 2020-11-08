---
layout: post
title: oAuth2 JWK 
---

현재 Spring 의 Autorization Server는 Deprecated 되었지만 다른 대안을 아직 찾지 못했기에 조금 더 사용할 것 같아서 자료를 찾다가 JWK Endpoint를 인증서버에 만들던 중 기록을 남깁니다. 

oAuth를 통해 인증을 처리할 때 매번 요청에 따라 데이터베이스를 조회해서 토큰에 따른 사용자 정보를 가져올 수 있지만 그에 따른 데이터베이스 부하를 줄이고자 JWT를 사용하게 됩니다. 

이 Json web token(이하 JWT)의 경우 일반적으로 JWS를 말하는데 이 JWS의 경우 "header, payload, signature" 세가지 값을 "." 으로 구분하고 있습니다. 

리소스서버에서는 API요청에서 header에 넘어오는 토큰의 유효성을 체크하고 그에 따라 응답 혹은 에러 메시지를 반환하게 됩니다. 

인증 서버와 리소스 서버가 한 곳에 개발된 형태라면 한 어플리케이션 내에서 토큰도 생성하고 토큰의 유효성도 체크할 수 있지만 Spring에서 인증 서버를 더이상 지원 안한다하여 앞으로를 대비하기 위해 분리를 고려할 수 밖에 없었습니다.

리소스 서버에서 JWT 토큰의 유효성을 확인하기 위해 인증 서버의 JWK Endpoint를 통해 받은 Json 정보를 사용하게 되는데 그에 따라 인증 서버에서는 JWK Endpoint를 추가합니다. 아래 소스 설정은 그에 대한 내용 입니다. 


```java
@Configuration
@Import(AuthorizationServerEndpointsConfiguration.class)
@RequiredArgsConstructor
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    private final AuthenticationManager authenticationManager;
    private final PasswordEncoder passwordEncoder;

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
		endpoints
			.authenticationManager(this.authenticationManager)
			.accessTokenConverter(accessTokenConverter())
			.tokenStore(tokenStore());
	}

    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        return converter;
    }

    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients
            .inMemory()
                .withClient("cline")
                .secret(passwordEncoder.encode("password"))
                .scopes("read")
                .authorizedGrantTypes("password");
    }

    @Bean
	public KeyPair keyPairBean() {
        KeyPair keyPair = new KeyStoreKeyFactory(new ClassPathResource("server.jks"), "password".toCharArray())
                .getKeyPair("key1", "password1".toCharArray());
	  	return keyPair;
	}
}
```

기존 인증 서버 설정에서는 @EnableAuthorizationServer 어노테이션을 통해 설정 하였던 부분을 해당 어노테이션을 제거하고 직접 AuthorizationServerEndpointsConfiguration 설정을 import 받았습니다. 

@EnableAuthorizationServer 의 내용을 보면 아래와 같이 되어있고 이 중 AuthorizationServerSecurityConfiguration 설정을 재정의 하기 위해 @EnableAuthorizationServer 어노테이션을 사용하지 않기로 하였습니다. 

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AuthorizationServerEndpointsConfiguration.class, AuthorizationServerSecurityConfiguration.class})
@Deprecated
public @interface EnableAuthorizationServer {

}
```
그에 따라 AuthorizationServerSecurityConfiguration 설정을 따로 해줄 필요가 있었고 아래와 같이 설정을 추가하였습니다. 

```java
@Configuration
public class JwkConfiguration extends AuthorizationServerSecurityConfiguration {
    @Override
	protected void configure(HttpSecurity http) throws Exception {
		super.configure(http);
		http
			.requestMatchers()
				.mvcMatchers("/jwks.json")
				.and()
			.authorizeRequests()
				.mvcMatchers("/jwks.json").permitAll();
	}
}
```

설정 후 JWK Endpoint는 아래와 같이 설정하였습니다.

```java
@RestController
@RequiredArgsConstructor
public class JwkEndpoint {
    private final KeyPair keyPair;

	@GetMapping("/jwks.json")
	@ResponseBody
	public Map<String, Object> getKey(Principal principal) {
		RSAPublicKey publicKey = (RSAPublicKey) this.keyPair.getPublic();
		RSAKey key = new RSAKey.Builder(publicKey).build();
		return new JWKSet(key).toJSONObject();
	}
}
```

해당 Endpoint를 통해 리소스 서버에서는 JWT의 유효성을 검증할 수 있었습니다. 