# K8build-part-2
Follow up project to build out existing K8s the hard way cluster adding additional features

## CoreDNS

We will implement a DNS within our cluster to assist with Service to Service communication.

CoreDNS is the standard used across multiple cluster types.

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

#### 1. Is incorrect, as we are trying to connect directly to via the IP, as long as the port is exposed in the initial yaml we
should be ok.
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