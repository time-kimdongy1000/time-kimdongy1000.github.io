---

title: Spring Secuirty 22 OAuth2 ClientRegistration 
author: kimdongy1000
date: 2023-07-02 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , OAuth2 ]
math: true
mermaid: true

---

우리는 지난시간에 KeyClock 의 연동과 간단한 용어 정리 그리고 개념에 대해서 공부를 해보았다 용어는 앞으로 반복 그리고 새로운것이 계속 나올것이기에 그때마다 정리를 하도록 하겠습니다 오늘부터는 어떻게 Spring - Security 가 KeyClocak 을 연동하고 최종적으로 사용자 자원을 가져오는지에 대해서 알아볼려고 합니다 

## ClientRegitsrtaion

시작하기전에 새로운 용어 ClientRegitsrtaion 에 대해서 알아보도록 하겠습니다 클라이언트에 Oauth2 정보 또는 클라이언트 정보를 저장소입니다 
이곳에 우리는 우리 클라이언트 정보와 연동할려고 하는 KeyClocak 의 정보를 심어두고 시큐리티는 이 ClientRegitsrtaion 정보를 바탕으로 필요한 요청정보를 만들어 
인가서버와 통신을 하게 됩니다 


## SecuirtyConfig 설정 

```

package com.cybb.main.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

        httpSecurity.authorizeRequests().anyRequest().authenticated();
        httpSecurity.oauth2Login();

        return httpSecurity.build();
    }
}



```

간단하게 작성을 하고 우리는 oauth2Login 디버그를 잡고 안에서 무슨일이 일어나는지 알아볼 예정입니다 

## OAuth2ClientRegistrationRepositoryConfiguration 

```

@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(OAuth2ClientProperties.class)
@Conditional(ClientsConfiguredCondition.class)
class OAuth2ClientRegistrationRepositoryConfiguration {

	@Bean
	@ConditionalOnMissingBean(ClientRegistrationRepository.class)
	InMemoryClientRegistrationRepository clientRegistrationRepository(OAuth2ClientProperties properties) {
		List<ClientRegistration> registrations = new ArrayList<>(
				OAuth2ClientPropertiesRegistrationAdapter.getClientRegistrations(properties).values());
		return new InMemoryClientRegistrationRepository(registrations);
	}

}


```
OAuth2ClientRegistrationRepositoryConfiguration 는 우리가 앞에서 application.properties 에서 정의한 keyClocak 연동값을 시큐리티에 저장을 하는 클래스 입니다 

이때 returnType InMemoryClientRegistrationRepository 있으며 즉 연동 결과를 메모리에 저장하는 로직입니다 

그리고 파라미터로 넘어오는 OAuth2ClientProperties properties 에서는 

```

result = {OAuth2ClientProperties@5080} 
 provider = {HashMap@5081}  size = 1
  "keycloak" -> {OAuth2ClientProperties$Provider@5087} 
   key = "keycloak"
   value = {OAuth2ClientProperties$Provider@5087} 
    authorizationUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
    tokenUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token"
    userInfoUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo"
    userInfoAuthenticationMethod = null
    userNameAttribute = "preferred_username"
    jwkSetUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/certs"
    issuerUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project"
 registration = {HashMap@5082}  size = 1
  "keycloak" -> {OAuth2ClientProperties$Registration@5099} 
   key = "keycloak"
   value = {OAuth2ClientProperties$Registration@5099} 
    provider = null
    clientId = "Spring-Oauth2-Authorizaion-client"
    clientSecret = "NIe2qftuPcclGWFiBFicEWoK5SfYs7ql"
    clientAuthenticationMethod = "client_secret_post"
    authorizationGrantType = "authorization_code"
    redirectUri = "http://localhost:8081/login/oauth2/code/keycloak"
    scope = {LinkedHashSet@5107}  size = 2
    clientName = "Spring-Oauth2-Authorizaion-client"

```

우리가 앞에서 선언한 값들이 의존에 의해서 넘어오는것을 확인할 수 있습니다 

## OAuth2ClientPropertiesRegistrationAdapter 

