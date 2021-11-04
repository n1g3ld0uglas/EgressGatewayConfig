# EgressGatewayConfig

Configure local shell to connect to the cluster, e.g. ```export KUBECONFIG=/path/to/kubeconfig```.

When using a cloud environment to host K8s cluster, the worker nodes could be configured on different subnets. For Egress Gateways feature to work properly in such environment, both the Egress Gateway pod, that is selected to serve as the gateway, must run on the same worker node as the client pod (i.e. netshoot). Otherwise, this feature won't work. Note that this issue is specific to using a cloud infrastructure as a demo environment for Egress Gateways feature.

configure IPPool resource
Once cluster is setup and Calico Enterprise is installed, deploy an IPPool resource that will be dedicated to egress gateways only.

Make sure the IP range you select for this IPPool does not overlap with AWS VPC CIDR (default is 10.0.0.0/16).

```
kubectl apply -f setup/ippool-egress.yml
```

enable Egress gateways feature
Enable Egress gateways in calico-node (a.k.a. Felix).

# example to enable per namespace only
```
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"egressIPSupport":"EnabledPerNamespace"}}'
```
# example to enable per namespace or per pod
```
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"egressIPSupport":"EnabledPerNamespaceOrPerPod"}}'
```

# verify configuration
```
kubectl describe felixconfiguration.p default
```
Configure pull secret for egress gateways namespace. 
Get pull secret to pull CaliEnt images and copy it into the namespace you will create egress gateways in (e.g. egress-gw)

```
EGRESS_GW_NS='egress-gw'
kubectl create ns $EGRESS_GW_NS
kubectl get secret tigera-pull-secret --namespace=calico-system --export -o yaml | kubectl apply --namespace=$EGRESS_GW_NS -f -
```
# deploy egress gateways into your namespace
```
kubectl apply -f setup/egress-gw.yml
```
# view annotation configuration on egress gateways
```
kubectl -n $EGRESS_GW_NS get po -oyaml | grep -i 'cni.projectcalico.org/ipv4pools'
```
configure namespace or pod to use egress gateway
If you've configured Egress gateways feature on per namespace basis, then apply annotations to namespaces only. 
If you enabled the feature to be use on per pod basis too, then annotate the pods that should use egress gateways functionality.

This example labels egress gateways with egress-code: red label. 
This key-value pair is used to annotate the namespace that hosts pods configured to use egress gateways.

Deploy namespace and the client pod (netshoot) to test the feature operation.

```
kubectl apply -f app/
```
The namespace definition in 00-namespace.yaml already includes required annotations. 
However, if you choose to use a different namespace, you can label the namespace following example below:

```
CLIENT_POD_NS='ns1-poc'
```

# review annotations on client POD namespace
```
kubectl get ns $CLIENT_POD_NS -oyaml | grep -m2 -i -m1 'annotations:' -A6
```
# label namespace if required annotations are missing
# selector annotation must match label(s) configured on egress gateways deployment
```
kubectl annotate ns $CLIENT_POD_NS egress.projectcalico.org/selector='egress-code == "red"'
```
# namespaceSelector annotation is required when egress gateways pods are hosted in a different namespace than the client pods that use them
```
kubectl annotate ns $CLIENT_POD_NS egress.projectcalico.org/namespaceSelector='projectcalico.org/name == "egress-gw"'
```
verify the feature operation
If you shutdown the utility instance and then start it again later, make sure to add back the necessary route for the Egress IPPool.

The testing plan:

launch an ec2 instance that is not a part of k8s cluster (i.e. a utility instance). The easiest option is to right click on one of the existing worker nodes and use Lunch More Like This option from the popup menu to create an ec2 instance with Docker engine in it. Edit Name tag to give the instance a meaningful name. The other default options for the instance are OK.
SSH into the newly created ec2 node and launch a container on the host network with a sever that listens on a certain port, e.g. netcat server
configure the route on the ec2 instance for the Egress gateway IP(s) so that the request with the Egress Gateway IP as a source, can be properly routed back to the origin, i.e. the worker node that houses the client pod that initiated the connection
use the deployed client pod (i.e. netshoot) to connect to the netcat server
review the logs of netcat server to verify source IP for incoming connections
# SSH into a utility node
```
EC2_PUB_IP='xx.xx.xx.xx'
SSH_KEY='/path/to/ssh_key'
```
# assuming Ubuntu image is used for the host that has 'ubuntu' user
```
ssh -i $SSH_KEY -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no ubuntu@$EC2_PUB_IP
```
# if docker is not installed, install docker on the utility node
```
if [ -z $(which docker) ]; then
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  sudo apt-get update -y
  sudo apt-get install -y docker-ce docker-ce-cli containerd.io
fi
```

