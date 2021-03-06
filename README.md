<p align="center">
  <img src="https://pac4j.github.io/pac4j/img/logo-j2e.png" width="300" />
</p>

The `j2e-pac4j` project is an **easy and powerful security library for J2E** web applications which supports authentication and authorization, but also application logout and advanced features like session fixation and CSRF protection.
It's based on Java 8, servlet 3 and on the **[pac4j security engine](https://github.com/pac4j/pac4j)**. It's available under the Apache 2 license.

**Main concepts:**

1) A [**client**](https://github.com/pac4j/pac4j/wiki/Clients) represents an authentication mechanism (CAS, OAuth, SAML, OpenID Connect, LDAP, JWT...) It performs the login process and returns a user profile. An indirect client is for UI authentication while a direct client is for web services authentication

2) An [**authorizer**](https://github.com/pac4j/pac4j/wiki/Authorizers) is meant to check authorizations on the authenticated user profile(s) (role / permission, ...) or on the current web context (IP check, CSRF...)

3) The `SecurityFilter` protects an url by checking that the user is authenticated and that the authorizations are checked, according to the clients and authorizers configuration. If the user is not authenticated, it performs authentication for direct clients or starts the login process for indirect clients

4) The `CallbackFilter` finishes the login process for an indirect client.

Just follow these easy steps to secure your J2E web application:


### 1) Add the required dependencies (`j2e-pac4j` + `pac4j-*` libraries)

You need to add a dependency on:
 
- the `j2e-pac4j` library (<em>groupId</em>: **org.pac4j**, *version*: **1.3.0-SNAPSHOT**)
- the appropriate `pac4j` [submodules](https://github.com/pac4j/pac4j/wiki/Clients) (<em>groupId</em>: **org.pac4j**, *version*: **1.9.0-SNAPSHOT**): `pac4j-oauth` for OAuth support (Facebook, Twitter...), `pac4j-cas` for CAS support, `pac4j-ldap` for LDAP authentication, etc.

All released artifacts are available in the [Maven central repository](http://search.maven.org/#search%7Cga%7C1%7Cpac4j).


### 2) Define the configuration (`Config` + `Client` + `Authorizer`)

The configuration (`org.pac4j.core.config.Config`) contains all the clients and authorizers required by the application to handle security.

It must be built via a configuration factory (`org.pac4j.core.config.ConfigFactory`):

```java
public class DemoConfigFactory implements ConfigFactory {

  public Config build() {
    GoogleOidcClient oidcClient = new GoogleOidcClient();
    oidcClient.setClientID("id");
    oidcClient.setSecret("secret");
    oidcClient.addCustomParam("prompt", "consent");

    SAML2ClientConfiguration cfg = new SAML2ClientConfiguration("resource:samlKeystore.jks",
                "pac4j-demo-passwd", "pac4j-demo-passwd", "resource:testshib-providers.xml");
    cfg.setMaximumAuthenticationLifetime(3600);
    cfg.setServiceProviderEntityId("urn:mace:saml:pac4j.org");
    cfg.setServiceProviderMetadataPath("sp-metadata.xml");
    SAML2Client saml2Client = new SAML2Client(cfg);

    FacebookClient facebookClient = new FacebookClient("fbId", "fbSecret");
    TwitterClient twitterClient = new TwitterClient("twId", "twSecret");

    FormClient formClient = new FormClient("http://localhost:8080/theForm.jsp", new SimpleTestUsernamePasswordAuthenticator());
    IndirectBasicAuthClient basicAuthClient = new IndirectBasicAuthClient(new SimpleTestUsernamePasswordAuthenticator());

    CasClient casClient = new CasClient("http://mycasserver/login");

    ParameterClient parameterClient = new ParameterClient("token", new JwtAuthenticator("salt"));

    Config config = new Config("http://localhost:8080/callback", oidcClient, saml2Client, facebookClient,
                                  twitterClient, formClient, basicAuthClient, casClient, parameterClient);

    config.addAuthorizer("admin", new RequireAnyRoleAuthorizer("ROLE_ADMIN"));
    config.addAuthorizer("custom", new CustomAuthorizer());

    return config;
  }
}
```

`http://localhost:8080/callback` is the url of the callback endpoint, which is only necessary for indirect clients.

Notice that you can define:

1) a specific [`SessionStore`](https://github.com/pac4j/pac4j/wiki/SessionStore) using the `setSessionStore(sessionStore)` method (by default, the `J2ESessionStore` uses the HTTP session)

2) specific [matchers](https://github.com/pac4j/pac4j/wiki/Matchers) via the `addMatcher(name, Matcher)` method.

If your application is configured via dependency injection, no factory is required to build the configuration, you can directly inject the `Config` via the appropriate setter.


### 3) Protect urls (`SecurityFilter`)

You can protect (authentication + authorizations) the urls of your J2E application by using the `SecurityFilter` and defining the appropriate mapping. The following parameters are available:

1) `configFactory`: the factory to initialize the configuration. By default, the configuration is shared across filters so it can be specificied only once, but each filter can defined its own configuration if necessary

2) `clients` (optional): the list of client names (separated by commas) used for authentication:
- in all cases, this filter requires the user to be authenticated. Thus, if the `clients` is blank or not defined, the user must have been previously authenticated
- if the user is not authenticated, the defined direct clients are tried successively to login the user
- then if the user is still not authenticated and if the first defined client is indirect, this client is used to redirect the user to the appropriate identity provider for login
- otherwise, a 401 HTTP error is returned
- if the `client_name` request parameter is provided, only this client (if it exists in the `clients`) is selected.

