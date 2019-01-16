# docker-vernemq

## What is VerneMQ?

VerneMQ is a high-performance, distributed MQTT message broker. It scales
horizontally and vertically on commodity hardware to support a high number of
concurrent publishers and consumers while maintaining low latency and fault
tolerance. VerneMQ is the reliable message hub for your IoT platform or smart
products.

VerneMQ is an Apache2 licensed distributed MQTT broker, developed in Erlang.

## How to use this image

### Start a VerneMQ cluster node

    docker run --name vernemq1 -d erlio/docker-vernemq

Somtimes you need to configure a forwarding for ports (on a Mac for example):

    docker run -p 1883:1883 --name vernemq1 -d erlio/docker-vernemq

This starts a new node that listens on 1883 for MQTT connections and on 8080 for MQTT over websocket connections. However, at this moment the broker won't be able to authenticate the connecting clients. To allow anonymous clients use the ```DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on``` environment variable.

    docker run -e "DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on" --name vernemq1 -d erlio/docker-vernemq

### Autojoining a VerneMQ cluster

This allows a newly started container to automatically join a VerneMQ cluster. Assuming you started your first node like the example above you could autojoin the cluster (which currently consists of a single container 'vernemq1') like the following:

    docker run -e "DOCKER_VERNEMQ_DISCOVERY_NODE=<IP-OF-VERNEMQ1>" --name vernemq2 -d erlio/docker-vernemq

(Note, you can find the IP of a docker container using `docker inspect <containername/cid> | grep \"IPAddress\"`).

### Automated clustering on Kubernetes

When running VerneMQ inside Kubernetes, it is possible to cause pods matching a specific label to cluster altogether automatically.
This feature uses Kubernetes' API to discover other peers, and relies on the [default pod service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) which has to be enabled.

Simply set ```DOCKER_VERNEMQ_DISCOVERY_KUBERNETES=1``` in your pod's environment, and expose your own pod name through ```MY_POD_NAME``` . By default, this setting will cause all pods in the same namespace with the ```app=vernemq``` label to join the same cluster. Cluster name (defaults to `cluster.local`), namespace and label settings can be overridden with ```DOCKER_VERNEMQ_KUBERNETES_CLUSTER_NAME```, ```DOCKER_VERNEMQ_KUBERNETES_NAMESPACE``` and ```DOCKER_VERNEMQ_KUBERNETES_APP_LABEL``` respectively.

An example configuration of your pod's environment looks like this:

    env:
      - name: DOCKER_VERNEMQ_DISCOVERY_KUBERNETES
        value: "1"
      - name: MY_POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: DOCKER_VERNEMQ_KUBERNETES_APP_LABEL
        value: "myverneinstance"

When enabling Kubernetes autoclustering, don't set ```DOCKER_VERNEMQ_DISCOVERY_NODE```.

> If you encounter "SSL certification error (subject name does not match the host name)" like below, you may try to set ```DOCKER_VERNEMQ_KUBERNETES_INSECURE``` to "1".
>
> ```text
> kubectl logs vernemq-0
>   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
>                                  Dload  Upload   Total   Spent    Left  Speed
>   0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (51) SSL: certificate subject name 'client' does not match target host name 'kubernetes.default.svc.cluster.local'
>   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
>                                  Dload  Upload   Total   Spent    Left  Speed
>   0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (51) SSL: certificate subject name 'client' does not match target host name 'kubernetes.default.svc.cluster.local'
> vernemq failed to start within 15 seconds,
> see the output of 'vernemq console' for more information.
> If you want to wait longer, set the environment variable
> WAIT_FOR_ERLANG to the number of seconds to wait.
> ...
> ```

### Checking cluster status

To check if the bove containers have successfully clustered you can issue the ```vmq-admin``` command:

    docker exec vernemq1 vmq-admin cluster show
    +--------------------+-------+
    |        Node        |Running|
    +--------------------+-------+
    |VerneMQ@172.17.0.151| true  |
    |VerneMQ@172.17.0.152| true  |
    +--------------------+-------+

If you started VerneMQ cluster inside Kubernetes using ```DOCKER_VERNEMQ_DISCOVERY_KUBERNETES=1```, you can execute ```vmq-admin``` through ```kubectl```:

    kubectl exec vernemq-0 -- vmq-admin cluster show
    +---------------------------------------------------+-------+
    |                       Node                        |Running|
    +---------------------------------------------------+-------+
    |VerneMQ@vernemq-0.vernemq.default.svc.cluster.local| true  |
    |VerneMQ@vernemq-1.vernemq.default.svc.cluster.local| true  |
    +---------------------------------------------------+-------+

All ```vmq-admin``` commands are available. See https://vernemq.com/docs/administration/ for more information.

### Automated clustering on Consul + Nomad

When running VerneMQ inside Nomad, it is possible to leverage Consul service catalog, populated by Nomad, to make the VerneMQ containers cluster altogether automatically.

