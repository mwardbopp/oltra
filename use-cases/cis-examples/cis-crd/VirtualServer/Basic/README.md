# Virtual Server Examples

In this section we provide 3 Virtual Server deployment examples. The first two examples are simple VirtualServer deployments whereas the third provides an example of Path based routing.

- [HTTP Virtual Server without Host parameter](#http-virtual-server-without-host-parameter)
- [HTTP Virtual Server with Host parameter and a single service](#http-virtual-server-with-host-parameter-and-a-single-service)
- [HTTP Virtual Server with two services (Path Based Routing)](#http-virtual-server-with-two-services-path-based-routing)
<br><br>

> *To run the demos, use the terminal on VS Code. VS Code is under the `bigip-01` on the `Access` drop-down menu. Click <a href="https://raw.githubusercontent.com/F5EMEA/oltra/main/vscode.png"> here </a> to see how.*

## HTTP Virtual Server without Host parameter.

This section demonstrates the deployment of a Basic Virtual Server without Host parameter and a single service as the pool. The virtual server should send traffic for all Hostnames and Paths to the same pool.

Eg: noHost.yml
```yml
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: noHost
  labels:
    f5cr: "true"
spec:
  virtualServerAddress: "10.1.10.54"
  pools:
  - path: /
    service: echo-svc
    servicePort: 80
```

Change the working directory to `basic`.
```
cd ~/oltra/use-cases/cis-examples/cis-crd/VirtualServer/Basic
```

Create the Application deployment and service: 
```
kubectl apply -f ~/oltra/setup/apps/my-echo.yml
```

Create the VirtualServer resource. 
```
kubectl apply -f noHost.yml
```

> **Note:** CIS will create a Virtual Server on BIG-IP with VIP "10.1.10.54" and attaches a policy which forwards all traffic to service echo-svc.   

Confirm that the VirtualServer resource is deployed correctly. You should see `Ok` under the Status column for the VirtualServer that was just deployed.
```
kubectl get f5-vs nohost-vs 
```

Access the service as per the examples below. 
```
curl http://10.1.10.54 
curl http://10.1.10.54/test.php
curl http://test.f5demo.local --resolve test.f5demo.local:80:10.1.10.54
```

In all cases you should be able to access the service running in K8s. The output should be similar to:
```json
{
    "Server Name": "test.f5demo.local",
    "Server Address": "10.244.140.93",
    "Server Port": "80",
    "Request Method": "GET",
    "Request URI": "/",
    "Query String": "",
    "Headers": [{"host":"test.f5demo.local","user-agent":"curl\/7.58.0","accept":"*\/*"}],
    "Remote Address": "10.1.20.5",
    "Remote Port": "39478",
    "Timestamp": "1657610216",
    "Data": "0"
}
```

***Clean up the environment (Optional)***
```
kubectl delete -f noHost.yml
```

## HTTP Virtual Server with Host parameter and a single service.

This section demonstrates the deployment of a Virtual Server with a single service as the pool.
The virtual server should send traffic for all paths to the same pool.

Eg: virtual-single-pool.yml
```yml
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: single-pool-vs
  labels:
    f5cr: "true"
spec:
  host: app1.f5demo.local
  virtualServerAddress: "10.1.10.55"
  pools:
  - path: /
    service: echo-svc
    servicePort: 80
```

Change the working directory to `basic`.
```
cd ~/oltra/use-cases/cis-examples/cis-crd/VirtualServer/Basic
```

Create the Application deployment and service: 
```
kubectl apply -f ~/oltra/setup/apps/my-echo.yml
```

Create the VS CRD resource. 
```
kubectl apply -f virtual-single-pool.yml
```

> **Note:** CIS will create a Virtual Server on BIG-IP with VIP `10.1.10.55` and will attach a policy that forwards all traffic to pool echo-svc when the Host Header is equal to `app1.f5demo.local`.   

Confirm that the VirtualServer resource is deployed correctly. You should see `Ok` under the Status column for the VirtualServer that was just deployed.
```
kubectl get f5-vs single-pool-vs
```

Try accessing the service with curl as per the examples below. 
```
curl http://10.1.10.55
curl http://app5.f5demo.local --resolve app5.f5demo.local:80:10.1.10.55
```

In all the above examples you should see a reset connection as it didnt match the configured Host Header.
`curl: (56) Recv failure: Connection reset by peer`

Try again with the examples below
```
curl http://app1.f5demo.local --resolve app1.f5demo.local:80:10.1.10.55
curl http://app1.f5demo.local/test --resolve app1.f5demo.local:80:10.1.10.55
```

In both cases you should be able to access the service running in K8s. The output should be similar to:
```json
{
    "Server Name": "app1.f5demo.local",
    "Server Address": "10.244.196.135",
    "Server Port": "80",
    "Request Method": "GET",
    "Request URI": "/test",
    "Query String": "",
    "Headers": [{"host":"app1.f5demo.local","user-agent":"curl\/7.58.0","accept":"*\/*"}],
    "Remote Address": "10.1.20.5",
    "Remote Port": "34724",
    "Timestamp": "1657610340",
    "Data": "0"
}
```

***Clean up the environment (Optional)***
```
kubectl delete -f virtual-single-pool.yml
```

## HTTP Virtual Server with two services (Path Based Routing).

This section demonstrates the deployment of a Virtual Server with 2 services as the pools. This is a typical Path Based Forwarding example
The virtual server should send traffic to the corresponding K8s service, according to the URI Path. In the following example traffic with URI Path `/lib` will be forwarded to service `app1-svc` while traffic with URI Path `/portal` will be forwarded to `app2-svc`. Traffic on all other URI Paths will be dropped. 

Eg: virtual-two-pools.yml

```yml
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: two-pools-vs
  labels:
    f5cr: "true"
spec:
  virtualServerAddress: "10.1.10.56"
  host: pools.f5demo.local
  pools:
  - path: /svc1
    service: app1-svc
    servicePort: 8080
  - path: /svc2
    service: app2-svc
    servicePort: 8080    
```

Change the working directory to `basic`.
```
cd ~/oltra/use-cases/cis-examples/cis-crd/VirtualServer/Basic
```

Create the Application deployment and service: 
```
kubectl apply -f ~/oltra/setup/apps/apps.yml
```

Create the VirtualServer resource. 
```
kubectl apply -f virtual-two-pools.yml
```
> **Note:** CIS will create a Virtual Server on BIG-IP with VIP `10.1.10.56` and will attach a policy that forwards traffic to service app1-svc or app2-svc based on the URI path.   

Confirm that the VirtualServer resource is deployed correctly. You should see `Ok` under the Status column for the VirtualServer that was just deployed.
```
kubectl get f5-vs two-pools-vs
```

Try accessing the service with curl as per the examples below. 
```
curl http://pools.f5demo.local/ --resolve pools.f5demo.local:80:10.1.10.56

```
In the above example you should see a reset connection as it didnt match the configured URI Path.
`curl: (56) Recv failure: Connection reset by peer`

Try again with the examples below
```
curl http://pools.f5demo.local/svc1 --resolve pools.f5demo.local:80:10.1.10.56
curl http://pools.f5demo.local/svc2 --resolve pools.f5demo.local:80:10.1.10.56
```

Verify that the traffic was forwarded to the right service depending on the path that was entered. The output should be similar to:
```
Server address: 10.244.140.116:8080
Server name: app2-78c95bccb5-jvfnr
Date: 12/Jul/2022:07:21:49 +0000
URI: /svc2                                    <======== URI Path
Request ID: a5b08e8249b65a11aaaacd307feeca8e  
```

***Clean up the environment (Optional)***
```
kubectl delete -f virtual-two-pools.yml
```