```

public final class OAuth2ClientPropertiesRegistrationAdapter {
    
    ....

    private static ClientRegistration getClientRegistration(String registrationId,
            OAuth2ClientProperties.Registration properties, Map<String, Provider> providers) {
        Builder builder = getBuilderFromIssuerIfPossible(registrationId, properties.getProvider(), providers);
        if (builder == null) {
            builder = getBuilder(registrationId, properties.getProvider(), providers);
        }
        PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
        map.from(properties::getClientId).to(builder::clientId);
        map.from(properties::getClientSecret).to(builder::clientSecret);
        map.from(properties::getClientAuthenticationMethod).as(ClientAuthenticationMethod::new)
                .to(builder::clientAuthenticationMethod);
        map.from(properties::getAuthorizationGrantType).as(AuthorizationGrantType::new)
                .to(builder::authorizationGrantType);
        map.from(properties::getRedirectUri).to(builder::redirectUri);
        map.from(properties::getScope).as(StringUtils::toStringArray).to(builder::scope);
        map.from(properties::getClientName).to(builder::clientName);
        return builder.build();
    }

}


```

여기서 보면 OAuth2ClientPropertiesRegistrationAdapter 클래스는 앞에서 넘어오는 properties 값들을 넣어주는데 사실 

Builder builder = getBuilderFromIssuerIfPossible(registrationId, properties.getProvider(), providers); 만 호출이 되더라도 

```

result = {ClientRegistration$Builder@5385} 
 registrationId = "keycloak"
 clientId = null
 clientSecret = null
 clientAuthenticationMethod = {ClientAuthenticationMethod@5386} 
 authorizationGrantType = {AuthorizationGrantType@5387} 
 redirectUri = "{baseUrl}/{action}/oauth2/code/{registrationId}"
 scopes = null
 authorizationUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
 tokenUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token"
 userInfoUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo"
 userInfoAuthenticationMethod = {AuthenticationMethod@5392} 
 userNameAttributeName = "preferred_username"
 jwkSetUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/certs"
 issuerUri = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project"
 configurationMetadata = {LinkedHashMap@5396}  size = 52
 clientName = "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project"

```

이미 필요한 값들은 다 들어가 있는것을 알 수 있다 현재 이곳은 Client 그리고 Provider 에 대한 데이터는 이미 다들어가 있는것을 확인할 수 있다 이런 원리는 사실 
우리가 앞에서 본 Properties 값 중에 

spring.security.oauth2.client.provider.keycloak.issuerUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project

우리 지난시간에 issuer 는 이는 ID 토큰 액세스 토큰을 발급하는 주체를 뜻하며 현재 클라이언트가 사용하는 인증서버의 엔드포인트입니다 이렇게 설명을 했습니다 
즉 이 하나의 uri 만 가지고 있어도 시큐리티는 우리가 다른 설정 authorizationUri , tokenUri 등 설정을 해줄 필요가 없습니다 시큐리티는 이 주소만 읽고도 
Provier 가 필요한 모든 값들을 읽어 올 수 있습니다 


```

private static Builder getBuilderFromIssuerIfPossible(String registrationId, String configuredProviderId,
			Map<String, Provider> providers) {
		String providerId = (configuredProviderId != null) ? configuredProviderId : registrationId;
		if (providers.containsKey(providerId)) {
			Provider provider = providers.get(providerId);
			String issuer = provider.getIssuerUri();
			if (issuer != null) {
				Builder builder = ClientRegistrations.fromIssuerLocation(issuer).registrationId(registrationId);
				return getBuilder(builder, provider);
			}
		}
		return null;
	}

```

이 부분에서 provider 값과 issuer 값을 분리해내서 fromIssuerLocation 을 호출하게 됩니다 

```

public static ClientRegistration.Builder fromIssuerLocation(String issuer) {
    Assert.hasText(issuer, "issuer cannot be empty");
    URI uri = URI.create(issuer);
    return getBuilder(issuer, oidc(uri), oidcRfc8414(uri), oauth(uri));
}

```

여기서 url http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project 을 가리키는데 이는 계속해서 앞에서 말하는 issurUri 를 가져오게 되는것이고 

getBuilder 호출시

