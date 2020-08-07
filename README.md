OpenID Connect with Apache example
===

This project represents a simple example of how to add OpenID Connect (OIDC) SSO to an app.

The project uses:

  * [mod_auth_openidc](https://github.com/zmartzone/mod_auth_openidc) as the OIDC Relying Party
  * [Keycloak](https://www.keycloak.org) as the OIDC Provider

These instructions were tested on MacOS. The app behind Apache is just a demo server that dumps all HTTP headers on the
console. The app-specific configuration is contained in file `httpd-ssl.conf`:

    <Location "/app">
        AuthType openid-connect
        Require valid-user
        RequestHeader set REMOTE_USER %{REMOTE_USER}s
        ProxyPass "http://192.168.1.2:9176"
        ProxyPassReverse "http://192.168.1.2:9176"
    </Location>

You can replace it by pointing at your app.

Note - The project is using a real IP, 192.168.1.2, instead of localhost (didn't work otherwise).

## Creating the server private key and SSL certificate

Create the private key `server.key` and the self-signed SSL certificate `server.crt` as follows:

    openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
    openssl rsa -passin pass:x -in server.pass.key -out server.key
    openssl req -new -key server.key -out server.csr \
      -subj "/C=IT/ST=Italy/L=Bologna/O=FooBar inc/OU=Nerds/CN=foo.bar.baz"
    openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
    rm server.pass.key server.csr

## Setting up Apache with HTTPS and the OpenID Connect Apache module

    docker run -dit --name oidc -p 443:443 httpd:2.4
    docker cp httpd.conf oidc:/usr/local/apache2/conf
    docker cp httpd-ssl.conf oidc:/usr/local/apache2/conf/extra
    docker cp server.crt oidc:/usr/local/apache2/conf
    docker cp server.key oidc:/usr/local/apache2/conf
    docker exec oidc apt-get update
    docker exec oidc apt-get install -y libapache2-mod-auth-openidc
    docker exec oidc cp /usr/lib/apache2/modules/mod_auth_openidc.so /usr/local/apache2/modules/
    docker restart oidc
    
Note - restarting the container was necessary due to <https://github.com/zmartzone/mod_auth_openidc/issues/458>.
Otherwise, you could just restart the Apache inside:

    docker exec oidc apachectl restart
    
# Installing Keycloak

    docker run -d --name keycloak -p 8080:8080 \
        -e KEYCLOAK_USER=keycloak -e KEYCLOAK_PASSWORD=keycloak \
        -e KEYCLOAK_IMPORT=/tmp/realm-export.json -v $(pwd)/realm-export.json:/tmp/realm-export.json \
        jboss/keycloak
    docker logs -f keycloak
    
When done, create a user on the OIDC realm at <http://localhost:8080/auth/admin/master/console/#/realms/oidc/users/>
so that you can use it to log into the app (the keycloak export does not include users).

# Installing the dummy app

    nc -l -p 9176
    
If you navigate to the app at <https://192.168.1.2/app/> the page will stay there hanging, but on the console
you can see all HTTP headers coming through:

    GET / HTTP/1.1
    Host: 192.168.1.2:9176
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:79.0) Gecko/20100101 Firefox/79.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Accept-Language: en,en-US;q=0.8,it-IT;q=0.5,it;q=0.3
    Accept-Encoding: gzip, deflate, br
    DNT: 1
    Cookie: mod_auth_openidc_session=fc570c0a-c00f-4ccd-b30d-1cb19919058c
    Upgrade-Insecure-Requests: 1
    OIDC_CLAIM_sub: 995b7562-267b-4f40-997a-f54f903fe3fd
    OIDC_CLAIM_email_verified: 1
    OIDC_CLAIM_name: Foo Bar
    OIDC_CLAIM_preferred_username: foo
    OIDC_CLAIM_given_name: Foo
    OIDC_CLAIM_family_name: Bar
    OIDC_CLAIM_email: foo@foobar.com
    OIDC_CLAIM_exp: 1596801497
    OIDC_CLAIM_iat: 1596801197
    OIDC_CLAIM_auth_time: 1596800809
    OIDC_CLAIM_jti: 979d9a93-ed3c-4fbf-86fd-80da5beda523
    OIDC_CLAIM_iss: http://192.168.1.2:8080/auth/realms/oidc
    OIDC_CLAIM_aud: myapp
    OIDC_CLAIM_typ: ID
    OIDC_CLAIM_azp: myapp
    OIDC_CLAIM_nonce: C5eCP-3PmjOq-vy8_Mkgqam73yPJSFgHWIhYfFctboo
    OIDC_CLAIM_session_state: 58359056-1947-4093-ab6b-501479345812
    OIDC_CLAIM_at_hash: f_oyq15ohlmJEJ6AtLUBGQ
    OIDC_CLAIM_acr: 0
    OIDC_access_token: eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJ3WjVxNzhpaTFuX3k5bHlHNVNPQXZ0b2ZtSGMzVkJsSTFzSFQzYnRaeW80In0.eyJleHAiOjE1OTY4MDE0OTcsImlhdCI6MTU5NjgwMTE5NywiYXV0aF90aW1lIjoxNTk2ODAwODA5LCJqdGkiOiJiZWZjZTEyNi0zODI4LTRjYTUtYjU2OS1hZTliOTYzMjhhZmIiLCJpc3MiOiJodHRwOi8vMTkyLjE2OC4xLjI6ODA4MC9hdXRoL3JlYWxtcy9vaWRjIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6Ijk5NWI3NTYyLTI2N2ItNGY0MC05OTdhLWY1NGY5MDNmZTNmZCIsInR5cCI6IkJlYXJlciIsImF6cCI6Im15YXBwIiwibm9uY2UiOiJDNWVDUC0zUG1qT3Etdnk4X01rZ3FhbTczeVBKU0ZnSFdJaFlmRmN0Ym9vIiwic2Vzc2lvbl9zdGF0ZSI6IjU4MzU5MDU2LTE5NDctNDA5My1hYjZiLTUwMTQ3OTM0NTgxMiIsImFjciI6IjAiLCJhbGxvd2VkLW9yaWdpbnMiOlsiaHR0cHM6Ly8xOTIuMTY4LjEuMi9hcHAiXSwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIm9mZmxpbmVfYWNjZXNzIiwidW1hX2F1dGhvcml6YXRpb24iXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6Im9wZW5pZCBwcm9maWxlIGVtYWlsIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsIm5hbWUiOiJGb28gQmFyIiwicHJlZmVycmVkX3VzZXJuYW1lIjoiZm9vIiwiZ2l2ZW5fbmFtZSI6IkZvbyIsImZhbWlseV9uYW1lIjoiQmFyIiwiZW1haWwiOiJmb29AZm9vYmFyLmNvbSJ9.CdWrKoWYqPV6rdT3v-ZrfZOcN_gfDqrrp454P2TAlzkAFXABkTp3vQsyiOSdl2jJ5Ddk-1Q3GtPQJzEuLT48yMZtl1mK3tnoBp_sP-XXr3YG_NONT73faGm81cy3dhsqpWXB4mJA2yyGamaWeRQYxh_j-Su8E7rBF_CdHd0GglwfjPiALq5nDWF0EMPM8QLGsbAr313cGxqyp1rZkIGinqrht1oh659h9P1kAaG1Rz5vvfSvAqX9Ofti_OyKaMH7mL6F0vmanstvwaziRGd05qdZSNTaqM6O6lu4n7tqrjmLsdsSxqCZ2w7-8JWueHc4sV-iqiqZE50EaU1_x84VrQ
    OIDC_access_token_expires: 1596801497
    X-Forwarded-Proto: https
    REMOTE_USER: foo
    X-Forwarded-For: 172.17.0.1
    X-Forwarded-Host: 192.168.1.2
    X-Forwarded-Server: 192.168.1.2
    Connection: close

Most importanly:

  * REMOTE_USER
  * OIDC_access_token

To logout, navigate to <https://192.168.1.2/app?logout=get>.

## References

* <https://devcenter.heroku.com/articles/ssl-certificate-self>
* <https://hub.docker.com/_/httpd>
* <https://docs.docker.com/docker-for-mac/networking/> (`host.docker.internal`)
* <https://www.keycloak.org/docs/latest/securing_apps/index.html#_mod_auth_openidc>
* <https://hub.docker.com/r/jboss/keycloak>
* <https://github.com/zmartzone/mod_auth_openidc>
* <https://github.com/zmartzone/mod_auth_openidc/wiki>
* <https://github.com/Reposoft/openidc-keycloak-test>
* <https://github.com/zmartzone/mod_auth_openidc/wiki/Access-Tokens-and-Refresh-Tokens>
* <https://github.com/zmartzone/mod_auth_openidc/issues/458>
* <https://github.com/docker/for-win/issues/2402>
* <https://github.com/zmartzone/mod_auth_openidc/issues/348>