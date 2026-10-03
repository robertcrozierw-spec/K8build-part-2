# K8build-part-2
Follow up project to build out existing K8s the hard way cluster adding additional features.
-DNS server
-Loadbalancer
-IngressController

## DNS Server

We will implement a DNS within our cluster to assist with Service to Service communication.

CoreDNS is an open-source solution that typically comes preconfigured in clusters deployed by Kubeadm. However
as we have manually deployed our cluster we will need to add it ourselves.

### What problem are we solving

Lets take a look at an example as that may better describe it.
We have created a clusterip service to expose an existing nginx pod internally to a linux container within our cluster. 

The test pod and service configurations:

- nginx.yaml (our web server we will try to contact)
- netutils.yaml (this is a test pod that contains basic network tools)
- nginx-clusterip-service.yaml

Once we apply our nginx-clusterip-service we can see that it has an ip configured

````
root@lima-jumpbox:~/k8build-part-2# kubectl get -o wide service 
NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes                ClusterIP   10.0.0.1     <none>        443/TCP   13d   <none>
nginx-clusterip-service   ClusterIP   10.0.0.161   <none>        80/TCP    53m   app=nginx
```

Please note this IP was changed as the original was on the different range, this will be discussed later in the document.

Next, lets exec into our netutils container
````
root@lima-jumpbox:~/k8build-part-2# kubectl exec --stdin --tty netutils -- /bin/bash
````

From here lets test a curl to our nginx server via the cluster-ip service we created

Via IP:
````
netutils:/# curl 10.0.0.161:80

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>
```

Now lets try using a DNS name instead
```
netutils:/# curl nginx-clusterip-service:80
curl: (6) Could not resolve host: nginx-clusterip-service (Could not contact DNS servers)
```

This is our issue, we have no way of communicating with the cluster-ip via DNS and therefore its nginx endpoint. This is problem as when we start to configure larger deployments:
- We would need to know the IP of the service, this is impractical as it assigns the IP at runtime fo the service. Making it more complex to reference the service in the same deployment file.
- DHCP leases exist meaning the IPs will change and require updating the configuration.

Implementing a DNS system should resolve this making for a cleaner environment, more readable and easier to maintain.


### coreDNS parts and how it works

Typically coreDNS will have the following components

DNS service
This is the service that the pods use to interact with DNS pods,this service's Cluster-IP is the IP of our "DNS server"
It needs to match what we define as the DNS server in the kubelet configuration

DNS pods
These are the running DNS server pods that do the actual lookup. Unlike a typical DNS server, these pods usually do not contain or maintain a zone file, instead they gather the information they need from the etcd database via the kube-apiserver. They then monitor the kube-apiserver for any updates or changes and modify their cache as such.

DNS corefile
This is of the type "configmap", it holds the configuration of the DNS, this corefile is mounted onto the DNS pods when they are created.

Actual Pods
When we create a new pod, inside the /etc/resolv.conf we will see the the Cluster-IP of the DNS service as the nameserver. It will use this namesever for its DNS requests. This value is handled by the kubelet configuration.

### Prerequisites

We added the range to the kube-apiserver.service file on the Control Server

root@server:~# cat /etc/systemd/system/kube-apiserver.service
  --service-cluster-ip-range=10.32.0.0/24 \

However after doing this we noticed that main ip of kubernetes cluster IP service remained the same and on a different range
```
NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
kubernetes                ClusterIP   10.0.0.1     <none>        443/TCP        22d
```

This is a problem as this Cluster IP needs to to be on the same range as our services. It turns out changing this IP is a non-trivial task. So instead we will work around it.

Instead of changing the range to match our existing configuration, we will change it to match the kubernetes Cluster IP above

So we should now see this:
```
root@server:~# cat /etc/systemd/system/kube-apiserver.service | grep service-cluster-ip-range
  --service-cluster-ip-range=10.0.0.0/24 \
```

Once this is done we reach another issue, the original certificate generated for our Kube-apiserver was generated for a different IP.
``` 
[WARNING] plugin/kubernetes: Kubernetes API connection failure: Get "https://10.0.0.1:443/version": x509: certificate is valid for 127.0.0.1, 10.32.0.1, not 10.0.0.1
```

Now that we are working with the kubernetes cluster IP above. We will generate a new CSR using this IP instead and then have ti signed by our CA. Once done we will replace it on our master server

Within our ca.conf file we will add the correct IP
```
[kube-api-server_alt_names]
IP.0  = 127.0.0.1
IP.1  = 10.0.0.1
DNS.0 = kubernetes
DNS.1 = kubernetes.default
DNS.2 = kubernetes.default.svc
DNS.3 = kubernetes.default.svc.cluster
DNS.4 = kubernetes.svc.cluster.local
DNS.5 = server.kubernetes.local
DNS.6 = api-server.kubernetes.local

[kube-api-server_distinguished_name]
CN = kubernetes
C  = US
ST = Washington
L  = Seattle
```

As we are just creating one certificate we will need to manually specify the section of the ca.conf when generation the csr
```
openssl req -new -key "kube-api-server.key" -sha256 -config "ca.conf" -section "kube-api-server" -out "kube-api-server.csr"
```

Next we sign the CSR with our CA private key