Set ```DOCKER_VERNEMQ_DISCOVERY_CONSUL=1``` in your Job's environment.
Make sure the `epmd` port (`4369`) is registered in a specific service, and set this service name into the `DOCKER_VERNEMQ_DISCOVERY_CONSUL_SERVICE_NAME` environment variable and has a health-check defined (so that each VerneMQ container is registered only when it is actually really listening on this port)

Here are the important snippets from your Nomad job to run this container (this job is incomplete, only the autodiscovery settings are illustrated here):

```hcl
job "vernemq" {
  type = "service"

  group "mqtt" {
    task "mqtt" {
      driver = "docker"
      config {
        # Because of the autodiscovery, name and IP, we use host networking
        # (we get the same kind of routing than clusterIP in Kubernetes)
        network_mode = "host"

        port_map {
          epmd                = 4369
        }
      }

      env {
        "DOCKER_IP_ADDRESS"                  = "${attr.unique.network.ip-address}"
        "DOCKER_VERNEMQ_NODENAME"            = "VerneMQ@${attr.unique.network.ip-address}"

        # VerneMQ clustering (Consul+Nomad)
        "DOCKER_VERNEMQ_DISCOVERY_CONSUL"              = "1"
        "DOCKER_VERNEMQ_DISCOVERY_CONSUL_HOST"         = "${attr.unique.network.ip-address}"
        "DOCKER_VERNEMQ_DISCOVERY_CONSUL_SERVICE_NAME" = "mqtt-discovery"
        "DOCKER_VERNEMQ_DISCOVERY_CONSUL_STAGGER_IND"  = "${NOMAD_ALLOC_INDEX}"
      }

      resources {
        network {
          port "epmd" {
            static = 4369
          }
        }
      }

      service {
        # This service is used only to expose the dynamic address of one of the MQTT broker
        # To make it easy for the other ones to join
        name = "mqtt-discovery"
        port = "epmd"

        check {
          type     = "tcp"
          port     = "epmd"
          interval = "10s"
          timeout  = "2s"
          initial_status = "critical"
        }
      }
    }
  }
}
```

The other useful env var for the autodiscovery are:

- `DOCKER_VERNEMQ_DISCOVERY_CONSUL_HOST` the host where to make API calls to Consul service catalog
- `DOCKER_VERNEMQ_DISCOVERY_CONSUL_PORT` let you customize Consul API port. Requests to the Consul API will be sent to port `8500` by default.
- `DOCKER_VERNEMQ_DISCOVERY_CONSUL_STAGGER_IND` is useful to add a time offset between the various allocations: the second one wait a little bit to make sure the first one ius ready to listen for joining nodes. Then node n+1 wait a little bit more than node n, and so on and so forth.

Known limitations:

- Consul TLS and ACLs are not yet handled
- host network is not ideal, and we plan to improve that later

### VerneMQ Configuration

All configuration parameters that are available in `vernemq.conf` can be defined
using the `DOCKER_VERNEMQ` prefix followed by the confguration parameter name.
E.g: `allow_anonymous=on` is `-e "DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on"` or
`allow_register_during_netsplit=on` is
`-e "DOCKER_VERNEMQ_ALLOW_REGISTER_DURING_NETSPLIT=on"`. All available configuration
parameters can be found on https://vernemq.com/docs/configuration/.

#### Logging

VerneMQ store crash and error log files in `/var/log/vernemq/`, and, by default, 
doesn't write console log to the disk to avoid filling the container disk space.
However this behaviour can be changed by setting the environment variable `DOCKER_VERNEMQ_LOG__CONSOLE` to `both` 
which tells VerneMQ to send logs to stdout and `/var/log/vernemq/console.log`.
For more information please see VerneMQ logging documentation: https://docs.vernemq.com/configuring-vernemq/logging

#### Remarks

Some of our configuration variables contain dots `.`. For example if you want to
adjust the log level of VerneMQ you'd use `-e
"DOCKER_VERNEMQ_LOG.CONSOLE.LEVEL=debug"`. However, some container platforms
such as Kubernetes don't support dots and other special characters in
environment variables. If you are on such a platform you could substitute the
dots with two underscores `__`. The example above would look like `-e
"DOCKER_VERNEMQ_LOG__CONSOLE__LEVEL=debug"`.

There some some exceptions on configuration names contains dots. You can see follow examples:

format in vernemq.conf | format in environment variable name
---------------------- | ------------------------------------
 `vmq_webhooks.pool_timeout = 60000` | `DOCKER_VERNEMQ_VMQ_WEBHOOKS__POOL_timeout=6000`
 `vmq_webhooks.pool_timeout = 60000` | `DOCKER_VERNEMQ_VMQ_WEBHOOKS.pool_timeout=60000`




#### File Based Authentication

You can set up [File Based Authentication](https://vernemq.com/docs/configuration/authentication.html)
by adding users and passwords as environment variables as follows:

`DOCKER_VERNEMQ_USER_<USERNAME>='password'`

where `<USERNAME>` is the username you want to use. This can be done as many times as necessary
to create the users you want. The usernames will always be created in lowercase

*CAVEAT* - You cannot have a `=` character in your password.
