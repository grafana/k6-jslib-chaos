> ### ⚠️ This is a proof of concept
>
> As this is a proof of concept, the API may break in the future. USE AT YOUR OWN RISK!

# k6-chaos

This repo contains a [collection of JavaScript helpers](./src/chaos.js) to run chaos experiments *effortlessly* on top of [k6](https://k6.io).

# Getting started

The first thing you need is a custom k6 binary with the extensions required to use this library: [xk6-kubernetes](https://github.com/grafana/xk6-kubernetes). 

You can also download the required k6 binary from the [releases page](https://github.com/grafana/k6-jslib-chaos/releases).


# APIs

## Helpers

The `helpers.js` module offers helper functions and clases to facilitate the setup of test scenariors

### DeploymentHelper

The `DeploymentHelper` class facilitates deploying applications and exposing them as services.

Methods

`constructor` creates a helper for deploying an application

      Parameters:
        client: k8s client from xk6-kubernetes
        name: name of the application
        namespace: target namespace
        image: image of the application
        replicas: number of replicas
        options:
          - app: value of the 'app' label used as pod selector. Defaults to the name
          - port: port to expose in the application's container. Defaults to '80' 

`deploy` deploys the application and waits for the replicas to be ready

`expose` exposes the application as a service

      Parameters
        options: additional parameters for configuring the service
          - port: port to expose the service. Defaults to '80'
          - name: servie name. Defaults to the application's name

`getPods` returns a list with the names of the deployment pods


## KubernetesChaos

The `KubernetesChaos` class offers methods for introducing faults in kubernetes resources (pods, jobs, etcetera)

Methods

`constructor`: creates an instance of KubernetesChaos

      Parameters
        client: k8s client from xk6-kubernetes
        namespace: namespace to inject chaos into

`killNamespace`: kill the namespace


## Node disruptor

The `NodeDisruptor` disrupts nodes

Methods:

`constructor`: creates a node distuptor

    Parameters:
     client: k8s client from xk6-kubernetes
     options: options
        - name: name of the attack (used as stress job name prefix)
        - namespace: namespace where stress jobs will be started
        - auto_clean: automatically delete stress jobs when attacks ends (defaults to true)

`stress`: stressing a list of nodes by exhausting resources

    Parameters:
      nodes: list of nodes to stress
      options: controls the stress attack
        - cores: sets the number of CPU concurrent stress processes to start
        - cpu_Load: sets the CPU load per stress process
        - duration: sets the duration of the stress attack. E.g. '5m'

`clean`: clean jobs started by the disruptor (if auto_cleanup was set to false)

## Pod Disruption

The `PodDisruptor` class allows disruption a Pod

Methods:

`constructor`: creates an disruptor for a pod

    Parameters:
      client: k8s client from xk6-kubernetes
      pod: name
      namespace:
      options: options

`kill`: kill pod

`slowdownNetwork`: delays network traffic for the pod 

    Parameters:
      options: controls the delay attack
        - delay: average delay in network packages (in milliseconds)
        - variation: variation in the delay (in milliseconds)
        - duration: duration of the disruption 

## Examples

The [./exaples](./examples) forlder contains examples of using `k6-chaos`. 


### Running examples
Notice `k6-chaos` needs the image `grafana/k6-agent`. This image can build using the command

```bash
make container
```

If you are using a local cluster (e.g Kind or Minikube, ) the `grafana/k6-chaos` available in the cluster where the test application will run. 

If using `kind` the following command make the image in the cluster

```
kind load docker-image grafana/k6-chaos`
```

If using `minikube` the following command makes the image available in the cluster:

```bash
minikube image load grafana/k6-chaos
```

## Kill pod in a deployment

The [kill-deployment-pod.js example](examples/kill-deployment-pod.js) shows how `PodDisruptor` can be used for testing the effect of killing an instance of a deloyment. When executed, we can see how the requests are momentarely affected until the pod is restarted:


```bash
WARN[0024] Request Failed                                error="Get \"http://172.18.255.200\": dial tcp 172.18.255.200:80: connect: connection refused"
WARN[0024] Request Failed                                error="Get \"http://172.18.255.200\": dial tcp 172.18.255.200:80: connect: connection refused"
WARN[0025] Request Failed                                error="Get \"http://172.18.255.200\": dial tcp 172.18.255.200:80: connect: connection refused"
WARN[0026] Insufficient VUs, reached 100 active VUs and cannot initialize more  executor=constant-arrival-rate scenario=load
WARN[0026] Request Failed                                error="Get \"http://172.18.255.200\": dial tcp 172.18.255.200:80: connect: connection refused"
WARN[0030] Request Failed                                error="Get \"http://172.18.255.200\": dial tcp 172.18.255.200:80: connect: no route to host"
WARN[0030] Request Failed                                error="Get \"http://172.18.255.200\": dial tcp 172.18.255.200:80: connect: no route to host"

running (01m34.0s), 000/101 VUs, 8831 complete and 0 interrupted iterations
load  ✓ [======================================] 000/100 VUs  1m30s           100.00 iters/s
delay ✓ [======================================] 1 VUs        00m10.0s/10m0s  1/1 shared iters

     ✗ successful request
      ↳  99% — ✓ 8824 / ✗ 6

```

## Introduce delay in network

The [delay-pod.js example](examples/delay-pod.js) shows how `PodDisruptor` can be used for testing the effect of delays in the traffic from an instance of a deloyment. The example uses a deployment of the `httpbin`. The initial setup makes each request to take `100ms`. When executed without any disruption the statistics looks like this.

```
http_req_duration..............: avg=101.66ms min=100.59ms med=101.69ms max=107.24ms p(90)=102.22ms p(95)=102.36ms
```

When the disruption of an additional `200ms` delay is introduced for a period of `30s`, we can see how the stastics are affected (in particular, `p(90)` and `p(95)`):

```bash
     http_req_duration..............: avg=126.21ms min=100.52ms med=101.7ms  max=952.95ms p(90)=301.28ms p(95)=301.8ms 
s
```

## Kill namespace

```javascript
import { Kubernetes } from 'k6/x/kubernetes';
import { KubernetesChaos } from './src/kubernetes.js';

export default function () {

  const k8sClient = new Kubernetes()
  const k8sChaos = new KubernetesChaos(k8sClient)

  // kill namespace.
  k8sChaos.killNamespace();
}
```