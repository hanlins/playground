# Compare different network solutions for kubernetes

Kubernetes container network uses plugable network solution (i.e. [CNI](https://github.com/containernetworking/cni). There are lots of open source CNI solutions, like [flannel](https://github.com/coreos/flannel) from coreos, [Calico](https://github.com/projectcalico/calico) from tigera. Each of these network solutions has its own trade-offs, and in this post we will discuss those trade-offs and see what's the use cases for these solutions.

In this post, we will cover flannel, calico and NSX's container solution. Like said previously, it will be focus on the comparasion rather than technical details. I will make tis post as self-contained as possible, in the following I will have one section that do a brief introduction to kubernetes container network, one section focused on each network solutions and appendix for the resources I found during the process.

## Introduction to Kubernetes network
In kubernetes network model, each node (host) will be allocated one IP, and each pod (that runs on nodes) will also have one IP. Notice the node network's CIDR should be seperate from pods' CIDR. Workloads in each pod should be talking with each other using pod IP without NAT rules. For CNI, it just need to implement two basic commands, ADD and DELETE (also VERSION which shows the version information of the plugin, that is trivial), which handles provision and deprovision works for container.

## Container network solutions
### Flannel: a overlay network solution with different backends
Flannel is an overlay network solution, that means pod network's L2 traffic is encapsulated on node network's L3 traffic. Each node should have an TEP (tunnel end point) which forwards pod traffic among each hosts. The traffic can be encapsulated in typically VxLAN or UDP, in which VxLAN has better performance and usually better vendor support, and UDP will have worse performance but it doesn't depend on any kernel features so better portability.
Notice that fannel focuses on overlay network, and it doesn't support network policy out of box.

### Calico: a pure L3 network solution
Calico calaimed to be focusing on scability, performance (it claimed that they can have 50,000 containers talking in the network) and easy of use. Since there's no encapsulation work, it won't involve perfocmance overhead. It's leverating some common-known tech like BGP to make routing more scalable and also it's kind of easier for user to do trouble shooting using route tracing since there's no package encapsulation.
Another thing need mention is that Calico can support network policy. To implement the micro-segmentation and network policy, it leverages iptables to setup firewalls.

## Discussions for the trade-offs
### Why we need overlay network
Overlay network involves encapsulation overhead and Calico got better performance when getting rid of that, but why we need overlay network at the first place?
1. flexibility, dynamically changing network topology
2. hide the topology details from containers, they can only see it's local L2 network
3. decouple underlay (physical) and overlay (logical) network

If your underlay and overlay network matches well, then overlay doesn't make much sense. But in modern SDDC scenarios, due to the fact that network topology is dynamic and changing rapidly, usually we will need overlay network. So from my understanding, if customer is managing kubernetes in SDDC, then overlay network will definitely help. On the other hand, if customer is managing a private network within its own company, the network topology doesn't change rapidly, then it will make more sense to simply use Calico solution.

### Does encapsulation really involves significent overhead
From a [post](https://packetpushers.net/vxlan-udp-ip-ethernet-bandwidth-overheads/), it seems we can achieve around 95% percent efficiency using overlay network if configuredlly. On the other hand, many vendors have already supportted hardware acceleration for vxlan, so the difference should not be very different.
Interestingly, a project also from tigera named [canal](https://github.com/projectcalico/canal), leverages overlay network from fannel and network policy from calico. I think tigera knows that calico is not for everyone, and they have to compromise performance (which is affordable) for more dynamic network topology.

## Resources
Overhead: https://packetpushers.net/vxlan-udp-ip-ethernet-bandwidth-overheads/
Discussions from Calico community: https://www.projectcalico.org/project-calico-needs-to-communicate-better/