```

private static Supplier<ClientRegistration.Builder> oidc(URI issuer) {
    // @formatter:off
    URI uri = UriComponentsBuilder.fromUri(issuer)
            .replacePath(issuer.getPath() + OIDC_METADATA_PATH)
            .build(Collections.emptyMap());
    // @formatter:on
    return () -> {
        RequestEntity<Void> request = RequestEntity.get(uri).build();
        Map<String, Object> configuration = rest.exchange(request, typeReference).getBody();
        OIDCProviderMetadata metadata = parse(configuration, OIDCProviderMetadata::parse);
        ClientRegistration.Builder builder = withProviderConfiguration(metadata, issuer.toASCIIString())
                .jwkSetUri(metadata.getJWKSetURI().toASCIIString());
        if (metadata.getUserInfoEndpointURI() != null) {
            builder.userInfoUri(metadata.getUserInfoEndpointURI().toASCIIString());
        }
        return builder;
    };
}



```

이 부분으로 넘어오게 되는데 이곳에서 `RequestEntity<Void> request = RequestEntity.get(uri).build();` 요청정보를 만들고 

`Map<String, Object> configuration = rest.exchange(request, typeReference).getBody();` 요청이 들어가고 그것을 body 를 추출해서 

Map 타입의 설정정보를 만들어내게 되는데 

## configuration 
```

configuration = {LinkedHashMap@6558}  size = 53
 "issuer" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project"
 "authorization_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth"
 "token_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token"
 "introspection_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/token/introspect"
 "userinfo_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/userinfo"
 "end_session_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/logout"
 "frontchannel_logout_session_supported" -> {Boolean@6629} true
 "frontchannel_logout_supported" -> {Boolean@6629} true
 "jwks_uri" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/certs"
 "check_session_iframe" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/login-status-iframe.html"
 "grant_types_supported" -> {ArrayList@6636}  size = 7
 "acr_values_supported" -> {ArrayList@6638}  size = 2
 "response_types_supported" -> {ArrayList@6640}  size = 8
 "subject_types_supported" -> {ArrayList@6642}  size = 2
 "id_token_signing_alg_values_supported" -> {ArrayList@6644}  size = 12
 "id_token_encryption_alg_values_supported" -> {ArrayList@6646}  size = 3
 "id_token_encryption_enc_values_supported" -> {ArrayList@6648}  size = 6
 "userinfo_signing_alg_values_supported" -> {ArrayList@6650}  size = 13
 "userinfo_encryption_alg_values_supported" -> {ArrayList@6652}  size = 3
 "userinfo_encryption_enc_values_supported" -> {ArrayList@6654}  size = 6
 "request_object_signing_alg_values_supported" -> {ArrayList@6656}  size = 13
 "request_object_encryption_alg_values_supported" -> {ArrayList@6658}  size = 3
 "request_object_encryption_enc_values_supported" -> {ArrayList@6660}  size = 6
 "response_modes_supported" -> {ArrayList@6662}  size = 7
 "registration_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/clients-registrations/openid-connect"
 "token_endpoint_auth_methods_supported" -> {ArrayList@6666}  size = 5
 "token_endpoint_auth_signing_alg_values_supported" -> {ArrayList@6668}  size = 12
 "introspection_endpoint_auth_methods_supported" -> {ArrayList@6670}  size = 5
 "introspection_endpoint_auth_signing_alg_values_supported" -> {ArrayList@6672}  size = 12
 "authorization_signing_alg_values_supported" -> {ArrayList@6674}  size = 12
 "authorization_encryption_alg_values_supported" -> {ArrayList@6676}  size = 3
 "authorization_encryption_enc_values_supported" -> {ArrayList@6678}  size = 6
 "claims_supported" -> {ArrayList@6680}  size = 10
 "claim_types_supported" -> {ArrayList@6682}  size = 1
 "claims_parameter_supported" -> {Boolean@6629} true
 "scopes_supported" -> {ArrayList@6685}  size = 10
 "request_parameter_supported" -> {Boolean@6629} true
 "request_uri_parameter_supported" -> {Boolean@6629} true
 "require_request_uri_registration" -> {Boolean@6629} true
 "code_challenge_methods_supported" -> {ArrayList@6690}  size = 2
 "tls_client_certificate_bound_access_tokens" -> {Boolean@6629} true
 "revocation_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/revoke"
 "revocation_endpoint_auth_methods_supported" -> {ArrayList@6695}  size = 5
 "revocation_endpoint_auth_signing_alg_values_supported" -> {ArrayList@6697}  size = 12
 "backchannel_logout_supported" -> {Boolean@6629} true
 "backchannel_logout_session_supported" -> {Boolean@6629} true
 "device_authorization_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/auth/device"
 "backchannel_token_delivery_modes_supported" -> {ArrayList@6703}  size = 2
 "backchannel_authentication_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/ext/ciba/auth"
 "backchannel_authentication_request_signing_alg_values_supported" -> {ArrayList@6707}  size = 9
 "require_pushed_authorization_requests" -> {Boolean@6709} false
 "pushed_authorization_request_endpoint" -> "http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project/protocol/openid-connect/ext/par/request"
 "mtls_endpoint_aliases" -> {LinkedHashMap@6713}  size = 8

```

