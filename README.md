# Name

haproxy-cloudflare-jwt-validator - JSON Web Token validation for haproxy

# Erisa Fork description

This Docker image allows you to simply and easily validate a JWT for a dockerized Cloudflare Access application before passing off requests to the application backend.

The image is available as `erisamoe/haproxy-cf-jwt` and `ghcr.io/erisa/haproxy-cf-jwt`.  

Currently, only `linux/amd64` is supported. I would like to add more architectures where possible in the near future.

As well as trying to keep up to date with HAProxy releases, this fork also makes the change to define all required values in environment variables.

To explain this, here's a `docker-compose.yml` example:

```yaml
services:
  access:
    image: erisamoe/haproxy-cf-jwt
    restart: unless-stopped
    environment:
      - OAUTH_HOST=erisa.cloudflareaccess.com
      - AUDIENCE_TAG=1234567890abcde1234567890abcde1234567890abcde
      - BACKEND=webserver:80
    depends_on:
      - webserver
```

The required environment variables are:
- `OAUTH_HOST`: Your Cloudflare Access domain, e.g. `erisa.cloudflareaccess.com`.
- `AUDIENCE_TAG`: The Application Audience (AUD) Tag for your Access application.
- `BACKEND`: The backend webserver to serve requests after authentication.

Currently only one application and backend per each container instance is supported, you will have to make manual edits to `haproxy.cfg` and mount it as `/usr/local/etc/haproxy/haproxy.cfg` for custom logic and behaviour.

# Description

This was tested & developed with HAProxy version 2.3 & Lua version 5.3.
This library provides the ability to validate JWT headers sent by Cloudflare Access. 

Based off of [haproxytech/haproxy-lua-jwt](https://github.com/haproxytech/haproxy-lua-jwt)

# Installation

Install the following dependencies:

* [haproxy-lua-http](https://github.com/haproxytech/haproxy-lua-http)
* [rxi/json](https://github.com/rxi/json.lua)
* [wahern/luaossl](https://github.com/wahern/luaossl)
* [diegonehab/luasocket](https://github.com/diegonehab/luasocket)

Extract base64.lua & jwtverify.lua to the same directory like so:

```shell
git clone git@github.com:kudelskisecurity/haproxy-cloudflare-jwt-validator.git
sudo cp haproxy-cloudflare-jwt-validator/src/* /usr/local/share/lua/5.3
```

# Version

0.3.1

# Usage

JWT Issuer: `https://test.cloudflareaccess.com` (replace with yours in the config below)

Add the following settings in your `/etc/haproxy/haproxy.cfg` file: 

Define a HAProxy backend, DNS Resolver, and ENV variables with the following names:

```
global
  lua-load  /usr/local/share/lua/5.3/jwtverify.lua
    setenv  OAUTH_HOST     test.cloudflareaccess.com
    setenv  OAUTH_JWKS_URL https://|cloudflare_jwt|/cdn-cgi/access/certs
    setenv  OAUTH_ISSUER   https://"${OAUTH_HOST}"

backend cloudflare_jwt
  mode http
  default-server inter 10s rise 2 fall 2
  server "${OAUTH_HOST}" "${OAUTH_HOST}":443 check resolvers dnsresolver resolve-prefer ipv4

resolvers dnsresolver
  nameserver dns1 1.1.1.1:53
  nameserver dns2 1.0.0.1:53
  resolve_retries 3
  timeout retry 1s
  hold nx 10s
  hold valid 10s
```

Obtain your Application Audience (AUD) Tag from Cloudflare and define your backend with JWT validation:

```
backend my_jwt_validated_app
  mode http
  http-request deny unless { req.hdr(Cf-Access-Jwt-Assertion) -m found }
  http-request set-var(txn.audience) str("1234567890abcde1234567890abcde1234567890abcde")
  http-request lua.jwtverify
  http-request deny unless { var(txn.authorized) -m bool }
  server haproxy 127.0.0.1:8080
```

# Docker

```
docker run \
-v path_to/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg \
kudelskisecurity/haproxy-cloudflare-jwt-validator:0.3.0 
```

# Blogpost

This work has been publish in [our Kudelskisecurity Research blog](https://research.kudelskisecurity.com/2020/08/04/first-steps-towards-a-zero-trust-architecture/)
