OpenID Connect with Apache example
===

This project represents a simple example of how to add OpenID Connect (OIDC) SSO to an app via Apache.

The project uses the Apache module <https://github.com/zmartzone/mod_auth_openidc> to integrate with OIDC.

These instructions were tested on MacOS. The app behind Apache is just a demo server that dumps all HTTP headers on the
console. The app-specific configuration is contained in fil `httpd.conf`:

    <Location /app>
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

## Setting up Apache with HTTPS

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

    docker run --name keycloak -p 8080:8080 jboss/keycloak
    docker run -e KEYCLOAK_USER=keycloak -e KEYCLOAK_PASSWORD=keycloak keycloak
    cp realm-export.json /tmp/
    docker run -e KEYCLOAK_USER=keycloak -e KEYCLOAK_PASSWORD=keycloak \
        -e KEYCLOAK_IMPORT=/tmp/realm-export.json -v /tmp/realm-export.json:/tmp/realm-export.json keycloak
    rm /tmp/realm-export.json
    
# Installing the dummy app

    nc -l -p 9176
    
If you navigate to the app at <https://localhost/app> the page will stay there hanging, but on the console
you can see all HTTP headers coming through in clear text (i.e., unencrypted), which means that the Apache HTTPS proxy is working:

    GET /redirect_uri?state=wL_vwbmuw8u2LH1hGsDuPdKeZfg&session_state=50dda3d8-59a9-4969-b759-69aab657c4aa&code=b6cf3e23-22f2-4674-893f-42b32d35dc1e.50dda3d8-59a9-4969-b759-69aab657c4aa.00a8e7da-d158-481e-b28f-daac993f0f26 HTTP/1.1
    Host: host.docker.internal:9176
    Connection: keep-alive
    Cache-Control: max-age=0
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.105 Safari/537.36
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
    Sec-Fetch-Site: same-site
    Sec-Fetch-Mode: navigate
    Sec-Fetch-User: ?1
    Sec-Fetch-Dest: document
    Referer: http://localhost:8080/auth/realms/oidc/protocol/openid-connect/auth?response_type=code&scope=openid&client_id=myapp&state=wL_vwbmuw8u2LH1hGsDuPdKeZfg&redirect_uri=http%3A%2F%2Flocalhost%3A9176%2Fredirect_uri&nonce=RxZ-j3jUS_i5Dtnja0F9ugBn3Nu75dChkNnwSdOeZzs
    Accept-Encoding: gzip, deflate, br
    Accept-Language: en-US,en;q=0.9,it;q=0.8,pl;q=0.7
    Cookie: ajs_group_id=null; ajs_anonymous_id=%2226658319-024e-48c9-8abb-ccaab2cc9cea%22; _ga=GA1.1.320453682.1589111484; ajs_user_id=%2203789f083f9848f696dd318ce17c8f5e%22; mp_956569872348636f9bc2d62e4f838261_mixpanel=%7B%22distinct_id%22%3A%20%2203789f083f9848f696dd318ce17c8f5e%22%2C%22%24device_id%22%3A%20%22171fe6cf090240-043721c2619412-30667d00-13c680-171fe6cf0945b6%22%2C%22mp_lib%22%3A%20%22Segment%3A%20web%22%2C%22%24initial_referrer%22%3A%20%22%24direct%22%2C%22%24initial_referring_domain%22%3A%20%22%24direct%22%2C%22app%22%3A%20%22studio%22%2C%22studio_version%22%3A%20%221.9.6%22%2C%22%24user_id%22%3A%20%2203789f083f9848f696dd318ce17c8f5e%22%2C%22mp_name_tag%22%3A%20%2203789f083f9848f696dd318ce17c8f5e%22%2C%22memsql_version%22%3A%20%227.0.16%22%2C%22id%22%3A%20%2203789f083f9848f696dd318ce17c8f5e%22%7D; mod_auth_openidc_state_wL_vwbmuw8u2LH1hGsDuPdKeZfg=eyJhbGciOiAiZGlyIiwgImVuYyI6ICJBMjU2R0NNIn0..XFSGtg1gDIHY4rXv.LDFuqXOtGc61yNGQUPY6v54SfnglJPZwOogAxeFXvFwZejk5BT-bQCpb0JXiWyF35T-WQF-eAxA6MOkCtNL2vdE8swgE8GE0xzRKUceO4ij_olamq23_nZ0cAa7NCDbYgrft9cBqz6aHpgpNo7vjT9hogFxwn72Uyrl0qyV4fPm-fwOGCOzUC4kjgkStFkPNePQUxu4Ie7qW7TgF8KWbnp4s70vgjLbI_CasRpgNJP-mQCkrFBYuqgfCVOuiJOO7EIATZVGydvcTBNeEou4HZ7V87jriINTTnAq7FFbstWR2T83H3XVNPpqZEUR9tpVyqoYw3uSKoXlyZzFfcWCelGCEMabXNQD2ygy7hhJn4IHMSdVZDTIEmaighCe3tSlnRtdDCqPLTFTmjbo.aSiO9T1QE2qpvOgPYb_YGw

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