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
        ProxyPass "http://host.docker.internal:9176"
        ProxyPassReverse "http://host.docker.internal:9176"
    </Location>

You can replace it by pointing at your app.

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
    docker logs -f oidc
    
Note - restarting the container was necessary due to <https://github.com/zmartzone/mod_auth_openidc/issues/458>.
Otherwise, you could just restart the Apache inside:

    docker exec oidc apachectl restart
    
# Making networking work from the host

    echo 127.0.0.1 host.docker.internal | sudo tee -a /etc/hosts

This is because the Apache module will both need to contact the OIDC provider and then redirect the browser to it
(running on the host), so I need an address that works from both.

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
    
If you navigate to the app at <https://localhost/app/> the page will stay there hanging, but on the console
you can see all HTTP headers coming through:

    GET / HTTP/1.1
    Host: host.docker.internal:9176
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:79.0) Gecko/20100101 Firefox/79.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Accept-Language: en,en-US;q=0.8,it-IT;q=0.5,it;q=0.3
    Accept-Encoding: gzip, deflate, br
    Referer: http://host.docker.internal:8080/
    DNT: 1
    Cookie: mod_auth_openidc_state_pI_0PcEsOCdq8GW1n5AtqitZtnQ=eyJhbGciOiAiZGlyIiwgImVuYyI6ICJBMjU2R0NNIn0..wPnMXXs48LZSSCCN.uOpBcuA5lv7Xl3oNIiQmZnbfFhAvilY9AAcBsGLGL1W7iRQJeSi9vaO51Q2I6BB5OY0Th9uMKoZEN1ufLRzjMWMGtk1iqX3-Fjq-vjdyjrZWppcB3wtWupum20b1FjVsv1Y5Sie_Po_fJIomPP0jye8UTDAgsRSjPoLpChaD_ua-JK5mm-E8SEespJ0m-Q5q1HR3dWjgmsKoTCeOcPUDqPROjFgS7ECMQYx6kLJkC_5Jeze7KF2QHgRRc0ZudA7oP2G-CvVftpKQak-AJpGSjbXzWeQGQZudc-YWsQocXHP2OneLN35PHD76HsSWNq99kPWERfu39D7ia4NhSFOeHfDE1bsuEI_BywObiPE2zdjBNAm1odM9fzuIqUHn18S-edplJdg0scuU-8ZzZg.9JwtSHVFATW2gcOLMyLvyw; mod_auth_openidc_session=a2153620-076d-49cd-9738-c04bebdae5a7
    Upgrade-Insecure-Requests: 1
    OIDC_CLAIM_sub: aad7c98a-f9b3-4d09-adc4-de8c63b1680b
    OIDC_CLAIM_email_verified: 1
    OIDC_CLAIM_name: Foo Bar
    OIDC_CLAIM_preferred_username: foo
    OIDC_CLAIM_given_name: Foo
    OIDC_CLAIM_family_name: Bar
    OIDC_CLAIM_email: foo@foobar.com
    OIDC_CLAIM_exp: 1596811107
    OIDC_CLAIM_iat: 1596810807
    OIDC_CLAIM_auth_time: 1596810806
    OIDC_CLAIM_jti: 66500442-e093-4f64-9cd8-7d8492a97e97
    OIDC_CLAIM_iss: http://host.docker.internal:8080/auth/realms/oidc
    OIDC_CLAIM_aud: myapp
    OIDC_CLAIM_typ: ID
    OIDC_CLAIM_azp: myapp
    OIDC_CLAIM_nonce: VF2I13j7PUR_zGS7fTj6nDf2y56Ujz0dMG6r1ELDDaM
    OIDC_CLAIM_session_state: 5233170c-8c5e-4172-b500-b8c8a2dc52a7
    OIDC_CLAIM_at_hash: giRR-x-V-hCF7BEU-v41Qw
    OIDC_CLAIM_acr: 1
    OIDC_access_token: eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJDWFJxaWxEWnludGtUeFBwOWJOeWZwVmdnSEp2QklTM0JjRkRuRy1EcXJVIn0.eyJleHAiOjE1OTY4MTExMDcsImlhdCI6MTU5NjgxMDgwNywiYXV0aF90aW1lIjoxNTk2ODEwODA2LCJqdGkiOiIwNDYyMTc3MS1jMTI2LTQ2ZWYtOGI0YS0zOTQzYzcxNGE3ZTciLCJpc3MiOiJodHRwOi8vaG9zdC5kb2NrZXIuaW50ZXJuYWw6ODA4MC9hdXRoL3JlYWxtcy9vaWRjIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6ImFhZDdjOThhLWY5YjMtNGQwOS1hZGM0LWRlOGM2M2IxNjgwYiIsInR5cCI6IkJlYXJlciIsImF6cCI6Im15YXBwIiwibm9uY2UiOiJWRjJJMTNqN1BVUl96R1M3ZlRqNm5EZjJ5NTZVanowZE1HNnIxRUxERGFNIiwic2Vzc2lvbl9zdGF0ZSI6IjUyMzMxNzBjLThjNWUtNDE3Mi1iNTAwLWI4YzhhMmRjNTJhNyIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiaHR0cHM6Ly9sb2NhbGhvc3QvYXBwIl0sInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJvZmZsaW5lX2FjY2VzcyIsInVtYV9hdXRob3JpemF0aW9uIl19LCJyZXNvdXJjZV9hY2Nlc3MiOnsiYWNjb3VudCI6eyJyb2xlcyI6WyJtYW5hZ2UtYWNjb3VudCIsIm1hbmFnZS1hY2NvdW50LWxpbmtzIiwidmlldy1wcm9maWxlIl19fSwic2NvcGUiOiJvcGVuaWQgcHJvZmlsZSBlbWFpbCIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJuYW1lIjoiRm9vIEJhciIsInByZWZlcnJlZF91c2VybmFtZSI6ImZvbyIsImdpdmVuX25hbWUiOiJGb28iLCJmYW1pbHlfbmFtZSI6IkJhciIsImVtYWlsIjoiZm9vQGZvb2Jhci5jb20ifQ.Gy-BbXZLS9lS8WcsA9FCIC16AIaX5BcFKojNTwAnU1363EaGsRNsMnzxF17HNBIGU7dRhgtae2_DIwgjQ16VoGFt_V2nhxrmlL0PBADlIdGmm4ooOVS3i1SRj9Aigtv4CtTDITOX-6qxziFy90NzGPOlaUq9O1n-hwLlVfHH_Hyncgh-nh2FYk97DiDop7lAUa3FzpjKJpkatAk8P4cibk2FcsxCMy7kp-H_GRd4B7TRhZQYGvGOpWJfYaMtzqxrUPCrNFATXD3wGLBkVpGL0pKjVgzbMY90qGzpLItRjut_UMiXp9p7ka3BHdjgvw1adsmc--NFQuCt36Sxnxx4Hg
    OIDC_access_token_expires: 1596811107
    X-Forwarded-Proto: https
    REMOTE_USER: foo
    X-Forwarded-For: 172.17.0.1
    X-Forwarded-Host: localhost
    X-Forwarded-Server: localhost
    Connection: close

Most importanly:

  * REMOTE_USER
  * OIDC_access_token

To logout, navigate to <https://localhost/app/redirect_uri/?logout=get>.

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