3) `authorizers` (optional): the list of authorizer names (separated by commas) used to check authorizations:
- if the `authorizers` is blank or not defined, no authorization is checked
- if the authorization checks fail, a 403 HTTP error is returned
- the following authorizers are available by default (without defining them in the configuration):
  * `isFullyAuthenticated` to check if the user is authenticated but not remembered, `isRemembered` for a remembered user, `isAnonymous` to ensure the user is not authenticated, `isAuthenticated` to ensure the user is authenticated (not necessary by default unless you use the `AnonymousClient`)
  * `hsts` to use the `StrictTransportSecurityHeader` authorizer, `nosniff` for `XContentTypeOptionsHeader`, `noframe` for `XFrameOptionsHeader `, `xssprotection` for `XSSProtectionHeader `, `nocache` for `CacheControlHeader ` or `securityHeaders` for the five previous authorizers
  * `csrfToken` to use the `CsrfTokenGeneratorAuthorizer` with the `DefaultCsrfTokenGenerator` (it generates a CSRF token and saves it as the `pac4jCsrfToken` request attribute and in the `pac4jCsrfToken` cookie), `csrfCheck` to check that this previous token has been sent as the `pac4jCsrfToken` header or parameter in a POST request and `csrf` to use both previous authorizers.

4) `matchers` (optional): the list of matcher names (separated by commas) that the request must satisfy to check authentication / authorizations:
- if the `matchers` is blank or not defined, all requests are checked.

5) `multiProfile` (optional): it indicates whether multiple authentications (and thus multiple profiles) must be kept at the same time (`false` by default).

6) `renewSession` (optional): it indicates whether the web session must be renewed after login, to avoid session hijacking (`true` by default).

In the `web.xml` file:

```xml
<filter>
  <filter-name>FacebookAdminFilter</filter-name>
  <filter-class>org.pac4j.j2e.filter.SecurityFilter</filter-class>
  <init-param>
    <param-name>configFactory</param-name>
    <param-value>org.pac4j.demo.j2e.DemoConfigFactory</param-value>
  </init-param>
  <init-param>
    <param-name>clients</param-name>
    <param-value>FacebookClient</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>FacebookAdminFilter</filter-name>
  <url-pattern>/facebook/*</url-pattern>
</filter-mapping>
```

This filter can be defined via dependency injection as well. In that case, these parameters will be defined via setters.


### 4) Define the callback endpoint only for indirect clients (`CallbackFilter`)

For indirect clients (like Facebook), the user is redirected to an external identity provider for login and then back to the application.
Thus, a callback endpoint is required in the application. It is managed by the `CallbackFilter`. The following parameters are available:

1) `configFactory` (optional): the factory to initialize the configuration. By default, the configuration is shared across filters so it can be specificied only once, but each filter can defined its own configuration if necessary

2) `defaultUrl` (optional): it's the default url after login if no url was originally requested (`/` by default)

3) `multiProfile` (optional): it indicates whether multiple authentications (and thus multiple profiles) must be kept at the same time (`false` by default).

In the `web.xml` file:

```xml
<filter>
  <filter-name>callbackFilter</filter-name>
  <filter-class>org.pac4j.j2e.filter.CallbackFilter</filter-class>
  <init-param>
    <param-name>defaultUrl</param-name>
    <param-value>/</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>callbackFilter</filter-name>
  <url-pattern>/callback</url-pattern>
</filter-mapping>
```

Using dependency injection via Spring, you can define the callback filter as a `DelegatingFilterProxy` in the `web.xml` file:

```xml
<filter>
  <filter-name>callbackFilter</filter-name>
  <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
  <filter-name>callbackFilter</filter-name>
  <url-pattern>/callback</url-pattern>
</filter-mapping>
```
    
and the specific bean in the `application-context.xml` file:

