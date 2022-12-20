[![][kong-logo]][kong-url]



# Kong Ingress Controller for Kubernetes (KIC)

Use [Kong] for Kubernetes [Ingress].
Configure [plugins], health checking,
load balancing, and more in Kong
for Kubernetes Services, all using
Custom Resource Definitions (CRDs) and Kubernetes-native tooling.

[**Features**](#features) | [**Get started**](#get-started) | [**PORT**](#PORT) | [**Testing connectivity to Kong Gateway**](#Testing connectivity to Kong Gateway) | [**Kong Custom Configuration**](#Kong Custom Configuration)

## Features

- **Ingress routing**
  Use Ingress resources to configure Kong.
- **Enhanced API management using plugins**
  Use a wide array of plugins to monitor, transform
  and protect your traffic.
- **Native gRPC support**
  Proxy gRPC traffic and gain visibility into it using Kong's plugins.
- **Health checking and Load-balancing**
  Load balance requests across your pods and supports active & passive health-checks.
- **Request/response transformations**
  Use plugins to modify your requests/responses on the fly.
- **Authentication**
  Protect your services using authentication methods of your choice.
- **Declarative configuration for Kong**
  Configure all of Kong using CRDs in Kubernetes and manage Kong declaratively.
- **Rate Limit Plugin**
  A plugin that limits the number of access requests in a specific time.

## Get started


There are 2 versions of kong ingress controller that we will introduce below, each version has its own advantages and disadvantages, depending on your current project situation to choose the suitable version:
- `kong-dbless`: with this version will not use the database, configuration about kong's resources will be stored on k8s, routing information under k8s management will be backed up if kong has problems. however its custom resources( Example: KongIngress, Plugin ) will be lost and you will have to reconfigure.
- `kong-db`: with this version will use database to store kong's data, currently kong is supporting 2 types of databases, postgresql and casscadra, in our installed version will use postgesql. With the use of a database, in addition to backing up data, you can also interact directly with kong via Dashboard (Konga) or Kong Admin API (this is not available in kong-dbless version).

`Option:1`  To start kong-dbless, apply 2 file included inside the installation folder [1-kong-dbless-install](/1-kong-dbless-install).
```cmd
  $ kubectl apply -f /1-kong-dbless-install/1.namespace.yaml
  $ kubectl apply -f /1-kong-dbless-install/2.kong-install.yaml
```
- In addition, you can install konga to manage the kong ingress controller, in this dbless version, konga only supports displaying your configuration information on kong, but you cannot interact directly on this interface.
- To start konga-dbless, apply file included inside the installation folder [2-konga-dbless-install](/2-konga-dbless-install/1-konga-dbless-install.yaml).
```cmd
  $ kubectl apply -f /2-konga-dbless-install/1.konga-dbless-install.yaml
```
`Option:2` To start kong-db, apply 3 file included inside the installation folder [3-kong-db-install](/3-kong-db-install).\
Before installing the kong db version, you need to configure the database for kong first, kong currently supports 2 types of databases, Postgresql and Casscadra, in this version we use Postgresql. 
Here's what you need to install, you can change it in [3-kong-db-install/2-kong-secret](/3-kong-db-install/2-kong-secret.yaml):


- `Kong`
```yaml
    KONG_DATABASE: postgres # Kind of database kong using, we recommend using postgresql
    KONG_PG_HOST: <HOST> # Database host
    KONG_PG_DATABASE: <DB_NAME> # Kong database name
    KONG_PG_USER: <DB_USER> # Kong database user ( Must have permission to create database )
    KONG_PG_PASSWORD: <DB_PASSWORD> # Kong database password
    KONG_PG_PORT: <DB_PORT> # Database port ( String )
```

- `Konga`
```yaml
    DB_ADAPTER: postgres # Kind of database kong using, we recommend using postgresql
    DB_HOST: <HOST> # Database host
    DB_DATABASE: <DB_NAME> # Konga database name
    DB_USER: <DB_USER> # Kong database user 
    DB_PASSWORD: <DB_PASSWORD> # Kong database password
    DB_PORT: <DB_PORT> # Database port ( String )
```
```cmd
  $ kubectl apply -f /3-kong-db-install/1.namespace.yaml
  $ kubectl apply -f /3-kong-db-install/2.kong-secret.yaml
  $ kubectl apply -f /3-kong-db-install/3.kong-install.yaml
```
- In addition, you can install konga to manage the kong ingress controller.
- To start konga-db, apply file included inside the installation folder [4-konga-db-install](/4-konga-db-install/1-konga-db-install.yaml).
```cmd
  $ kubectl apply -f /4-konga-db-install/1.konga-db-install.yaml
```
## PORT

The Gateway will be available on the following ports on your server:\
These are the default settings, you can change it in [1-kong-dbless-install](/1-kong-dbless-install/2.kong-install.yaml) or [3-kong-db-install](/3-kong-db-install/3-kong-install.yaml) before install. \
`:31313` on which Kong listens for incoming HTTP traffic from your clients, and forwards it to your upstream services.\
`:31314` on which Kong listens for incoming HTTPS traffic from your clients, and forwards it to your upstream services.\
`:31315` on which the Admin API used to configure Kong HTTP listens.\
`:31316` on which the Konga used to configure Kong on Dashboard HTTP listens.


## Testing connectivity to Kong Gateway
If everything is setup correctly, making a request to Kong Gateway should return back a HTTP 404 Not Found status code:
```cmd
curl -i <YOUR_SERVER>:31313
```
Response:
```cmd
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 48
X-Kong-Response-Latency: 0
Server: kong/3.0.0

{"message":"no Route matched with those values"}

```

## Deploy a test upstream HTTP application
To proxy requests, you need an upstream application to proxy to. Deploying this test server provides a simple application that returns information about the Pod it’s running in:
[5-plugin-ingresscontroller/1-service-test](5-plugin-ingresscontroller/1-service-test.yaml)
```cmd
kubectl apply -f /5-plugin-ingresscontroller/1-service-test.yaml
```

## Add routing configuration
Create routing configuration to proxy /test requests to the test server: [5-plugin-ingresscontroller/3-kong-ingress](5-plugin-ingresscontroller/3-kong-ingress.yaml)
```cmd
kubectl apply -f /5-plugin-ingresscontroller/3-kong-ingress.yaml
```
If everything is setup correctly, making a request to Kong Gateway with path you config in ingress should return back a HTTP 200 status code:
```cmd
curl -i <KONG_SERVER>:31313/test/health/check
```
Response:
```cmd
{"status":"ok"}
```

## Using plugins in Kong
Setup a KongPlugin resource:
In this example we will use the `rate limit plugin`, this is a plugin that limits the number of access requests in a specific time.

You can change the config parameters for this plugin here: [5-plugin-ingresscontroller/2-create-rate-limit-plugin](/5-plugin-ingresscontroller/2-create-rate-limit-plugin.yaml)\
`rate limit plugin`
```yaml
metadata:
  name: rate-limiting-example # plugin name
  namespace: kong-dbless
config:
  second: 300                   # limited number of requests per second
  minute: 18000                   # limited number of requests per minute
  #hour: 100000                 # limited number of requests per hour
  policy: local
plugin: rate-limiting         # plugin
```
```cmd
kubectl apply -f /5-plugin-ingresscontroller/2-create-rate-limit-plugin.yaml
```

To specify which service to apply the plugin to, we need to configure it in the annotation on [5-plugin-ingresscontroller/4-ingress-with-plugin](/5-plugin-ingresscontroller/4-ingress-with-plugin.yaml)
```yaml
annotations:
  konghq.com/plugins: rate-limiting-example  # plugin name
```
Now we need update ingress [5-plugin-ingresscontroller/4-ingress-with-plugin](/5-plugin-ingresscontroller/4-ingress-with-plugin.yaml)
```cmd
kubectl apply -f /5-plugin-ingresscontroller/4-ingress-with-plugin.yaml
```
## Kong Custom Configuration
### Custom timeout kong ingress
In some specific cases, we need to configure to change some default values of kong, specifically in this case, when the service uses websocket, it is necessary to remove the timeout request limit of kong ( default is 60s ). So need to create a custom resource of kong to config [6-kong-customconfig/1-custom-time-out-kong](/6-kong-customconfig/1-custom-time-out-kong-ingresss.yaml).
```yaml
kind: KongIngress
apiVersion: configuration.konghq.com/v1
metadata:
  name: timeout-kong-ingress
  namespace: kong-dbless
  annotations:
    kubernetes.io/ingress.class: "kong"
proxy:
  protocol: http
  connect_timeout: 360000
  read_timeout: 360000
  write_timeout: 360000
```
And then we need to set annotations on ingress and service that we need to apply.\
[Ingress](/5-plugin-ingresscontroller/3-kong-ingress.yaml)
```cmd
annotations:
    kubernetes.io/ingress.class: "kong"
```
[Service](/5-plugin-ingresscontroller/1-service-test.yaml)
```cmd
annotations:
    konghq.com/override: timeout-kong-ingress
```

```cmd
curl <KONG_SERVER>:31315/services -k
```
Response:
```json
{
  "data": [
    {
      "retries": 5,
      "enabled": true,
      "port": 80,
      "path": "/",
      "client_certificate": null,
      "protocol": "http",
      "connect_timeout": 360000,
      "read_timeout": 360000,
      "updated_at": 1671099901,
      "ca_certificates": null,
      "id": "9310189f-44a4-5813-bd1e-2bd8d03ffa1b",
      "write_timeout": 360000,
      "tags": null,
      "host": "hello-svc.kong-dbless.8080.svc",
      "tls_verify": null,
      "created_at": 1671099901,
      "name": "kong-dbless.hello-svc.pnum-8080",
      "tls_verify_depth": null
    }
  ],
  "next": null
}
```

Note: Custom resources that are applied on services must be in the same namespace.

## License

```
Copyright 2016-2022 Kong Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

[kong-url]: https://konghq.com/
[kong-logo]: https://konghq.com/wp-content/uploads/2018/05/kong-logo-github-readme.png
[kong-benefits]: https://konghq.com/wp-content/uploads/2018/05/kong-benefits-github-readme.png
[kong-master-builds]: https://hub.docker.com/r/kong/kong/tags
[badge-action-url]: https://github.com/Kong/kong/actions
[badge-action-image]: https://github.com/Kong/kong/workflows/Build%20&%20Test/badge.svg

[busted]: https://github.com/Olivine-Labs/busted
[luacheck]: https://github.com/mpeterv/luacheck