# route vars
# use IP of your egress gateway pod, e.g. 10.10.10.0
```
EGRESS_IP='10.10.10.0/31'
```
# get internal IP of a worker node that runs the client pod from AWS console
# or using shell that can talk to your k8s cluster
```
echo "EGRESS_WORKER_NODE_IP=$(kubectl -n ns1-poc get po -l 'app=netshoot' -o jsonpath='{.items[].status.hostIP}')"
```
# configure route for Egress Gateway IP(s)
```
sudo ip r a ${EGRESS_IP} via ${EGRESS_WORKER_NODE_IP}
```
# test the route
```
ping 10.10.10.0
```
# launch netcat server as a standalone Docker container using host network
# the shell will attach to container STDOUT; if you don't want this, add '-d' flag to 'docker run' command
```
sudo docker run --name netcat-srv --net=host --privileged subfuzion/netcat -n -v -l -k -p 8089
```
# you can use tcmpdump to capture ICMP or TCP packets on ec2 instance
```
sudo tcpdump -i any -n -v tcp port 8089
sudo tcpdump -i any -n -v icmp
```
Use netshoot client pod to connect to the netcat server container.

# get utility VM IP address that runs netcat server container
# SSH into the VM and get its internal IP or use AWS console to get the IP
```
echo "UTILITY_PVT_IP=$(hostname -I | awk '{print $1}')"
```
# initiate connection from netshoot pod to the standalone netcat server container
```
kubectl -n ns1-poc exec -t netshoot -- bash -c "nc -zv $UTILITY_PVT_IP 8089"
```
# test using ICMP
```
kubectl -n ns1-poc exec -t netshoot -- bash -c "ping -c3 $UTILITY_PVT_IP"
```

# Troubleshooting Egress Gateway
When using AWS infrastructure for K8s cluster and seeing the egress traffic not using the Egress Gateway IP, make sure that the Egress Gateway pod, that was selected as the gateway, runs on the worker node that is on the same subnet as the node where the client pod runs, i.e. netshoot. For the guaranteed result, you can configure the Egress Gateway pod to run on the same worker node as the client pod.

Check IP rules and route table on the host that runs the client pod(s). <br/>
If you see a rule with your pod's IP, but no routes in the table, then review and make sure that annotations configuration for the client pod and/or its namespace match the egress gateways namespace and labels.
```
# SSH into the node that runs your client pod(s)

# check ip rules to get route table name (could be a word or a number)
ip rule
TABLE='xxx' # set table name
# view routes in the table with your pod IP
ip route list table $TABLE
```

Egress Gateway configures ```egress.calico``` device that it uses to establish VXLAN tunnel between EG pods and the client pods.
```
ip link | grep -i egress
```

when EG is configured, ```calico-node``` would have some log entries similar to these:

```
2021-05-26 17:47:06.676 [INFO][61] felix/egress_ip_mgr.go 619: egress ip VXLAN tunnel device configured
2021-05-26 17:47:16.677 [INFO][61] felix/egress_ip_mgr.go 619: egress ip VXLAN tunnel device configured
2021-05-26 17:47:18.342 [INFO][61] felix/route_table.go 445: Queueing a resync of routing table. ifaceRegex="^egress.calico$" ipVersion=0x4
2021-05-26 17:47:18.342 [INFO][61] felix/route_table.go 445: Queueing a resync of routing table. ifaceRegex="^egress.calico$" ipVersion=0x4
2021-05-26 17:47:18.344 [INFO][61] felix/route_table.go 981: Syncing interface L2 routes ifaceName="egress.calico" ifaceRegex="^egress.calico$" ipVersion=0x4
```

## Example outputs from Ubuntu worker
ip rules

```
$ ip rule

0:      from all lookup local
100:    from 192.168.36.214 fwmark 0x80000/0x80000 lookup 250
32766:  from all lookup main
32767:  from all lookup default
```
table routes when egress gateway has 2 pods

```
$ ip route list table 250

default onlink
  nexthop via 10.10.10.0 dev egress.calico weight 1 onlink
  nexthop via 10.10.10.1 dev egress.calico weight 1 onlink
```
ip rules if both, client pod and egress gateway pod are housed by the same worker node

```
$ ip rule
0:      from all lookup local
100:    from 10.10.10.0 fwmark 0x80000/0x80000 lookup 250
100:    from 192.168.36.214 fwmark 0x80000/0x80000 lookup 250
32766:  from all lookup main
32767:  from all lookup default
```