```xml
<bean id="callbackFilter" class="org.pac4j.j2e.filter.CallbackFilter">
  <property name="defaultUrl" value="/" />
</bean>
```


### 5) Get the user profile (`ProfileManager`)

You can get the profile of the authenticated user using `profileManager.get(true)` (`false` not to use the session, but only the current HTTP request).
You can test if the user is authenticated using `profileManager.isAuthenticated()`.
You can get all the profiles of the authenticated user (if ever multiple ones are kept) using `profileManager.getAll(true)`.

Example:

```java
WebContext context = new J2EContext(request, response);
ProfileManager manager = new ProfileManager(context);
Optional<CommonProfile> profile = manager.get(true);
```

The retrieved profile is at least a `CommonProfile`, from which you can retrieve the most common attributes that all profiles share. But you can also cast the user profile to the appropriate profile according to the provider used for authentication. For example, after a Facebook authentication:

```java
FacebookProfile facebookProfile = (FacebookProfile) commonProfile;
```


### 6) Logout (`ApplicationLogoutFilter`)

You can log out the current authenticated user using the `ApplicationLogoutFilter`.
After logout, the user is redirected to the url defined by the `url` request parameter if it matches the `logoutUrlPattern`.
Or the user is redirected to the `defaultUrl` if it is defined. Otherwise, a blank page is displayed.
The following parameters are available:

1) `defaultUrl` (optional): the default logout url if no `url` request parameter is provided or if the `url` does not match the `logoutUrlPattern` (not defined by default)

2) `logoutUrlPattern` (optional): the logout url pattern that the `url` parameter must match (only relative urls are allowed by default).

In the `web.xml` file:

```xml
<filter>
  <filter-name>logoutFilter</filter-name>
  <filter-class>org.pac4j.j2e.filter.ApplicationLogoutFilter</filter-class>
  <init-param>
    <param-name>defaultUrl</param-name>
    <param-value>/urlAfterLogout</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>logoutFilter</filter-name>
  <url-pattern>/logout</url-pattern>
</filter-mapping>
```


## Migration guide

### 1.2 - > 1.3

The `RequiresAuthenticationFilter` is now named `SecurityFilter` with the `clients`, `authorizers` and `matchers` parameters instead of the previous `clientName`, `authorizerName` and `matcherName`.

The `ApplicationLogoutFilter` behaviour has slightly changed: even without any `url` request parameter, the user will be redirect to the `defaultUrl` if it has been defined.

### 1.1 -> 1.2

Authorizations are now handled by the library so the `ClientFactory` can now longer be used and is replaced by a `ConfigFactory` which builds a `Config` which gathers clients (for authentication) and authorizers (for authorizations).

The `isAjax` parameter is no longer available as AJAX requests are now automatically detected. The `stateless` parameter is no longer available as the stateless nature is held by the client itself.

The `requireAnyRole` and `requieAllRoles` parameters are no longer available and authorizers must be used instead (with the `authorizerName` parameter).

The application logout process can be managed with the `ApplicationLogoutFilter`.


## Demo

The demo webapp: [j2e-pac4j-demo](https://github.com/pac4j/j2e-pac4j-demo) is available for tests and implement many authentication mechanisms: Facebook, Twitter, form, basic auth, CAS, SAML, OpenID Connect, JWT...


## Release notes

See the [release notes](https://github.com/pac4j/j2e-pac4j/wiki/Release-Notes). Learn more by browsing the [j2e-pac4j Javadoc](http://www.javadoc.io/doc/org.pac4j/j2e-pac4j/1.3.0) and the [pac4j Javadoc](http://www.pac4j.org/apidocs/pac4j/1.9.0/index.html).


## Need help?

If you have any question, please use the following mailing lists:

- [pac4j users](https://groups.google.com/forum/?hl=en#!forum/pac4j-users)
- [pac4j developers](https://groups.google.com/forum/?hl=en#!forum/pac4j-dev)


## Development

The version 1.3.0-SNAPSHOT is under development.

Maven artifacts are built via Travis: [![Build Status](https://travis-ci.org/pac4j/j2e-pac4j.png?branch=master)](https://travis-ci.org/pac4j/j2e-pac4j) and available in the [Sonatype snapshots repository](https://oss.sonatype.org/content/repositories/snapshots/org/pac4j). This repository must be added in the Maven `pom.xml` file for example:

```xml
<repositories>
  <repository>
    <id>sonatype-nexus-snapshots</id>
    <name>Sonatype Nexus Snapshots</name>
    <url>https://oss.sonatype.org/content/repositories/snapshots</url>
    <releases>
      <enabled>false</enabled>
    </releases>
    <snapshots>
      <enabled>true</enabled>
    </snapshots>
  </repository>
</repositories>
```
