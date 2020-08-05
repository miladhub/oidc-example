Apache with HTTPS example
===

This project represents a simple example of how to expose a service on Apache using HTTPS.

These instructions were tested on MacOS. The app behind Apache is just a demo server that dumps all HTTP headers on the console. The app-specific configuration is contained in fil `httpd-ssl.conf`:

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

    docker run -dit --name apache -p 443:443 httpd:2.4
    docker cp httpd.conf apache:/usr/local/apache2/conf
    docker cp httpd-ssl.conf apache:/usr/local/apache2/conf/extra
    docker cp server.crt apache:/usr/local/apache2/conf
    docker cp server.key apache:/usr/local/apache2/conf
    docker exec apache apachectl restart

# Installing the dummy app

    nc -l 9176
    
If you navigate to the app at <https://localhost/app> the page will stay there hanging, but on the console
you can see all HTTP headers coming through in clear text (i.e., unencrypted), which means that the Apache HTTPS proxy is working:

    GET / HTTP/1.1
    Host: host.docker.internal:9176
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-gb
    Accept-Encoding: gzip, deflate, br
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_4) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.1 Safari/605.1.15
    X-Forwarded-For: 172.17.0.1
    X-Forwarded-Host: localhost
    X-Forwarded-Server: www.example.com
    Connection: Keep-Alive

## References

* <https://devcenter.heroku.com/articles/ssl-certificate-self>
* <https://hub.docker.com/_/httpd>
* <https://docs.docker.com/docker-for-mac/networking/> (`host.docker.internal`)

