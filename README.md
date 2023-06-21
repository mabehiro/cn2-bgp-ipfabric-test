# BGP Peering with Juniper CN2
OCP config for IP Fabric Forwarding with MikroTik RouterOS

## Intro
In this repo, I will explore the process of establishing BGP (Border Gateway Protocol) peering to make pod network and service IPs visible to an external router, specifically MikroTik Router. Additionally, I will test the external connectivity without a traditional load balancer, leveraging CN2's Fabric Forwarding feature.

![image](https://github.com/mabehiro/cn2-bgp-ipfabric-test/assets/104660935/8a884ce8-26c0-4965-9d4e-0f3548a1dbe0)

## MikroTik config

```
/interface bridge add name=lobridge
/ip address add address=10.16.0.180/24 interface=ether2
/ip address add address=10.1.1.1/32 interface=lobridge
/ip address add address=192.168.21.1/24 interface=ether3
/interface bridge port
add bridge=lobridge interface=ether2
add bridge=lobridge interface=ether3
/routing bgp connection
add address-families=ip as=64512 disabled=no local.address=10.16.0.180 role=ibgp name=bgp1 output.redistribute=connected,static remote.address=10.16.0.0/24 as=64512 router-id=10.1.1.1 routing-table=main
```
## BGPRouter
Set up a BGP router CR (Custom Resource) for an external router, which in this case, is the MikroTik router. <br>
The configuration involves specifying the external router's IP and identifier in the CR, establishing BGP peering between Contrail Control Node and the external router, and advertising routes from the Contrail Virtual Network to the external router.

Once the configuration is applied, you can check the CNS2 status.
```
$ oc get bgprouter -A
NAMESPACE   NAME        TYPE           IDENTIFIER    STATE     AGE
contrail    bgprouter   router         10.1.1.1      Success   5m34s
contrail    master1     control-node   10.16.0.249   Success   4d23h
contrail    master2     control-node   10.16.0.228   Success   4d23h
contrail    master3     control-node   10.16.0.194   Success   4d22h
```
On the router, there should be three bgp sessions.

```
[admin@MikroTik] /routing/bgp/session> print
Flags: E - established 
 0 E name="bgp1-1" 
     remote.address=10.16.0.194 .as=64512 .id=10.16.0.194 .capabilities=mp,gr .afi=ip,vpnv4 .hold-time=1m .messages=31 .bytes=1027 .eor="" 
     local.role=ibgp .address=10.16.0.180 .as=64512 .id=10.1.1.1 .capabilities=mp,rr,gr,as4 .messages=25 .bytes=514 .eor="" 
     output.procid=20 
     input.procid=20 .last-notification=ffffffffffffffffffffffffffffffff0015030603 ibgp

-- omit ---
```

## Creating a Namespace with IP Forwarding
create a namespace with IP forwarding enabled. <br>
This configuration allows the IP fabric to be applied to both the default pod network and the default service network within the namespace. I am using isolated-namespace.

Create a namespace with the appropriate label and annotation. In this example, I use the namespace name "ns1" and apply the "ip-fabric" forwarding mode with the label "core.juniper.net/isolated-namespace: true". Use the following YAML configuration:

```
apiVersion: v1
kind: Namespace
metadata:
  name: ns1
  annotations:
    core.juniper.net/forwarding-mode: "ip-fabric"
  labels:
    core.juniper.net/isolated-namespace: "true"

```

## Testing
For simplicity and testing purpose, add the anyuid security constraints to the default service account in the newly created namespace. Run the following command:

```
oc adm policy add-scc-to-user -n ns1 anyuid -z default
```
```
oc apply -f pod_svc.yaml
```
Verify the routes in the MikroTik router and ensure that the service and Pod IP addresses are exported correctly

```
[admin@MikroTik] /routing/bgp/session> /ip/route/print
Flags: D - DYNAMIC; A - ACTIVE; c, b, y - BGP-MPLS-VPN
Columns: DST-ADDRESS, GATEWAY, DISTANCE
    DST-ADDRESS       GATEWAY      DISTANCE
DAc 10.1.1.1/32       lobridge            0
DAc 10.16.0.0/24      lobridge            0
D b 10.128.0.54/32    10.16.0.155       200   
DAb 10.128.0.54/32    10.16.0.155       200
DAb 10.128.0.105/32   10.16.0.195       200
D b 10.128.0.105/32   10.16.0.195       200
D b 172.30.141.68/32  10.16.0.155       200
DAb 172.30.141.68/32  10.16.0.155       200

-- omit --
```
Send a curl request from Test VM, verifying the test VM can reach to the service through the external router.

```
root@testvm:~# curl http://172.30.141.68
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
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