아래처럼 이렇게 되는것입니다 

그런정보를 

```

ClientRegistration.Builder builder = withProviderConfiguration(metadata, issuer.toASCIIString())
        .jwkSetUri(metadata.getJWKSetURI().toASCIIString());
if (metadata.getUserInfoEndpointURI() != null) {
    builder.userInfoUri(metadata.getUserInfoEndpointURI().toASCIIString());
}

```

withProviderConfiguration 부분을 통해서 

```

private static ClientRegistration.Builder withProviderConfiguration(AuthorizationServerMetadata metadata,
        String issuer) {
    String metadataIssuer = metadata.getIssuer().getValue();
    Assert.state(issuer.equals(metadataIssuer),
            () -> "The Issuer \"" + metadataIssuer + "\" provided in the configuration metadata did "
                    + "not match the requested issuer \"" + issuer + "\"");
    String name = URI.create(issuer).getHost();
    ClientAuthenticationMethod method = getClientAuthenticationMethod(metadata.getTokenEndpointAuthMethods());
    Map<String, Object> configurationMetadata = new LinkedHashMap<>(metadata.toJSONObject());
    // @formatter:off
    return ClientRegistration.withRegistrationId(name)
            .userNameAttributeName(IdTokenClaimNames.SUB)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .clientAuthenticationMethod(method)
            .redirectUri("{baseUrl}/{action}/oauth2/code/{registrationId}")
            .authorizationUri((metadata.getAuthorizationEndpointURI() != null) ? metadata.getAuthorizationEndpointURI().toASCIIString() : null)
            .providerConfigurationMetadata(configurationMetadata)
            .tokenUri(metadata.getTokenEndpointURI().toASCIIString())
            .issuerUri(issuer)
            .clientName(issuer);
    // @formatter:on
}

```

이곳에서 나머지 필요한 모든 값들을 설정을 하게 됩니다 이렇게 해서 ClientRegitsrtaion 이 하나 만들어지게 됩니다 그럼 사실 우리는 앞에서 


```

server.port=8081

spring.security.oauth2.client.registration.keycloak.clientId=Spring-Oauth2-Authorizaion-client
spring.security.oauth2.client.registration.keycloak.clientSecret=NIe2qftuPcclGWFiBFicEWoK5SfYs7ql
spring.security.oauth2.client.registration.keycloak.redirectUri=http://localhost:8081/login/oauth2/code/keycloak
spring.security.oauth2.client.registration.keycloak.scope=email,profile
spring.security.oauth2.client.registration.keycloak.clientName=Spring-Oauth2-Authorizaion-client
spring.security.oauth2.client.registration.keycloak.authorizationGrantType=authorization_code
spring.security.oauth2.client.registration.keycloak.clientAuthenticationMethod=client_secret_post

spring.security.oauth2.client.provider.keycloak.issuerUri=http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project



```

이렇게 client 부분은 client 설정이기때문에 고정 Provider 부분은 사실상 issuerUri 있으면 연동이 되는것이다 그래서 주석잡고나 또는 지우고 진행을 해도 알아서 연동을 하게 됩니다 실제 모든 정보는 http://localhost:8080/realms/Srping-Oauth2-Authorizaion-Project 와 통신해서 전부 가져올 수 있기 때문입니다 

오늘은 기본적으로 properties 에 등록된 정보를 어떻게 시큐리티가 가져와서 연동을 하게 되는지에 대해서 알아보았습니다 앞으로는 이런식으로 계속 글이 쓰여질 예정입니다 









