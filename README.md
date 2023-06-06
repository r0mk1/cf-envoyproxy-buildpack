# cf-envoyproxy-buildpack

The buildpack starts an envoy proxy sidecar process with a configuration file
`envoy.yaml.erb` provided by an application.  The configuration is an
[ERB](https://docs.ruby-lang.org/en/master/ERB.html) template with
`VCAP_SERVICES` and `VCAP_APPLICATION` envars already loaded and available as
Ruby's hashes with correspondent names.


## Example

Assume we have an application deployed to CloudFoundry and we want to secure
it that only requests with a valid JWT token could reach its pages.

First, we create an instance of the `xsuaa` service and bind it to the
application.  `VCAP_SERVICES` environment variable will contain necessary
information how to validate JWT tokens with the service instance.

Second, we add the buildpack to the app's `manifest.yml`:

```yaml
    buildpacks:
    - https://github.com/r0mk1/cf-envoyproxy-buildpack.git
    ...
```

Last, we create the envoy configuration ERB template with parameters taken from
the binding to the `xsuaa` service.

Assume the application listens to the port `8000` and we have a single xsuaa
instance bound to it.  Then the `envoy.yaml.erb` template may look like this:

```yaml
<% require 'uri' -%>
<% XSUAA = URI.parse(VCAP_SERVICES["xsuaa"][0]["credentials"]["url"]) -%>
---
static_resources:
  listeners:
  - name: listener_8080
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: app }
          http_filters:
          - name: envoy.filters.http.jwt_authn
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
              providers:
                xsuaa:
                  issuer: <%= URI.join(XSUAA.to_s, "/oauth/token") %>
                  forward: true
                  remote_jwks:
                    http_uri:
                      uri: <%= URI.join(XSUAA.to_s, "/token_keys") %>
                      cluster: xsuaa
                      timeout: 5s
                    cache_duration: { seconds: 600 }
              rules:
              - match: { prefix: / }
                requires: { provider_name: xsuaa }
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: app
    connect_timeout: 5s
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: app
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: 127.0.0.1, port_value: 8000 }
  - name: xsuaa
    connect_timeout: 5s
    type: logical_dns
    lb_policy: round_robin
    load_assignment:
      cluster_name: xsuaa
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: <%= XSUAA.host %>, port_value: <%= XSUAA.port %> }
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: <%= XSUAA.host %>
        common_tls_context:
          validation_context:
            match_typed_subject_alt_names:
            - san_type: DNS
              matcher: { exact: "<%= XSUAA.host.split('.').drop(1).prepend('*').join('.') %>" }
            trusted_ca: { filename: /etc/ssl/certs/ca-certificates.crt }
```

Push the application to CloudFoundry and open its URL in the browser.  Should see a message:

```
Jwt is missing
```


## License

Apache License 2.0
