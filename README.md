# kubernetes-ambassador
Kubernetes configuration yamls to construct Ambassador.

## Usage
### Prepare server certification files
You have to prepare server certification files using Let's Encrypt and so on.

### Store server certification files to Kubernetes
1. Store server certification files to Kubernetes Secret as `ambassador-certs`.

    ```bash
    $ kubectl create secret tls ambassador-certs --cert=path/to/fullchain.pem --key=path/to/privkey.pem
    ```

### Start Ambassador
1. Start Ambassador Service & Pods.

    ```bash
    $ kubectl apply -f ambassador/ambassador.yaml
    ```
    * This `ambassador/ambassador.yaml` starts 3 ambassador pods. Change `replicas: 3` if you want to change the number of pods to start up.

### Register FQDN to DNS
1. Check the External IPv4 of Ambassador.

    ```bash
    $ kubectl get services -l service=ambassador
    NAME         TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)         AGE
    ambassador   LoadBalancer   10.0.149.207   XX.YY.ZZ.WWW   443:32704/TCP   6m
    ```
1. Register A record to DNS in order to resolve Ambassador's External IPv4 as server's FQDN.

### Check the TLS Termination
1. Check the TLS Termination of Ambassador.

    ```bash
    $ curl -v https://FQDN.of.Ambassador/
    * Rebuilt URL to: https://FQDN.of.Ambassador/
    *   Trying XX.YY.ZZ.WWW...
    * TCP_NODELAY set
    * Connected to FQDN.of.Ambassador (XX.YY.ZZ.WWW) port 443 (#0)
    * TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    * Server certificate: FQDN.of.Ambassador
    * Server certificate: Let's Encrypt Authority X3
    * Server certificate: DST Root CA X3
    > GET / HTTP/1.1
    > Host: FQDN.of.Ambassador
    > User-Agent: curl/7.54.0
    > Accept: */*
    >
    < HTTP/1.1 404 Not Found
    < date: Mon, 23 Apr 2018 04:24:49 GMT
    < server: envoy
    < content-length: 0
    <
    * Connection #0 to host FQDN.of.Ambassador left intact
    ```
1. Check the redirection from http(80) to https(443).

    ```bash
    $ curl -v http://FQDN.of.Ambassador
    * Rebuilt URL to: http://FQDN.of.Ambassador/
    *   Trying XX.YY.ZZ.WWW...
    * TCP_NODELAY set
    * Connected to FQDN.of.Ambassador (XX.YY.ZZ.WWW) port 80 (#0)
    > GET / HTTP/1.1
    > Host: FQDN.of.Ambassador
    > User-Agent: curl/7.54.0
    > Accept: */*
    >
    < HTTP/1.1 301 Moved Permanently
    < location: https://FQDN.of.Ambassador/
    < date: Mon, 23 Apr 2018 04:49:40 GMT
    < server: envoy
    < content-length: 0
    <
    ```

### Start two Dummy REST API services
1. Start two Dummy REST API services using [nmatsui/hello-world-api](https://hub.docker.com/r/nmatsui/hello-world-api/) image.

    ```bash
    $ kubectl apply -f dummy-restapi/dummy-restapi.yaml
    ```

### Check the PATH routing
1. Check the routing to "Dummy REST API service 1".

    ```bash
    $ curl -i https://FQDN.of.Ambassador/api1/
    HTTP/1.1 200 OK
    content-type: application/json
    date: Mon, 23 Apr 2018 05:38:43 GMT
    x-envoy-upstream-service-time: 2
    server: envoy
    transfer-encoding: chunked

    {"message":"dummy restapi 1"}
    ```
1. Check the routing to "Dummy REST API service 2".

    ```bash
    $ curl -i https://FQDN.of.Ambassador/api2/foo/
    HTTP/1.1 200 OK
    content-type: application/json
    date: Mon, 23 Apr 2018 05:38:59 GMT
    x-envoy-upstream-service-time: 1
    server: envoy
    transfer-encoding: chunked

    {"message":"dummy restapi 2"}
    ```

### Start Bearer auth service
1. Create auth token json file like below:

    ```json
    {
      "Znda7iglaqdoltsp7kDl60TvkkszcEGU": ["^/api1/.*$", "^/api2/.*$"],
      "fANtLRTszYAayjtmLFllSHBrt2zRyoqV": ["^/api1/.*$"]
    }
    ```
1. Store token json file to Kubernetes Secret as `auth-tokens`.

    ```bash
    $ kubectl create secret generic auth-tokens --from-file=/path/to/auth-token.json
    ```
1. Start bearer auth service using [nmatsui/bearer-auth-api](https://hub.docker.com/r/nmatsui/bearer-auth-api/) image.

    ```bash
    $ kubectl apply -f bearer-auth/bearer-auth.yaml
    ```

### Check the Bearer authorization and path authentication
1. Check the bearer authorization.
    1. valid Authorization header returns "200 OK".

        ```bash
        $ curl -i -H "Authorization: bearer fANtLRTszYAayjtmLFllSHBrt2zRyoqV" https://FQDN.of.Ambassador/api1/foo/bar/
        HTTP/1.1 200 OK
        content-type: application/json
        date: Mon, 23 Apr 2018 07:04:37 GMT
        x-envoy-upstream-service-time: 1
        server: envoy
        transfer-encoding: chunked

        {"message":"dummy restapi 1"}
        ```
    1. no Authorization header or invalid token returns "401 Unauthorized".

        ```bash
        $ curl -i https://FQDN.of.Ambassador/api1/
        HTTP/1.1 401 Unauthorized
        content-type: application/json; charset=utf-8
        www-authenticate: Bearer realm="token_required"
        date: Mon, 23 Apr 2018 07:03:16 GMT
        content-length: 60
        x-envoy-upstream-service-time: 2
        server: envoy

        {"authorized":false,"error":"missing Header: authorization"}
        ```
1. check the path authentication.
    1. allowed path returns "200 OK".

        ```bash
        $ curl -i -H "Authorization: bearer fANtLRTszYAayjtmLFllSHBrt2zRyoqV" https://FQDN.of.Ambassador/api1/foo/bar/
        HTTP/1.1 200 OK
        content-type: application/json
        date: Mon, 23 Apr 2018 07:04:37 GMT
        x-envoy-upstream-service-time: 1
        server: envoy
        transfer-encoding: chunked

        {"message":"dummy restapi 1"}
        ```
    1. not allowed path returns "403 Forbidden".

        ```bash
        $ curl -i -H "Authorization: bearer fANtLRTszYAayjtmLFllSHBrt2zRyoqV" https://FQDN.of.Ambassador/api2/
        HTTP/1.1 403 Forbidden
        content-type: application/json; charset=utf-8
        www-authenticate: Bearer realm="token_required" error="not_allowed"
        date: Mon, 23 Apr 2018 07:06:09 GMT
        content-length: 41
        x-envoy-upstream-service-time: 1
        server: envoy

        {"authorized":false,"error":"not allowd"}
        ```

## See also

* [Ambassador](https://www.getambassador.io/)
* [hello-world-api](https://github.com/nmatsui/hello-world-api)
* [bearer-auth-api](https://github.com/nmatsui/bearer-auth-api)

## License

[MIT License](/LICENSE)

## Copyright
Copyright (c) 2018 Nobuyuki Matsui <nobuyuki.matsui@gmail.com>
