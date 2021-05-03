# this is me learning envoy

Diagrams: https://asciiflow.com#/

## Example 1

In this example we will add envoy to an existing microservice (in this case it's a webserver) in order to expose metrics on incoming requests.

```
    │
    │
    │
    │  Traffic
    │
    │
    │
    ▼
┌─────────────┐      ┌─────────────┐
│  :10000     │      │             │
│             │      │             │
│    Envoy    ├─────►│    Nginx    │
│             │      │             │
│             │      │ :80         │
└─────────────┘      └─────────────┘
```

### Plain deployment without envoy

`example1/deployment-plain.yaml` deploy:

- An ingress pointing to the service `nginx-svc` on port 80
- The Service `nginx-svc` pointing to the deployment `nginx-deployment` on port 80
- The deployment `nginx-deployment` that is an nginx listinging on port 80

To deploy edit `example1/deployment-plain.yaml`: and change `<our-domain>` to something that makes sense, and:

```
kubectl create ns example
kubectl -n example apply -f example1/deployment-plain.yaml
```

To check the status: `curl -I https://envoy.<your-domain>`

### Adding envoy to manage incoming traffic

What we will do:

1. Add envoy as a sidecar, point the service to envoy and envoy to nginx: `example1/deployment-envoy-step1.yaml` and `example1/envoy-config.yaml`
2. Expose the envoy metrics to prometheus: `example1/deployment-envoy-step2.yaml`

#### Step 1

In step 1 we have added a second (sidecar) container to the deployment: envoy with its config (see `example1/envoy-config.yaml`). Envoy istens on port 10000 and redirects to port 80 (nginx). We also have changed the service to point to port 10000, i.e. to envoy.

#### Step 2

In step 2 we have added a second port (9901) to our service, as well as the scrape-config to notify prometheus that it can find the metrics under nginx-svc:9901/stats.

### Querying metrics with prometheus

- `envoy_cluster_internal_upstream_rq{envoy_cluster_name="some_service",kubernetes_namespace="example1"}`
- e.g. for the throughput per second: `sum(rate(envoy_cluster_internal_upstream_rq_time_count[2m]))`
- `envoy_cluster_internal_upstream_rq_xx` returns `envoy_response_code_class` being `2`(for 2xx), `5` (for 5xx).
- e.g. rate per second of 2xx: `rate(envoy_cluster_internal_upstream_rq_xx{envoy_cluster_name="some_service",kubernetes_namespace="example1",envoy_response_code_class="2"}[5m])`
- `envoy_cluster_internal_upstream_rq_time_bucket` returns the response times in buckets

### Handling outgoing traffic with Envoy

Unfortunately managing outgoing traffic with Envoy can not be done only by configuration: if a service calls another service (or an external URL) there is no easy way to transparently routing this traffic through Envoy. Instead, we add a listeners to the Envoy configuration (a port) that acts as a proxy for one specific service that will be then called through the proxy, and allow us to collect metrics about that outbound traffic.

#### Example 2: outbound http traffic

In this example we will pass outgoing traffic through Envoy to reach the httpbin service running on the cluster in the same namespace.

```
┌─────────────────────┐         ┌────────────────────┐        ┌──────────────────┐
│                     │         │                    │        │                  │
│                     │         │                    │        │                  │
│                     ├────────►│:10001              ├────────► :80              │
│       nginx         │         │       Envoy        │        │     httpbin      │
│                     │         │                    │        │                  │
│                     │         │                    │        │                  │
│                     │         │                    │        │                  │
└─────────────────────┘         └────────────────────┘        └──────────────────┘

```

1. Install httpbin: `kubectl -n example apply -f httpbin/deployment.yaml`
1. Add a listener to Envoy on port 10001, forwarding its traffic to httpbin: ``kubectl -n example apply -f example2/`
1. Get the name of the nginx pod: `kubectl -n example get pods`
1. Call httpbin through the listener: `kubectl -n example exec -it <nginx-pod-name> nginx -- curl http://localhost:10001/ip`

#### Example 3. outbound https traffic

In this example we will cover two options:

1. The microservice connects to Envoy with http, Envoy connect to the endpoint using https
1. The microservice connects to Envoy with https, Envoy connect to the endpoint using https (passthrough)

```
                                                                      ┌─────────────────┐
                                                                      │                 │
                                                                      │                 │
                                                                      │                 │
                                                                      │    httpbin.org  │
                                                                      │                 │
                                                                      │                 │
                                                                      │                 │
                                                                      └─────────────────┘
                                                                               ▲
                                                                               │
                                                                               │
                                                                               │
┌────────────────────┐          ┌───────────────────┐                          │
│                    │          │                   │                          │
│              (http)│          │                   │                          │
│                    ├──────────► :10002            │ (https)                  │
│      nginx         │          │     Envoy         ├──────────────────────────┘
│                    │          │                   │
│             (https)│          │                   │
│                    ├──────────► :10003            │
│                    │          │                   │
└────────────────────┘          └───────────────────┘
```

In `envoy-config-egress-https.yaml` we habe added two listeners:

- `listener_egress_http_https` on port 10002
- `listener_egress_https_passthrough` on port 10003

and two clusters:

- `egress_service_http_https` forwarding incoming http traffic to https://httpbin.org 
- `egress_service_https_passthrough` forwarding incomng https traffic to https://httpbin.org

Depending on the use case either config may be intersting:

1. `egress_service_http_https`: you don't want to deal with TLS in your application
1. `egress_service_https_passthrough`: your client is already up-and-running any you just want metrics

1. Add the listenerer to Envoy on port 10002 (http->https) and 10003 (https passthrough): ``kubectl -n example apply -f example3/`
1. Get the name of the nginx pod: `kubectl -n example get pods`
1. Call httpbin.org through the http->https listener: `kubectl -n example exec -it <nginx-pod-name> nginx -- curl http://localhost:10002/ip`
1. Call httpbin.org through the https-passthrough listener: `kubectl -n example exec -it <nginx-pod-name> nginx -- curl https://localhost:10003/ip`

## Example4 :more on observability

For this example we will create a different setup:

1. Create the namespace: `kubectl create ns observability`
1. Change the ingress host to suit your needs (in `example4/deployment.yaml`)
1. Deploy: `kubectl -n observability apply -f example4/`

This is how the "chain" is set-up:

- the ingress points to the service (port 80)
- the service forwards port 80 to port 10000 (envoy `listener_ingress`)
- the envoy cluster `ingress_service` forwards to `localhost:38000` (toxyproxy). Please note that with toxiproxy we should use ports outside of the ephemeral port range (see also https://github.com/Shopify/toxiproxy#2-populating-toxiproxy)
- toxiproxy listens on `localhost:38000` and forwards to `httpbin:80` (see `example4/toxiproxy.yaml` for the proxy config)
- run a loop querying the service: `while true; do sleep 5; curl https://observability.home.asksven.io/ip; echo -e '\n\n\n\n'$(date);done`

### Metrics

Now that we have everything in place it's time to build a dashboard. Since we have Envoy's Prometheus metrics in place we can build a dashboard showing:

#### Throughput
- Rate per second: `rate(envoy_cluster_internal_upstream_rq_time_count{kubernetes_namespace="observability",envoy_cluster_name="ingress_service"}[2m])`
- Rate per second < 10ms: `rate(envoy_cluster_upstream_rq_time_bucket{le="10",envoy_cluster_name="ingress_service",kubernetes_namespace="observability"}[2m])`

#### Error rate
- Rate of successful requests per second: `rate(envoy_cluster_internal_upstream_rq_xx{envoy_response_code_class="2",kubernetes_namespace="observability",envoy_cluster_name="ingress_service"}[5m])`
- Error rate per second: `rate(envoy_cluster_internal_upstream_rq_xx{envoy_response_code_class!="2",kubernetes_namespace="observability",envoy_cluster_name="ingress_service"}[5m])`

### Creating some chaos

Using toxiproxy we can now introduce some chaos:

- connect to toxiproxy: `kubectl -n observability exec -it <pod-name> -c toxiproxy -- /bin/sh`
- add some latency: `/go/bin/toxiproxy-cli toxic add httpbin -n myLatency -type latency --attribute latency=1000 --attribute jitter=1000 --attribute toxicity=0.1`
- add timeouts `/go/bin/toxiproxy-cli toxic add httpbin -n myTimeout -type timeout --attribute timeout=5000 --attribute toxicity=0.1` **Note:** at this moment it seems that "timeout" does not support the `toxicity` atribute. At least during my tests all requests timed-out once I created that toxic
- remove timeouts: `/go/bin/toxiproxy-cli toxic delete -n myTimeout httpbin`
- remove latency: `/go/bin/toxiproxy-cli toxic delete -n myLatency httpbin`
- add a slighter latency: `/go/bin/toxiproxy-cli toxic add httpbin -n myLatency -type latency --attribute latency=10 --attribute jitter=10 --attribute toxicity=0.5`
