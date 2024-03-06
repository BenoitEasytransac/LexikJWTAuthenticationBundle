# Getting started

## Prerequisites

This bundle requires Symfony 6.4+ and ext-openssl.

**Protip:** Though the bundle doesn't enforce you to do so, it is highly
recommended to use HTTPS.

## Installation

Add
`lexik/jwt-authentication-bundle](https://packagist.org/packages/lexik/jwt-authentication-bundle>)
to your ``composer.json`` file:

```
$ php composer.phar require "lexik/jwt-authentication-bundle"
```

### Register the bundle

Register bundle into ``config/bundles.php`` (Flex did it automatically):

```
return [
   //...
   Lexik\Bundle\JWTAuthenticationBundle\LexikJWTAuthenticationBundle::class => ['all' => true],
];
```

### Generate the SSL keys

>   For the version 2.xx of this bundle, you can use Web-Token_ and generate
>   JSON Web Keys (``JWK``) and JSON Web Keysets (``JWKSet``) instead of
>   PEM encoded keys.

>   Please refer to the dedicated page [Web-Token feature](./10-web-token.md) for
>   more information.

```
$ php bin/console lexik:jwt:generate-keypair
```

Your keys will land in ``config/jwt/private.pem`` and
``config/jwt/public.pem`` (unless you configured a different path).

Available options:

-  ``--skip-if-exists`` will silently do nothing if keys already exist.
-  ``--overwrite`` will overwrite your keys if they already exist.

Otherwise, an error will be raised to prevent you from overwriting your
keys accidentally.

## Configuration

Configure the SSL keys path and passphrase in your ``.env``:

```
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
JWT_PASSPHRASE=
```

```
# config/packages/lexik_jwt_authentication.yaml
lexik_jwt_authentication:
   secret_key: '%env(resolve:JWT_SECRET_KEY)%' # required for token creation
   public_key: '%env(resolve:JWT_PUBLIC_KEY)%' # required for token verification
   pass_phrase: '%env(JWT_PASSPHRASE)%' # required for token creation
   token_ttl: 3600 # in seconds, default is 3600
```

### Configure application security

> [!CAUTION]
> Make sure the firewall ``login`` is place before ``api``, and if
> ``main`` exists, put it after ``api``, otherwise you will encounter
> ``/api/login_check`` route not found.

```
# config/packages/security.yaml
security:
    enable_authenticator_manager: true # Only for Symfony 5.4
    # ...

    firewalls:
        login:
            pattern: ^/api/login
            stateless: true
            json_login:
                check_path: /api/login_check
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure

        api:
            pattern:   ^/api
            stateless: true
            jwt: ~

    access_control:
        - { path: ^/api/login, roles: PUBLIC_ACCESS }
        - { path: ^/api,       roles: IS_AUTHENTICATED_FULLY }
```

### Configure application routing

```
# config/routes.yaml
api_login_check:
    path: /api/login_check
```

### API Platform compatibility

If [API Platform](https://api-platform.com/) is detected, the integration will be done with your security configuration.

If you wish to change some parameters, you can do it with this configuration:

```
# config/packages/lexik_jwt_authentication.yaml
lexik_jwt_authentication:
   # ...
   api_platform:
       check_path: /api/login_check
       username_path: email
       password_path: security.credentials.password
```

## Usage

### 1. Obtain the token

The first step is to authenticate the user using its credentials.
You can test getting the token with a simple curl command like this
(adapt host and port):

Linux or macOS:

```
$ curl -X POST -H "Content-Type: application/json" https://localhost/api/login_check -d '{"username":"johndoe","password":"test"}'
```

Windows:

```
C:\> curl -X POST -H "Content-Type: application/json" https://localhost/api/login_check --data {\"username\":\"johndoe\",\"password\":\"test\"}
```

If it works, you will receive something like this:

```
{
    "token" : "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXUyJ9.eyJleHAiOjE0MzQ3Mjc1MzYsInVzZXJuYW1lIjoia29ybGVvbiIsImlhdCI6IjE0MzQ2NDExMzYifQ.nh0L_wuJy6ZKIQWh6OrW5hdLkviTs1_bau2GqYdDCB0Yqy_RplkFghsuqMpsFls8zKEErdX5TYCOR7muX0aQvQxGQ4mpBkvMDhJ4-pE4ct2obeMTr_s4X8nC00rBYPofrOONUOR4utbzvbd4d2xT_tj4TdR_0tsr91Y7VskCRFnoXAnNT-qQb7ci7HIBTbutb9zVStOFejrb4aLbr7Fl4byeIEYgp2Gd7gY"
}
```

Store it (client side), the JWT is reusable until its TTL has expired
(3600 seconds by default).

### 2. Use the token

Simply pass the JWT on each request to the protected firewall, either as
an authorization header or as a query parameter.

By default only the authorization header mode is enabled :
``Authorization: Bearer {token}``

See the [configuration reference](./1-configuration-reference.md) document
to enable query string parameter mode or change the header value prefix.

### Examples

See [Functionally testing a JWT protected](./3-functional-testing.md) document or the sandbox application
[Symfony4](https://github.com/chalasr/lexik-jwt-authentication-sandbox)
for a fully working example.

## Notes

### About token expiration

Each request after token expiration will result in a 401 response. Redo
the authentication process to obtain a new token.

Maybe you want to use a **refresh token** to renew your JWT. In this
case you can check
[JWTRefreshTokenBundle](https://github.com/markitosgv/JWTRefreshTokenBundle).

### Working with CORS requests

This is more of a Symfony related topic, but see [Working with CORS requests](./4-cors-requests.md) document to get a quick explanation on handling CORS requests.

### Impersonation

For impersonating users using JWT, see
https://symfony.com/doc/current/security/impersonating_user.html

### Important note for Apache users

As stated in [this
link](https://stackoverflow.com/questions/11990388/request-headers-bag-is-missing-authorization-header-in-symfony-2)
and [this one](https://stackoverflow.com/questions/19443718/symfony-2-3-getrequest-headers-not-showing-authorization-bearer-token/19445020),
Apache server will strip any ``Authorization header`` not in a valid
HTTP BASIC AUTH format.

If you intend to use the authorization header mode of this bundle (and
you should), please add those rules to your VirtualHost configuration :

```
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
```

## Further documentation

The following documents are available:

- [Configuration reference](./1-configuration-reference.rst)
- [Data customization and validation](./2-data-customization.rst>)
- [Functionally testing a JWT protected api](./3-functional-testing.rst>)
- [Working with CORS requests](./4-cors-requests.rst>)
- [JWT encoder service customization](./5-encoder-service.rst>)
- [Extending Authenticator](./6-extending-jwt-authenticator.rst>)
- [Creating JWT tokens programmatically](./7-manual-token-creation.rst>)
- [A database-less user provider](./8-jwt-user-provider.rst>)
- [Accessing the authenticated JWT token](./9-access-authenticated-jwt-token.rst>)
- [Web-Token feature](./10-web-token.rst>)
