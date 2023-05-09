---
sectionid: lab2-network
sectionclass: h2
title: Networking and Scaling
parent-id: lab-clusterapp
---

In this section we'll see how OSToy uses intra-cluster networking to separate functions by using microservices and visualize the scaling of pods.

Let's review how this application is set up...

![OSToy Diagram](media/managedlab/4-ostoy-arch.png)

As can be seen in the image above we have defined at least 2 separate pods, each with its own service.  One is the frontend web application (with a service and a publicly accessible route) and the other is the backend microservice with a service object created so that the frontend pod can communicate with the microservice (across the pods if more than one).  Therefore this microservice is not accessible from outside this cluster (or from other namespaces/projects, if configured, due to OpenShifts' network policy, [ovs-networkpolicy](https://docs.openshift.com/container-platform/latest/networking/network_policy/about-network-policy.html#nw-networkpolicy-about_about-network-policy)).  The sole purpose of this microservice is to serve internal web requests and return a JSON object containing the current hostname and a randomly generated color string.  This color string is used to display a box with that color displayed in the tile titled "Intra-cluster Communication".

### Networking

Click on *Networking* in the left menu. Review the networking configuration.

{% collapsible %}

The right tile titled "Hostname Lookup" illustrates how the service name created for a pod can be used to translate into an internal ClusterIP address. Enter the name of the microservice following the format of `my-svc.my-namespace.svc.cluster.local` which we created in our `ostoy-microservice.yaml` as seen here:

```
apiVersion: v1
kind: Service
metadata:
  name: ostoy-microservice-svc
  labels:
    app: ostoy-microservice
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: ostoy-microservice
```

In this case we will enter: `ostoy-microservice-svc.ostoy.svc.cluster.local`

We will see an IP address returned.  In our example it is `172.30.165.246`.  This is the intra-cluster IP address; only accessible from within the cluster.

![ostoy DNS](media/managedlab/20-ostoy-dns.png)


The ostoy-frontend service is exposed via an Openshift route object that has a DNS name. 
```
$ oc describe route ostoy-route
Name:                   ostoy-route
Namespace:              ostoy
Created:                12 hours ago
Labels:                 <none>
Annotations:            kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"route.openshift.io/v1","kind":"Route","metadata":{"annotations":{},"name":"ostoy-route","namespace":"ostoy"},"spec":{"to":{"kind":"Service","name":"ostoy-frontend-svc"}}}

                        openshift.io/host.generated=true
Requested Host:         ostoy-route-ostoy.apps.arouse2.eastus2.aroapp.io
                           exposed on router default (host router-default.apps.arouse2.eastus2.aroapp.io) 12 hours ago
Path:                   <none>
TLS Termination:        <none>
Insecure Policy:        <none>
Endpoint Port:          <all endpoint ports>

Service:        ostoy-frontend-svc
Weight:         100 (100%)
Endpoints:      <none>
  
$ nslookup ostoy-route-ostoy.apps.arouse2.eastus2.aroapp.io
Server:         172.21.112.1
Address:        172.21.112.1#53

Non-authoritative answer:
Name:   ostoy-route-ostoy.apps.arouse2.eastus2.aroapp.io
Address: 20.7.53.216

```



The IP address corresponding to that DNS name is configured on the Azure Load Balancer that is automatically created when the ARO cluster is created. 
```
$ az network lb list -g  aro-infra-lhff47hl-arouse2
Location    Name                    ProvisioningState    ResourceGroup               ResourceGuid
----------  ----------------------  -------------------  --------------------------  ------------------------------------
eastus2     arouse2-xs5pd           Succeeded            aro-infra-lhff47hl-arouse2  95dc1dda-096e-4018-b48c-d3805df76960
eastus2     arouse2-xs5pd-internal  Succeeded            aro-infra-lhff47hl-arouse2  653e7daa-4495-4162-8cd0-d696b76807f9

$ az network lb frontend-ip list -g aro-infra-lhff47hl-arouse2 --lb-name arouse2-xs5pd
Name                              PrivateIPAllocationMethod    ProvisioningState    ResourceGroup
--------------------------------  ---------------------------  -------------------  --------------------------
public-lb-ip-v4                   Dynamic                      Succeeded            aro-infra-lhff47hl-arouse2
a9ed4779105ff463986a597e0836f37d  Dynamic                      Succeeded            aro-infra-lhff47hl-arouse2

$ az network lb frontend-ip show -g aro-infra-lhff47hl-arouse2 --lb-name arouse2-xs5pd -n public-lb-ip-v4 -o yamlc | grep publicIPAddress
publicIPAddress:
  id: /subscriptions/sub-id/resourceGroups/aro-infra-lhff47hl-arouse2/providers/Microsoft.Network/publicIPAddresses/arouse2-xs5pd-pip-v4

$ az network lb frontend-ip show -g aro-infra-lhff47hl-arouse2 --lb-name arouse2-xs5pd -n a9ed4779105ff463986a597e0836f37d -o yamlc | grep publicIPAddress
publicIPAddress:
  id: /subscriptions/sub-id/resourceGroups/aro-infra-lhff47hl-arouse2/providers/Microsoft.Network/publicIPAddresses/arouse2-xs5pd-default-v4

$ az network public-ip list -g aro-infra-lhff47hl-arouse2
Name                      ResourceGroup               Location    Zones    Address        IdleTimeoutInMinutes    ProvisioningState
------------------------  --------------------------  ----------  -------  -------------  ----------------------  -------------------
arouse2-xs5pd-default-v4  aro-infra-lhff47hl-arouse2  eastus2              20.7.53.216    4                       Succeeded
arouse2-xs5pd-pip-v4      aro-infra-lhff47hl-arouse2  eastus2              20.230.87.192  4                       Succeeded
```



That IP is associated with the router-default kubernetes service that provides the routes functionality (similar to kubernetes ingress, implemented with a pair of HA proxy pods).
```  
$ oc get svc -n openshift-ingress
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
router-default            LoadBalancer   172.30.152.88    20.7.53.216   80:31745/TCP,443:32207/TCP   18h
router-internal-default   ClusterIP      172.30.117.108   <none>        80/TCP,443/TCP,1936/TCP      18h

$ oc describe svc router-default -n openshift-ingress
Name:                     router-default
Namespace:                openshift-ingress
Labels:                   app=router
                          ingresscontroller.operator.openshift.io/owning-ingresscontroller=default
Annotations:              traffic-policy.network.alpha.openshift.io/local-with-fallback:
Selector:                 ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       172.30.152.88
IPs:                      172.30.152.88
IP:                       20.7.53.216
LoadBalancer Ingress:     20.7.53.216
Port:                     http  80/TCP
TargetPort:               http/TCP
NodePort:                 http  31745/TCP
Endpoints:                10.129.2.13:80,10.131.0.15:80
Port:                     https  443/TCP
TargetPort:               https/TCP
NodePort:                 https  32207/TCP
Endpoints:                10.129.2.13:443,10.131.0.15:443
Session Affinity:         None
External Traffic Policy:  Local
HealthCheck NodePort:     31943
Events:                   <none>

$ oc get pods -n openshift-ingress
NAME                              READY   STATUS    RESTARTS   AGE
router-default-65bddfdd8d-b9q8v   1/1     Running   0          18h
router-default-65bddfdd8d-mkdnb   1/1     Running   0          17h

$ oc describe pods router-default-65bddfdd8d-b9q8v -n openshift-ingress
Name:                 router-default-65bddfdd8d-b9q8v
Namespace:            openshift-ingress
Priority:             2000000000
Priority Class Name:  system-cluster-critical
Node:                 arouse2-xs5pd-worker-eastus21-fjsgd/10.1.2.6
Start Time:           Mon, 08 May 2023 17:25:36 -0700
Labels:               ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
                      ingresscontroller.operator.openshift.io/hash=6968ddb865
                      pod-template-hash=65bddfdd8d
Annotations:          k8s.v1.cni.cncf.io/network-status:
                        [{
                            "name": "openshift-sdn",
                            "interface": "eth0",
                            "ips": [
                                "10.131.0.15"
                            ],
                            "default": true,
                            "dns": {}
                        }]
                      k8s.v1.cni.cncf.io/networks-status:
                        [{
                            "name": "openshift-sdn",
                            "interface": "eth0",
                            "ips": [
                                "10.131.0.15"
                            ],
                            "default": true,
                            "dns": {}
                        }]
                      openshift.io/scc: restricted
                      unsupported.do-not-use.openshift.io/override-liveness-grace-period-seconds: 10
Status:               Running
IP:                   10.131.0.15
IPs:
  IP:           10.131.0.15
Controlled By:  ReplicaSet/router-default-65bddfdd8d
Containers:
  router:
    Container ID:   cri-o://8beb16ee1858ea7e101a33fc28c41d05d8eb629aa60772f8724377111299c055
    Image:          quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:37a08ed049544ebfb0bd282df201edc7bec5250ad3911a02505a97cbece88374
    Image ID:       quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:37a08ed049544ebfb0bd282df201edc7bec5250ad3911a02505a97cbece88374
    Ports:          80/TCP, 443/TCP, 1936/TCP
    Host Ports:     0/TCP, 0/TCP, 0/TCP
    State:          Running
      Started:      Mon, 08 May 2023 17:25:38 -0700
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:      100m
      memory:   256Mi
    Liveness:   http-get http://:1936/healthz delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:  http-get http://:1936/healthz/ready delay=0s timeout=1s period=10s #success=1 #failure=3
    Startup:    http-get http://:1936/healthz/ready delay=0s timeout=1s period=1s #success=1 #failure=120
    Environment:
      DEFAULT_CERTIFICATE_DIR:                   /etc/pki/tls/private
      DEFAULT_DESTINATION_CA_PATH:               /var/run/configmaps/service-ca/service-ca.crt
      RELOAD_INTERVAL:                           5s
      ROUTER_ALLOW_WILDCARD_ROUTES:              false
      ROUTER_CANONICAL_HOSTNAME:                 router-default.apps.arouse2.eastus2.aroapp.io
      ROUTER_CIPHERS:                            ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
      ROUTER_CIPHERSUITES:                       TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
      ROUTER_DISABLE_HTTP2:                      true
      ROUTER_DISABLE_NAMESPACE_OWNERSHIP_CHECK:  false
      ROUTER_DOMAIN:                             apps.arouse2.eastus2.aroapp.io
      ROUTER_LOAD_BALANCE_ALGORITHM:             random
      ROUTER_METRICS_TLS_CERT_FILE:              /etc/pki/tls/metrics-certs/tls.crt
      ROUTER_METRICS_TLS_KEY_FILE:               /etc/pki/tls/metrics-certs/tls.key
      ROUTER_METRICS_TYPE:                       haproxy
      ROUTER_SERVICE_NAME:                       default
      ROUTER_SERVICE_NAMESPACE:                  openshift-ingress
      ROUTER_SET_FORWARDED_HEADERS:              append
      ROUTER_TCP_BALANCE_SCHEME:                 source
      ROUTER_THREADS:                            4
      SSL_MIN_VERSION:                           TLSv1.2
      STATS_PASSWORD_FILE:                       /var/lib/haproxy/conf/metrics-auth/statsPassword
      STATS_PORT:                                1936
      STATS_USERNAME_FILE:                       /var/lib/haproxy/conf/metrics-auth/statsUsername
    Mounts:
      /etc/pki/tls/metrics-certs from metrics-certs (ro)
      /etc/pki/tls/private from default-certificate (ro)
      /var/lib/haproxy/conf/metrics-auth from stats-auth (ro)
      /var/run/configmaps/service-ca from service-ca-bundle (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7dnlg (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-certificate:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  256cfe9b-2faf-4b2f-a1ef-5675d06b46f8-ingress
    Optional:    false
  service-ca-bundle:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      service-ca-bundle
    Optional:  false
  stats-auth:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  router-stats-default
    Optional:    false
  metrics-certs:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  router-metrics-certs-default
    Optional:    false
  kube-api-access-7dnlg:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
    ConfigMapName:           openshift-service-ca.crt
    ConfigMapOptional:       <nil>
QoS Class:                   Burstable
Node-Selectors:              kubernetes.io/os=linux
                             node-role.kubernetes.io/worker=
Tolerations:                 node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```

{% endcollapsible %}

### Scaling

OpenShift allows one to scale up/down the number of pods for each part of an application as needed.  This can be accomplished via changing our *replicaset/deployment* definition (declarative), by the command line (imperative), or via the web console (imperative). In our *deployment* definition (part of our `ostoy-fe-deployment.yaml`) we stated that we only want one pod for our microservice to start with. This means that the Kubernetes Replication Controller will always strive to keep one pod alive. We can also define pod autoscaling using the [Horizontal Pod Autoscaler](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-autoscaling.html) (HPA) based on load to expand past what we defined. We will do this in a later section of this lab.

{% collapsible %}

If we look at the tile on the left we should see one box randomly changing colors.  This box displays the randomly generated color sent to the frontend by our microservice along with the pod name that sent it. Since we see only one box that means there is only one microservice pod.  We will now scale up our microservice pods and will see the number of boxes change.

To confirm that we only have one pod running for our microservice, run the following command, or use the web console.

```
$ oc get pods
NAME                                   READY     STATUS    RESTARTS   AGE
ostoy-frontend-679cb85695-5cn7x       1/1       Running   0          1h
ostoy-microservice-86b4c6f559-p594d   1/1       Running   0          1h
```

Let's change our microservice definition yaml to reflect that we want 3 pods instead of the one we see. Download the [ostoy-microservice-deployment.yaml](yaml/ostoy-microservice-deployment.yaml) and save it on your local machine.

Open the file using your favorite editor. Ex: `vi ostoy-microservice-deployment.yaml`.

Find the line that states `replicas: 1` and change that to `replicas: 3`. Then save and quit.

It will look like this

```
spec:
    selector:
      matchLabels:
        app: ostoy-microservice
    replicas: 3
 ```

Assuming you are still logged in via the CLI, execute the following command:

`oc apply -f ostoy-microservice-deployment.yaml`

Confirm that there are now 3 pods via the CLI (`oc get pods`) or the web console (*Workloads > Deployments > ostoy-microservice*).

See this visually by visiting the OSToy app and seeing how many boxes you now see.  It should be three.

![UI Scale](media/managedlab/22-ostoy-colorspods.png)

Now we will scale the pods down using the command line.  Execute the following command from the CLI:

`oc scale deployment ostoy-microservice --replicas=2`

Confirm that there are indeed 2 pods, via the CLI (`oc get pods`) or the web console.

See this visually by visiting the OSToy App and seeing how many boxes you now see.  It should be two.

Lastly, let's use the web console to scale back down to one pod.  Make sure you are in the project you created for this app (i.e., "ostoy"), in the left menu click *Workloads > Deployments > ostoy-microservice*.  On the left you will see a blue circle with the number 2 in the middle. Click on the down arrow to the right of that to scale the number of pods down to 1.

![UI Scale](media/managedlab/21-ostoy-uiscale.png)

See this visually by visiting the OSToy app and seeing how many boxes you now see.  It should be one.  You can also confirm this via the CLI or the web console.

{% endcollapsible %}