```
openssl x509 -req -days 3653 -in "kube-api-server.csr" -copy_extensions copyall -sha256 -CA "ca.crt" -CAkey "ca.key" -CAcreateserial -out "kube-api-server.crt"
```
Copy to our master server
```
scp kube-api-server.crt root@server:~/
```

Finally I chose to to directly copy over the existing certificate to keep thing neat
```
cp kube-api-server.crt /var/lib/kubernetes/kube-api-server.crt
```

Restart the systemd service so it starts using the new certificate
```
systemctl restart kube-apiserver
```

As this certificate is also required by etcd, we also replaced it on that side too
```
cp /var/lib/kubernetes/kube-api-server.crt /etc/etcd/kube-api-server.crt
```

## Configure DNS

Looking online I managed to find an existing yaml file with the (mostly) configuration we need.

If we apply the cordns.yaml

Several objects will be created


Configure cluster-ip service range. As clusterDNS is failing

## Results and conclusion

Now when we re-run our test from the beginning
```
netutils-node0:/# curl nginx-clusterip-service:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>
```

We can successfully curl our my-nginx pod via its corresponding cluster-ip service!

## Loadbalancer

Much like DNS a typical cloud instance of a load balancer is already available as this is our own bare metal cluster we will need to implement one ourselves.

For this the most commonly used loadbalancer seems to be MetaLB, we will implement this in layer 2 mode, as BGP mode is outside the scope of this project.

### What problem are we solving

When we want to access a service from outside a node one approach could be a node port, however apart from security concerns this would not allow us to use a purely external IP only an IP of one of our nodes. 

A load balancer resolves this, we can use a single external IP to access a service across nodes, this IP will be also be unique for each service making it more practical if running multiple service. We have also the added benefit of a stable IP, and unlike a node port IP if the node changes or is removed we would not lose connectivity to the service.

### Components

Main components

### Installation and Configuration

We decided to go with the native mode as the documentation advises it is more suitable for L2 mode
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
```

Once deployed we can view the the running components within the newly created metallb-system namespace:

```
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-64bccf9875-5rkbx   1/1     Running   0          24m
pod/speaker-5ppx6                 1/1     Running   0          24m
pod/speaker-98phc                 1/1     Running   0          24m

NAME                              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/metallb-webhook-service   ClusterIP   10.0.0.205   <none>        443/TCP   24m

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   2         2         2       2            2           kubernetes.io/os=linux   24m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           24m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-64bccf9875   1         1         1       24m
```

Next we need to configure the pool that our loadbalancer can take IPs from to configure an externalIP for the service.

This is done in the IPAddressPool.yaml

However we received an error related to the webhook service:
``` 
root@lima-jumpbox:~/k8build-part-2/metallb# kubectl apply -f IPAddressPool.yaml
Error from server (InternalError): error when creating "IPAddressPool.yaml": Internal error occurred: failed calling webhook "ipaddresspoolvalidationwebhook.metallb.io": failed to call webhook: Post "https://metallb-webhook-service.metallb-system.svc:443/validate-metallb-io-v1beta1-ipaddresspool?timeout=10s": context deadline exceeded
```

As a test I ran a curl command to the metallb-webhook-service
```
curl -vk --max-time 5 https://10.0.0.205:443/
```

Testing successfully on both of my nodes but interestingly it does not work from the control plane server
```
*   Trying 10.0.0.205:443...
* Connection timed out after 5002 milliseconds
* Closing connection 0
curl: (28) Connection timed out after 5002 milliseconds
```

After some research it turns out that typically we would have kube-proxy running on the control plane server. This would give the server access to the service ip range. Up until now this access was not needed, however calling webhooks withing the cluster does in fact need it.
Our resolution could be to configure kube-proxy on our control plane server, this would probably be the case in production systems.
Alternatively we will manually add the route from our control plane server to one of our nodes. As this is project for learning, this solution is acceptable.

To add the route server to node-o
```
ip route add 10.0.0.0/24 via 192.168.105.6
````
Rerun the curl
```
root@server:~# curl -vk --max-time 5 https://10.0.0.205:443/
*   Trying 10.0.0.205:443...
* Connected to 10.0.0.205 (10.0.0.205) port 443 (#0)
```

lets re run the ipaddresspool configuration
´´´
root@lima-jumpbox:~/k8build-part-2/metallb# kubectl apply -f IPAddressPool.yaml
ipaddresspool.metallb.io/first-pool created
```

Next we can run the L2Advertisement.yaml as per the documentation
```
root@lima-jumpbox:~/k8build-part-2/metallb# kubectl apply -f L2Advertisement.yaml
l2advertisement.metallb.io/example created
```

### Testing

Using the nginxlb.yaml to create an nginx server to target and the loadbalancerService.yaml to create the loadbalancer service.

We can can see the loadbalancer service has now been configured, and has an external IP within the range of the IPpool we configured in the IPAddressPool.yaml
```
root@lima-jumpbox:~/k8build-part-2/metallb# kubectl get -o wide service nginx-loadbalancer-service
NAME                         TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)        AGE     SELECTOR
nginx-loadbalancer-service   LoadBalancer   10.0.0.237   192.168.105.50   80:30903/TCP   6h14m   app=nginx-loadbalanced
```
We can now speak with the nginx server we just created via the external IP

```
root@lima-jumpbox:~/k8build-part-2/metallb# curl 192.168.105.50
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

