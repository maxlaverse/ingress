## Contents

* [Customizing NGINX](#customizing-nginx)
* [Custom NGINX configuration](#custom-nginx-configuration)
* [Custom NGINX template](#custom-nginx-template)
* [Annotations](#annotations)
* [Custom NGINX upstream checks](#custom-nginx-upstream-checks)
* [Custom NGINX upstream hashing](#custom-nginx-upstream-hashing)
* [Authentication](#authentication)
* [Rewrite](#rewrite)
* [Rate limiting](#rate-limiting)
* [SSL Passthrough](#ssl-passthrough)
* [Secure backends](#secure-backends)
* [Server-side HTTPS enforcement through redirect](#server-side-https-enforcement-through-redirect)
* [Whitelist source range](#whitelist-source-range)
* [Allowed parameters in configuration ConfigMap](#allowed-parameters-in-configuration-configmap)
* [Default configuration options](#default-configuration-options)
* [Websockets](#websockets)
* [Optimizing TLS Time To First Byte (TTTFB)](#optimizing-tls-time-to-first-byte-tttfb)
* [Retries in non-idempotent methods](#retries-in-non-idempotent-methods)
* [Custom max body size](#custom-max-body-size)

### Customizing NGINX

There are 3 ways to customize NGINX:

1. [ConfigMap](#allowed-parameters-in-configuration-configmap): create a stand alone ConfigMap, use this if you want a different global configuration.
2. [annotations](#annotations): use this if you want a specific configuration for the site defined in the Ingress rule.
3. custom template: when more specific settings are required, like [open_file_cache](http://nginx.org/en/docs/http/ngx_http_core_module.html#open_file_cache), custom [log_format](http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format), adjust [listen](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) options as `rcvbuf` or when is not possible to change an through the ConfigMap.

#### Custom NGINX configuration

It is possible to customize the defaults in NGINX using a ConfigMap.
Please check the [custom configuration](../../examples/customization/custom-configuration/nginx/README.md) example.

#### Annotations

The following annotations are supported:

|Name                 |type|
|---------------------------|------|
|[ingress.kubernetes.io/add-base-url](#rewrite)|true or false|
|[ingress.kubernetes.io/app-root](#rewrite)|string|
|[ingress.kubernetes.io/affinity](#session-affinity)|cookie|
|[ingress.kubernetes.io/auth-realm](#authentication)|string|
|[ingress.kubernetes.io/auth-secret](#authentication)|string|
|[ingress.kubernetes.io/auth-type](#authentication)|basic or digest|
|[ingress.kubernetes.io/auth-url](#external-authentication)|string|
|[ingress.kubernetes.io/auth-tls-secret](#certificate-authentication)|string|
|[ingress.kubernetes.io/auth-tls-verify-depth](#certificate-authentication)|number|
|[ingress.kubernetes.io/auth-tls-verify-client](#certificate-authentication)|string|
|[ingress.kubernetes.io/auth-tls-error-page](#certificate-authentication)|string|
|[ingress.kubernetes.io/base-url-scheme](#rewrite)|string|
|[ingress.kubernetes.io/client-body-buffer-size](#client-body-buffer-size)|string|
|[ingress.kubernetes.io/configuration-snippet](#configuration-snippet)|string|
|[ingress.kubernetes.io/default-backend](#default-backend)|string|
|[ingress.kubernetes.io/enable-cors](#enable-cors)|true or false|
|[ingress.kubernetes.io/force-ssl-redirect](#server-side-https-enforcement-through-redirect)|true or false|
|[ingress.kubernetes.io/from-to-www-redirect](#redirect-from-to-www)|true or false|
|[ingress.kubernetes.io/limit-connections](#rate-limiting)|number|
|[ingress.kubernetes.io/limit-rps](#rate-limiting)|number|
|[ingress.kubernetes.io/proxy-body-size](#custom-max-body-size)|string|
|[ingress.kubernetes.io/proxy-connect-timeout](#custom-timeouts)|number|
|[ingress.kubernetes.io/proxy-send-timeout](#custom-timeouts)|number|
|[ingress.kubernetes.io/proxy-read-timeout](#custom-timeouts)|number|
|[ingress.kubernetes.io/proxy-request-buffering](#custom-timeouts)|string|
|[ingress.kubernetes.io/rewrite-target](#rewrite)|URI|
|[ingress.kubernetes.io/secure-backends](#secure-backends)|true or false|
|[ingress.kubernetes.io/server-alias](#server-alias)|string|
|[ingress.kubernetes.io/server-snippet](#server-snippet)|string|
|[ingress.kubernetes.io/service-upstream](#service-upstream)|true or false|
|[ingress.kubernetes.io/session-cookie-name](#cookie-affinity)|string|
|[ingress.kubernetes.io/session-cookie-hash](#cookie-affinity)|string|
|[ingress.kubernetes.io/ssl-redirect](#server-side-https-enforcement-through-redirect)|true or false|
|[ingress.kubernetes.io/ssl-passthrough](#ssl-passthrough)|true or false|
|[ingress.kubernetes.io/upstream-max-fails](#custom-nginx-upstream-checks)|number|
|[ingress.kubernetes.io/upstream-fail-timeout](#custom-nginx-upstream-checks)|number|
|[ingress.kubernetes.io/upstream-hash-by](#custom-nginx-upstream-hashing)|string|
|[ingress.kubernetes.io/whitelist-source-range](#whitelist-source-range)|CIDR|

#### Custom NGINX template

The NGINX template is located in the file `/etc/nginx/template/nginx.tmpl`. Mounting a volume is possible to use a custom version.
Use the [custom-template](../../examples/customization/custom-template/README.md) example as a guide.

**Please note the template is tied to the Go code. Do not change names in the variable `$cfg`.**

For more information about the template syntax please check the [Go template package](https://golang.org/pkg/text/template/).
In addition to the built-in functions provided by the Go package the following functions are also available:

  - empty: returns true if the specified parameter (string) is empty
  - contains: [strings.Contains](https://golang.org/pkg/strings/#Contains)
  - hasPrefix: [strings.HasPrefix](https://golang.org/pkg/strings/#HasPrefix)
  - hasSuffix: [strings.HasSuffix](https://golang.org/pkg/strings/#HasSuffix)
  - toUpper: [strings.ToUpper](https://golang.org/pkg/strings/#ToUpper)
  - toLower: [strings.ToLower](https://golang.org/pkg/strings/#ToLower)
  - buildLocation: helps to build the NGINX Location section in each server
  - buildProxyPass: builds the reverse proxy configuration
  - buildRateLimitZones: helps to build all the required rate limit zones
  - buildRateLimit: helps to build a limit zone inside a location if contains a rate limit annotation

### Custom NGINX upstream checks

NGINX exposes some flags in the [upstream configuration](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) that enable the configuration of each server in the upstream. The Ingress controller allows custom `max_fails` and `fail_timeout` parameters in a global context using `upstream-max-fails` and `upstream-fail-timeout` in the NGINX ConfigMap or in a particular Ingress rule. `upstream-max-fails` defaults to 0. This means NGINX will respect the container's `readinessProbe` if it is defined. If there is no probe and no values for `upstream-max-fails` NGINX will continue to send traffic to the container.

**With the default configuration NGINX will not health check your backends. Whenever the endpoints controller notices a readiness probe failure, that pod's IP will be removed from the list of endpoints. This will trigger the NGINX controller to also remove it from the upstreams.**

To use custom values in an Ingress rule define these annotations:

`ingress.kubernetes.io/upstream-max-fails`: number of unsuccessful attempts to communicate with the server that should occur in the duration set by the `upstream-fail-timeout` parameter to consider the server unavailable.

`ingress.kubernetes.io/upstream-fail-timeout`: time in seconds during which the specified number of unsuccessful attempts to communicate with the server should occur to consider the server unavailable. This is also the period of time the server will be considered unavailable.

In NGINX, backend server pools are called "[upstreams](http://nginx.org/en/docs/http/ngx_http_upstream_module.html)". Each upstream contains the endpoints for a service. An upstream is created for each service that has Ingress rules defined.

**Important:** All Ingress rules using the same service will use the same upstream. Only one of the Ingress rules should define annotations to configure the upstream servers.

Please check the [custom upstream check](../../examples/customization/custom-upstream-check/README.md) example.

### Custom NGINX upstream hashing

NGINX supports load balancing by client-server mapping based on [consistent hashing](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#hash) for a given key. The key can contain text, variables or any combination thereof. This feature allows for request stickiness other than client IP or cookies. The [ketama](http://www.last.fm/user/RJ/journal/2007/04/10/392555/) consistent hashing method will be used which ensures only a few keys would be remapped to different servers on upstream group changes.

To enable consistent hashing for a backend:

`ingress.kubernetes.io/upstream-hash-by`: the nginx variable, text value or any combination thereof to use for consistent hashing. For example `ingress.kubernetes.io/upstream-hash-by: "$request_uri"` to consistently hash upstream requests by the current request URI.

### Authentication

Is possible to add authentication adding additional annotations in the Ingress rule. The source of the authentication is a secret that contains usernames and passwords inside the key `auth`.

The annotations are:
```
ingress.kubernetes.io/auth-type: [basic|digest]
```

Indicates the [HTTP Authentication Type: Basic or Digest Access Authentication](https://tools.ietf.org/html/rfc2617).

```
ingress.kubernetes.io/auth-secret: secretName
```

The name of the secret that contains the usernames and passwords with access to the `path`s defined in the Ingress Rule.
The secret must be created in the same namespace as the Ingress rule.

```
ingress.kubernetes.io/auth-realm: "realm string"
```

Please check the [auth](/examples/auth/basic/nginx/README.md) example.

### Certificate Authentication

It's possible to enable Certificate-Based Authentication (Mutual Authentication) using additional annotations in Ingress Rule.

The annotations are:
```
ingress.kubernetes.io/auth-tls-secret: secretName
```

The name of the secret that contains the full Certificate Authority chain `ca.crt` that is enabled to authenticate against this ingress. It's composed of namespace/secretName.

```
ingress.kubernetes.io/auth-tls-verify-depth
```

The validation depth between the provided client certificate and the Certification Authority chain.

```
ingress.kubernetes.io/auth-tls-verify-client
```

Enables verification of client certificates.

```
ingress.kubernetes.io/auth-tls-error-page
```

The URL/Page that user should be redirected in case of a Certificate Authentication Error

Please check the [tls-auth](/examples/auth/client-certs/nginx/README.md) example.

### Configuration snippet

Using this annotation you can add additional configuration to the NGINX location. For example:

```
ingress.kubernetes.io/configuration-snippet: |
  more_set_headers "Request-Id: $request_id";
```
### Default Backend

The ingress controller requires a default backend. This service is handle the response when the service in the Ingress rule does not have endpoints.
This is a global configuration for the ingress controller. In some cases could be required to return a custom content or format. In this scenario we can use the annotation `ingress.kubernetes.io/default-backend: <svc name>` to specify a custom default backend.

### Enable CORS

To enable Cross-Origin Resource Sharing (CORS) in an Ingress rule add the annotation `ingress.kubernetes.io/enable-cors: "true"`. This will add a section in the server location enabling this functionality.
For more information please check https://enable-cors.org/server_nginx.html

### Server Alias

To add Server Aliases to an Ingress rule add the annotation `ingress.kubernetes.io/server-alias: "<alias>"`.
This will create a server with the same configuration, but a different server_name as the provided host.

*Note:* A server-alias name cannot conflict with the hostname of an existing server. If it does the server-alias
annotation will be ignored. If a server-alias is created and later a new server with the same hostname is created
the new server configuration will take place over the alias configuration.

For more information please see http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name

### Server snippet

Using the annotation `ingress.kubernetes.io/server-snippet` it is possible to add custom configuration in the server configuration block.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
annotations:
ingress.kubernetes.io/server-snippet: |
set $agentflag 0;

if ($http_user_agent ~* "(Mobile)" ){
  set $agentflag 1;
}

if ( $agentflag = 1 ) {
  return 301 https://m.example.com;
}
```

**Important:** This annotation can be used only once per host

### Client Body Buffer Size

Sets buffer size for reading client request body per location. In case the request body is larger than the buffer,
the whole body or only its part is written to a temporary file. By default, buffer size is equal to two memory pages.
This is 8K on x86, other 32-bit platforms, and x86-64. It is usually 16K on other 64-bit platforms. This annotation is
applied to each location provided in the ingress rule.

*Note:* The annotation value must be given in a valid format otherwise the
For example to set the client-body-buffer-size the following can be done:
* `ingress.kubernetes.io/client-body-buffer-size: "1000"` # 1000 bytes
* `ingress.kubernetes.io/client-body-buffer-size: 1k` # 1 kilobyte
* `ingress.kubernetes.io/client-body-buffer-size: 1K` # 1 kilobyte
* `ingress.kubernetes.io/client-body-buffer-size: 1m` # 1 megabyte
* `ingress.kubernetes.io/client-body-buffer-size: 1M` # 1 megabyte

For more information please see http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size

### External Authentication

To use an existing service that provides authentication the Ingress rule can be annotated with `ingress.kubernetes.io/auth-url` to indicate the URL where the HTTP request should be sent.
Additionally it is possible to set `ingress.kubernetes.io/auth-method` to specify the HTTP method to use (GET or POST) and `ingress.kubernetes.io/auth-send-body` to true or false (default).

```
ingress.kubernetes.io/auth-url: "URL to the authentication service"
```

Please check the [external-auth](/examples/auth/external-auth/nginx/README.md) example.

### Rewrite

In some scenarios the exposed URL in the backend service differs from the specified path in the Ingress rule. Without a rewrite any request will return 404.
Set the annotation `ingress.kubernetes.io/rewrite-target` to the path expected by the service.

If the application contains relative links it is possible to add an additional annotation `ingress.kubernetes.io/add-base-url` that will prepend a [`base` tag](https://developer.mozilla.org/en/docs/Web/HTML/Element/base) in the header of the returned HTML from the backend.

If the scheme of [`base` tag](https://developer.mozilla.org/en/docs/Web/HTML/Element/base) need to be specific, set the annotation `ingress.kubernetes.io/base-url-scheme` to the scheme such as `http` and `https`.

If the Application Root is exposed in a different path and needs to be redirected, set the annotation `ingress.kubernetes.io/app-root` to redirect requests for `/`.

Please check the [rewrite](/examples/rewrite/nginx/README.md) example.

### Rate limiting

The annotations `ingress.kubernetes.io/limit-connections`, `ingress.kubernetes.io/limit-rps`, and `ingress.kubernetes.io/limit-rpm` define a limit on the connections that can be opened by a single client IP address. This can be used to mitigate [DDoS Attacks](https://www.nginx.com/blog/mitigating-ddos-attacks-with-nginx-and-nginx-plus).

`ingress.kubernetes.io/limit-connections`: number of concurrent connections allowed from a single IP address.

`ingress.kubernetes.io/limit-rps`: number of connections that may be accepted from a given IP each second.

`ingress.kubernetes.io/limit-rpm`: number of connections that may be accepted from a given IP each minute.

You can specify the client IP source ranges to be excluded from rate-limiting through the `ingress.kubernetes.io/limit-whitelist` annotation. The value is a comma separated list of CIDRs.

If you specify multiple annotations in a single Ingress rule, `limit-rpm`, and then `limit-rps` takes precedence.

The annotation `ingress.kubernetes.io/limit-rate`, `ingress.kubernetes.io/limit-rate-after` define a limit the rate of response transmission to a client. The rate is specified in bytes per second. The zero value disables rate limiting. The limit is set per a request, and so if a client simultaneously opens two connections, the overall rate will be twice as much as the specified limit.

`ingress.kubernetes.io/limit-rate-after`: sets the initial amount after which the further transmission of a response to a client will be rate limited.

`ingress.kubernetes.io/limit-rate`: rate of request that accepted from a client each second.

To configure this setting globally for all Ingress rules, the `limit-rate-after` and `limit-rate` value may be set in the NGINX ConfigMap. if you set the value in ingress annotation will cover global setting.

### SSL Passthrough

The annotation `ingress.kubernetes.io/ssl-passthrough` allows to configure TLS termination in the pod and not in NGINX.

**Important:**
- Using the annotation `ingress.kubernetes.io/ssl-passthrough` invalidates all the other available annotations. This is because SSL Passthrough works in L4 (TCP).
- The use of this annotation requires the flag `--enable-ssl-passthrough` (By default it is disabled)

### Secure backends

By default NGINX uses `http` to reach the services. Adding the annotation `ingress.kubernetes.io/secure-backends: "true"` in the Ingress rule changes the protocol to `https`.

### Service Upstream

By default the NGINX ingress controller uses a list of all endpoints (Pod IP/port) in the NGINX upstream configuration. This annotation disables that behavior and instead uses a single upstream in NGINX, the service's Cluster IP and port. This can be desirable for things like zero-downtime deployments as it reduces the need to reload NGINX configuration when Pods come up and down. See issue [#257](https://github.com/kubernetes/ingress/issues/257).

#### Known Issues

If the `service-upstream` annotation is specified the following things should be taken into consideration:

* Sticky Sessions will not work as only round-robin load balancing is supported.
* The `proxy_next_upstream` directive will not have any effect meaning on error the request will not be dispatched to another upstream.

### Server-side HTTPS enforcement through redirect

By default the controller redirects (301) to `HTTPS` if TLS is enabled for that ingress. If you want to disable that behavior globally, you can use `ssl-redirect: "false"` in the NGINX config map.

To configure this feature for specific ingress resources, you can use the `ingress.kubernetes.io/ssl-redirect: "false"` annotation in the particular resource.

When using SSL offloading outside of cluster (e.g. AWS ELB) it may be useful to enforce a redirect to `HTTPS` even when there is not TLS cert available. This can be achieved by using the `ingress.kubernetes.io/force-ssl-redirect: "true"` annotation in the particular resource.

### Redirect from to www

In some scenarios is required to redirect from `www.domain.com` to `domain.com` or viceversa.
To enable this feature use the annotation `ingress.kubernetes.io/from-to-www-redirect: "true"`

**Important:**
If at some point a new Ingress is created with a host equal to one of the options (like `domain.com`) the annotation will be omitted.


### Whitelist source range

You can specify the allowed client IP source ranges through the `ingress.kubernetes.io/whitelist-source-range` annotation. The value is a comma separated list of [CIDRs](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing), e.g.  `10.0.0.0/24,172.10.0.1`.

To configure this setting globally for all Ingress rules, the `whitelist-source-range` value may be set in the NGINX ConfigMap.

*Note:* Adding an annotation to an Ingress rule overrides any global restriction.

Please check the [whitelist](/examples/affinity/cookie/nginx/README.md) example.

### Session Affinity

The annotation `ingress.kubernetes.io/affinity` enables and sets the affinity type in all Upstreams of an Ingress. This way, a request will always be directed to the same upstream server.
The only affinity type available for NGINX is `cookie`.

### Cookie affinity
If you use the ``cookie`` type you can also specify the name of the cookie that will be used to route the requests with the annotation `ingress.kubernetes.io/session-cookie-name`. The default is to create a cookie named 'route'.

In case of NGINX the annotation `ingress.kubernetes.io/session-cookie-hash` defines which algorithm will be used to 'hash' the used upstream. Default value is `md5` and possible values are `md5`, `sha1` and `index`.
The `index` option is not hashed, an in-memory index is used instead, it's quicker and the overhead is shorter Warning: the matching against upstream servers list is inconsistent. So, at reload, if upstreams servers has changed, index values are not guaranteed to correspond to the same server as before! USE IT WITH CAUTION and only if you need to!

In NGINX this feature is implemented by the third party module [nginx-sticky-module-ng](https://bitbucket.org/nginx-goodies/nginx-sticky-module-ng). The workflow used to define which upstream server will be used is explained [here](https://bitbucket.org/nginx-goodies/nginx-sticky-module-ng/raw/08a395c66e425540982c00482f55034e1fee67b6/docs/sticky.pdf)

### Custom timeouts

Using the configuration configmap it is possible to set the default global timeout for connections to the upstream servers.
In some scenarios is required to have different values. To allow this we provide annotations that allows this customization:

- ingress.kubernetes.io/proxy-connect-timeout
- ingress.kubernetes.io/proxy-send-timeout
- ingress.kubernetes.io/proxy-read-timeout
- ingress.kubernetes.io/proxy-request-buffering


### **Allowed parameters in configuration ConfigMap**

**proxy-body-size:** Sets the maximum allowed size of the client request body. See NGINX [client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size).

**custom-http-errors:** Enables which HTTP codes should be passed for processing with the [error_page directive](http://nginx.org/en/docs/http/ngx_http_core_module.html#error_page).
Setting at least one code also enables [proxy_intercept_errors](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_intercept_errors) which are required to process error_page.

Example usage: `custom-http-errors: 404,415`

**disable-access-log:** Disables the Access Log from the entire Ingress Controller. This is 'false' by default.

**access-log-path:** Access log path. Goes to '/var/log/nginx/access.log' by default. http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log

**error-log-path:** Error log path. Goes to '/var/log/nginx/error.log' by default. http://nginx.org/en/docs/ngx_core_module.html#error_log

**enable-modsecurity:**  enables the modsecurity module for NGINX
By default this is disabled

**enable-owasp-modsecurity-crs:** enables the OWASP ModSecurity Core Rule Set (CRS)
By default this is disabled

**disable-ipv6:** Disable listening on IPV6. This is 'false' by default.

**enable-dynamic-tls-records:** Enables dynamically sized TLS records to improve time-to-first-byte. Enabled by default. See [CloudFlare's blog](https://blog.cloudflare.com/optimizing-tls-over-tcp-to-reduce-latency) for more information.

**enable-underscores-in-headers:** Enables underscores in header names. This is disabled by default.

**enable-vts-status:** Allows the replacement of the default status page with a third party module named [nginx-module-vts](https://github.com/vozlt/nginx-module-vts).

**error-log-level:** Configures the logging level of errors. Log levels above are listed in the order of increasing severity.
http://nginx.org/en/docs/ngx_core_module.html#error_log

**gzip-types:** Sets the MIME types in addition to "text/html" to compress. The special value "\*" matches any MIME type.
Responses with the "text/html" type are always compressed if `use-gzip` is enabled.

**hsts:** Enables or disables the header HSTS in servers running SSL.
HTTP Strict Transport Security (often abbreviated as HSTS) is a security feature (HTTP header) that tell browsers that it should only be communicated with using HTTPS, instead of using HTTP. It provides protection against protocol downgrade attacks and cookie theft.
https://developer.mozilla.org/en-US/docs/Web/Security/HTTP_strict_transport_security
https://blog.qualys.com/securitylabs/2016/03/28/the-importance-of-a-proper-http-strict-transport-security-implementation-on-your-web-server

**hsts-include-subdomains:** Enables or disables the use of HSTS in all the subdomains of the server-name.

**hsts-max-age:** Sets the time, in seconds, that the browser should remember that this site is only to be accessed using HTTPS.

**hsts-preload:** Enables or disables the preload attribute in the HSTS feature (if is enabled)

**ignore-invalid-headers:** set if header fields with invalid names should be ignored. This is 'true' by default.

**keep-alive:** Sets the time during which a keep-alive client connection will stay open on the server side.
The zero value disables keep-alive client connections.
http://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout

**load-balance:** Sets the algorithm to use for load balancing. The value can either be round_robin to
use the default round robin loadbalancer, least_conn to use the least connected method, or
ip_hash to use a hash of the server for routing. The default is least_conn.
http://nginx.org/en/docs/http/load_balancing.html.

**log-format-upstream:** Sets the nginx [log format](http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format).

Example for json output:

```
log-format-upstream: '{ "time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr",
    "x-forward-for": "$proxy_add_x_forwarded_for", "request_id": "$request_id", "remote_user":
    "$remote_user", "bytes_sent": $bytes_sent, "request_time": $request_time, "status":
    $status, "vhost": "$host", "request_proto": "$server_protocol", "path": "$uri",
    "request_query": "$args", "request_length": $request_length, "duration": $request_time,
    "method": "$request_method", "http_referrer": "$http_referer", "http_user_agent":
    "$http_user_agent" }'
  ```

**log-format-stream:** Sets the nginx [stream format](https://nginx.org/en/docs/stream/ngx_stream_log_module.html#log_format).

**max-worker-connections:** Sets the maximum number of simultaneous connections that can be opened by each [worker process](http://nginx.org/en/docs/ngx_core_module.html#worker_connections).

**proxy-buffer-size:** Sets the size of the buffer used for [reading the first part of the response](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size) received from the proxied server. This part usually contains a small response header.

**proxy-connect-timeout:** Sets the timeout for [establishing a connection with a proxied server](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_connect_timeout). It should be noted that this timeout cannot usually exceed 75 seconds.

**proxy-cookie-domain:** Sets a text that [should be changed in the domain attribute](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cookie_domain) of the “Set-Cookie” header fields of a proxied server response.

**proxy-cookie-path:** Sets a text that [should be changed in the path attribute](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cookie_path) of the “Set-Cookie” header fields of a proxied server response.

**proxy-read-timeout:** Sets the timeout in seconds for [reading a response from the proxied server](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout). The timeout is set only between two successive read operations, not for the transmission of the whole response.

**proxy-send-timeout:** Sets the timeout in seconds for [transmitting a request to the proxied server](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_send_timeout). The timeout is set only between two successive write operations, not for the transmission of the whole request.

**proxy-next-upstream:** Specifies in [which cases](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream) a request should be passed to the next server.

**proxy-request-buffering:** Enables or disables [buffering of a client request body](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_request_buffering).

**retry-non-idempotent:** Since 1.9.13 NGINX will not retry non-idempotent requests (POST, LOCK, PATCH) in case of an error in the upstream server.

The previous behavior can be restored using the value "true".

**server-name-hash-bucket-size:** Sets the size of the bucket for the server names hash tables.
http://nginx.org/en/docs/hash.html
http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_bucket_size

**server-name-hash-max-size:** Sets the maximum size of the [server names hash tables](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_max_size) used in server names, map directive’s values, MIME types, names of request header strings, etc.
http://nginx.org/en/docs/hash.html

**proxy-headers-hash-bucket-size:** Sets the size of the bucket for the proxy headers hash tables.
http://nginx.org/en/docs/hash.html
https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_headers_hash_bucket_size

**proxy-headers-hash-max-size:** Sets the maximum size of the proxy headers hash tables.
http://nginx.org/en/docs/hash.html
https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_headers_hash_max_size

**server-tokens:** Send NGINX Server header in responses and display NGINX version in error pages. Enabled by default.

**map-hash-bucket-size:** Sets the bucket size for the [map variables hash tables](http://nginx.org/en/docs/http/ngx_http_map_module.html#map_hash_bucket_size). The details of setting up hash tables are provided in a separate [document](http://nginx.org/en/docs/hash.html).

**ssl-buffer-size:** Sets the size of the [SSL buffer](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_buffer_size) used for sending data.
The default of 4k helps NGINX to improve TLS Time To First Byte (TTTFB).
https://www.igvita.com/2013/12/16/optimizing-nginx-tls-time-to-first-byte/

**ssl-ciphers:** Sets the [ciphers](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_ciphers) list to enable. The ciphers are specified in the format understood by the OpenSSL library.

The default cipher list is:
 `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256`.

The ordering of a ciphersuite is very important because it decides which algorithms are going to be selected in priority.
The recommendation above prioritizes algorithms that provide perfect [forward secrecy](https://wiki.mozilla.org/Security/Server_Side_TLS#Forward_Secrecy).

Please check the [Mozilla SSL Configuration Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/).

**ssl-dh-param:** Sets the name of the secret that contains Diffie-Hellman key to help with "Perfect Forward Secrecy".
https://www.openssl.org/docs/manmaster/apps/dhparam.html
https://wiki.mozilla.org/Security/Server_Side_TLS#DHE_handshake_and_dhparam
http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_dhparam

**ssl-protocols:** Sets the [SSL protocols](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_protocols) to use.
The default is: `TLSv1.2`.

TLSv1 is enabled to allow old clients like:
- [IE 8-10 / Win 7](https://www.ssllabs.com/ssltest/viewClient.html?name=IE&version=8-10&platform=Win%207&key=113)
- [Java 7u25](https://www.ssllabs.com/ssltest/viewClient.html?name=Java&version=7u25&key=26)

If you don't need to support these clients please remove `TLSv1` to improve security.

Please check the result of the configuration using `https://ssllabs.com/ssltest/analyze.html` or `https://testssl.sh`.

**ssl-redirect:** Sets the global value of redirects (301) to HTTPS if the server has a TLS certificate (defined in an Ingress rule)
Default is "true".

**ssl-session-cache:** Enables or disables the use of shared [SSL cache](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_cache) among worker processes.

**ssl-session-cache-size:** Sets the size of the [SSL shared session cache](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_cache) between all worker processes.

**ssl-session-tickets:** Enables or disables session resumption through [TLS session tickets](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_tickets).

**ssl-session-ticket-key:** sets the secret key used to encrypt and decrypt TLS session tickets. The value must be a valid base64 string.
http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_tickets
By default, a randomly generated key is used.
To create a ticket: `openssl rand 80 | base64 -w0`

**ssl-session-timeout:** Sets the time during which a client may [reuse the session](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_timeout) parameters stored in a cache.

**upstream-max-fails:** Sets the number of unsuccessful attempts to communicate with the [server](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) that should happen in the duration set by the `fail_timeout` parameter to consider the server unavailable.

**upstream-fail-timeout:** Sets the time during which the specified number of unsuccessful attempts to communicate with the [server](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) should happen to consider the server unavailable.

**use-gzip:** Enables or disables compression of HTTP responses using the ["gzip" module](http://nginx.org/en/docs/http/ngx_http_gzip_module.html)
The default mime type list to compress is: `application/atom+xml application/javascript aplication/x-javascript application/json application/rss+xml application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/svg+xml image/x-icon text/css text/plain text/x-component`.

**use-http2:** Enables or disables [HTTP/2](http://nginx.org/en/docs/http/ngx_http_v2_module.html) support in secure connections.

**use-proxy-protocol:** Enables or disables the [PROXY protocol](https://www.nginx.com/resources/admin-guide/proxy-protocol/) to receive client connection (real IP address) information passed through proxy servers and load balancers such as HAProxy and Amazon Elastic Load Balancer (ELB).

**whitelist-source-range:** Sets the default whitelisted IPs for each `server` block. This can be overwritten by an annotation on an Ingress rule. See [ngx_http_access_module](http://nginx.org/en/docs/http/ngx_http_access_module.html).

**worker-processes:** Sets the number of [worker processes](http://nginx.org/en/docs/ngx_core_module.html#worker_processes). The default of "auto" means number of available CPU cores.

**worker-shutdown-timeout:** Sets a timeout for Nginx to [wait for worker to gracefully shutdown](http://nginx.org/en/docs/ngx_core_module.html#worker_shutdown_timeout). The default is "10s".

**limit-conn-zone-variable:** Sets parameters for a shared memory zone that will keep states for various keys of [limit_conn_zone](http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html#limit_conn_zone). The default of "$binary_remote_addr" variable’s size is always 4 bytes for IPv4 addresses or 16 bytes for IPv6 addresses.

**proxy-set-headers:** Sets custom headers from a configmap before sending traffic to backends. See [example](https://github.com/kubernetes/ingress/tree/master/examples/customization/custom-headers/nginx)

**add-headers:** Sets custom headers from a configmap before sending traffic to the client. See `proxy-set-headers` [example](https://github.com/kubernetes/ingress/tree/master/examples/customization/custom-headers/nginx)

**bind-address:** Sets the addresses on which the server will accept requests instead of *. It should be noted that these addresses must exist in the runtime environment or the controller will crash loop.

**enable-opentracing:** enables the nginx Opentracing extension https://github.com/rnburn/nginx-opentracing
Default is "false"

**zipkin-collector-host:** specifies the host to use when uploading traces. It must be a valid URL

**zipkin-collector-port:** specifies the port to use when uploading traces
Default: 9411

**zipkin-service-name:** specifies the service name to use for any traces created
Default: nginx

**http-snippet:** adds custom configuration to the http section of the nginx configuration
Default: ""

**server-snippet:** adds custom configuration to all the servers in the nginx configuration
Default: ""

**sendfile:** enable or disable the use of sendfile()
Default: true

**location-snippet:** adds custom configuration to all the locations in the nginx configuration
Default: ""


### Default configuration options

The following table shows the options, the default value and a description.

|name                 |default|
|---------------------------|------|
|body-size|1m|
|custom-http-errors|" "|
|enable-dynamic-tls-records|"true"|
|enable-sticky-sessions|"false"|
|enable-underscores-in-headers|"false"|
|enable-vts-status|"false"|
|error-log-level|notice|
|gzip-types|see use-gzip description above|
|hsts|"true"|
|hsts-include-subdomains|"true"|
|hsts-max-age|"15724800"|
|hsts-preload|"false"|
|ignore-invalid-headers|"true"|
|keep-alive|"75"|
|log-format-stream|[$time_local] $protocol $status $bytes_sent $bytes_received $session_time|
|log-format-upstream|[$the_real_ip] - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time [$proxy_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status|
|map-hash-bucket-size|"64"|
|max-worker-connections|"16384"|
|proxy-body-size|same as body-size|
|proxy-buffer-size|"4k"|
|proxy-request-buffering|"on"|
|proxy-connect-timeout|"5"|
|proxy-cookie-domain|"off"|
|proxy-cookie-path|"off"|
|proxy-read-timeout|"60"|
|proxy-real-ip-cidr|0.0.0.0/0|
|proxy-send-timeout|"60"|
|retry-non-idempotent|"false"|
|server-name-hash-bucket-size|"64"|
|server-name-hash-max-size|"512"|
|server-tokens|"true"|
|ssl-buffer-size|4k|
|ssl-ciphers||
|ssl-dh-param|value from openssl|
|ssl-protocols|TLSv1 TLSv1.1 TLSv1.2|
|ssl-session-cache|"true"|
|ssl-session-cache-size|10m|
|ssl-session-tickets|"true"|
|ssl-session-timeout|10m|
|use-gzip|"true"|
|use-http2|"true"|
|upstream-keepalive-connections|"0" (disabled)|
|variables-hash-bucket-size|64|
|variables-hash-max-size|2048|
|vts-status-zone-size|10m|
|vts-default-filter-key|$geoip_country_code country::*|
|whitelist-source-range|permit all|
|worker-processes|number of CPUs|
|limit-conn-zone-variable|$binary_remote_addr|
|bind-address||

### Websockets

Support for websockets is provided by NGINX out of the box. No special configuration required.

The only requirement to avoid the close of connections is the increase of the values of `proxy-read-timeout` and `proxy-send-timeout`. The default value of this settings is `60 seconds`.
A more adequate value to support websockets is a value higher than one hour (`3600`).

### Optimizing TLS Time To First Byte (TTTFB)

NGINX provides the configuration option [ssl_buffer_size](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_buffer_size) to allow the optimization of the TLS record size. This improves the [Time To First Byte](https://www.igvita.com/2013/12/16/optimizing-nginx-tls-time-to-first-byte/) (TTTFB). The default value in the Ingress controller is `4k` (NGINX default is `16k`).

### Retries in non-idempotent methods

Since 1.9.13 NGINX will not retry non-idempotent requests (POST, LOCK, PATCH) in case of an error.
The previous behavior can be restored using `retry-non-idempotent=true` in the configuration ConfigMap.

### Custom max body size
For NGINX, 413 error will be returned to the client when the size in a request exceeds the maximum allowed size of the client request body. This size can be configured by the parameter [`client_max_body_size`](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size).

To configure this setting globally for all Ingress rules, the `proxy-body-size` value may be set in the NGINX ConfigMap.
To use custom values in an Ingress rule define these annotation:

```
ingress.kubernetes.io/proxy-body-size: 8m
```
