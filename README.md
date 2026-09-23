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



#### Prerequisites

### First lets confirm we can communicate across pods via IP
After creating my-nginx we exec into the pod, I will try to ping and curl the ip of my-nginx1

```
exec --stdin --tty my-nginx -- /bin/bash
root@my-nginx:/# ping 10.200.1.58
PING 10.200.1.58 (10.200.1.58) 56(84) bytes of data.
```
```
curl 10.200.1.58 
```

However I receive no response from either

I can see 2 possible reasons here:
1. The my-nginx1 is not exposed via a service
2. The my-nginx1 IP is not routable as it is on another node

#### 1. Is incorrect, as we are trying to connect directly to via the IP, as long as the port is exposed in the initial yaml we should be ok.

I can confirm this via a curl command from a pod running on the same node
```
root@my-nginx:/# curl 10.200.0.80:80
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

#### 2. This is the more likely as checking we are unable to ping any pods running on opposite clusters
```
root@node-1:~# ping 10.200.0.78 
PING 10.200.0.78 (10.200.0.78) 56(84) bytes of data.
^C
```
Lets add a route using this command and try again
```
ip route add 10.200.0.0/24 via 192.168.105.6
```
This commands will redirect anything for the 10.200.0.0 range(subnet used by node-0) via the 192.168.105.6(node-1)

Running the ping again
```
root@node-1:~# ping 10.200.0.78 
PING 10.200.0.78 (10.200.0.78) 56(84) bytes of data.
64 bytes from 10.200.0.78: icmp_seq=1 ttl=63 time=0.985 ms
64 bytes from 10.200.0.78: icmp_seq=2 ttl=63 time=0.912 ms
```

Add route to other node as well
```
ip route add 10.200.1.0/24 via 192.168.105.7
```

Also lets add both of these routes to the Controller
```
root@server:~# ip route add 10.200.0.0/24 via 192.168.105.6
root@server:~# ip route add 10.200.1.0/24 via 192.168.105.7
```

Now finally when we exec into our nginx pod we can successfully curl our nginx pod running on node-1
```
root@my-nginx:/# curl 10.200.1.59:80
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

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional 
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



## DNS components

For Inter-pod communication via name we need will need some form of DNS.

Components on the Cluster

Typically coreDNS will have the following components

DNS service
Tha actual service


DNS deployment

DNS configmap



## Configure DNS

Looking online I managed to find an existing yaml file with the (mostly) configuration we need.

To break it down we have 



If we apply the  cordns.yaml

Several objects will be created



Configure cluster-ip service range. As clusterDNS is failing




We added the range to the kube-apiserver.service file on the Control Server

root@server:~# cat /etc/systemd/system/kube-apiserver.service
  --service-cluster-ip-range=10.32.0.0/24 \


-

## Results and conclusion
Now when we re run our test at the beging we can the 