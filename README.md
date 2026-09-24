# K8build-part-2
Follow up project to build out existing K8s the hard way cluster adding additional features.

## CoreDNS

We will implement a DNS within our cluster to assist with Service to Service communication.

CoreDNS is an open-source solution that typically comes preconfigured in clusters deployed by Kubeadm. However
as we have manually deployed our cluster we will need to add it ourselves.

### What problem are we solving

Lets take a look at an example as that may better describe it.
We have created a clusterip service to expose an existing nginx pod internally to a linux container within our cluster. 

The test pod and service configurations:
nginx.yaml (our web server we will try to contact)
netutils.yaml (this is a test pod that contains basic network tools)
nginx-clusterip-service.yaml

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

This is the our issue, we have no way of communicating with the cluster-ip via DNS and therefore its nginx endpoint. This is problem as when we start to configure larger deployments:
- We would need to know the IP of the service, this is impractical as it assigns the IP at runtime fo the service. Making it more complex to reference the service in the same deployment file.
- DHCP leases exist meaning the IPs will change, meaning we need to manually update our configuration.

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


## Configure DNS

Looking online I managed to find an existing yaml file with the (mostly) configuration we need.

To break it down we have 

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
````

We can successfully curl our my-nginx pod via its corresponding cluster-ip service
