# calico-notes-exam
Certified Calico Operator Exam Notes



Operator mode is recommened for installation.


calico-node: Calico-node runs on every Kubernetes cluster node as a DaemonSet. It is responsible for enforcing network policy, setting up routes on the nodes, plus managing any virtual interfaces for IPIP, VXLAN, or WireGuard.

calico-typha: Typha is as a stateful proxy for the Kubernetes API server. It's used by every calico-node pod to query and watch Kubernetes resources without putting excessive load on the Kubernetes API server.  The Tigera Operator automatically scales the number of Typha instances as the cluster size grows.

calico-kube-controllers: Runs a variety of Calico specific controllers that automate synchronization of resources. For example, when a Kubernetes node is deleted, it tidies up any IP addresses or other Calico resources associated with the node.


Calico network policies are more advanced than Kubernetes Net Pol.\
They can add selectors like service accounts, setup order / precedence for rules and many more.



By default Kubernetes allows all. Hence add defaut Deny can be added on namespace basis.\
Calico globalnetwork policy can be used to achive the same on Cluster level rather than namespace levels.


Be careful as this may break the entire cluster.
USe namespace selectors and other egress rules for pod etc like kubedns, calico ns objects etc.


Sample file

            cat <<EOF | calicoctl apply -f -
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: default-app-policy
            spec:
              namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system"}
              types:
              - Ingress
              - Egress
              egress:
                - action: Allow
                  protocol: UDP
                  destination:
                    selector: k8s-app == "kube-dns"
                    ports:
                      - 53
            EOF



We'll create a Calico GlobalNetworkPolicy to restrict egress to the Internet to only pods that have a ServiceAccount that is labeled "internet-egress = allowed".

Examine the policy below. While Kubernetes network policies only have Allow rules, Calico network policies also support Deny rules.\
As this policy has Deny rules in it, it is important that we set its precedence higher than the lazy developer's Allow rules in their Kubernetes policy.\
To do this we specify order value of 600 in this policy, which gives this higher precedence than Kubernetes Network Policy\
(which does not have the concept of setting policy precedence, and is assigned a fixed order value of 1000 by Calico - i.e, policy order 600 gets precedence over policy order 1000).


                                    cat <<EOF | calicoctl apply -f -
                                    apiVersion: projectcalico.org/v3
                                    kind: GlobalNetworkPolicy
                                    metadata:
                                      name: egress-lockdown
                                    spec:
                                      order: 600
                                      namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system"}
                                      serviceAccountSelector: internet-egress not in {"allowed"}
                                      types:
                                      - Egress
                                      egress:
                                        - action: Deny
                                          destination:
                                            notNets:
                                              - 10.0.0.0/8
                                              - 172.16.0.0/12
                                              - 192.168.0.0/16
                                              - 198.18.0.0/15
                                    EOF



Manging trust across different teams.

The secops team is responsible for creating Namespaces and Services accounts for dev teams. They are also responsible for writing Calico Global Network Policies to define the cluster’s overall security posture. Kubernetes RBAC is set up so that only they can do this.

Dev teams are given Kubernetes RBAC permissions to create pods and Kubernetes Network Policies in their Namespaces, and they can use, but not modify any Service Account in their Namespaces. 

In this scenario, the secops team can control which teams should be allowed to have pods that access the internet. If a dev team is allowed to have pods that access the internet then the dev team can choose which of their pods access the internet by using the appropriate Service Account.




Hostendpoints\
Thus far, we've created policies that protect pods in Kubernetes.\
However, Calico Policy can also be used to protect the host interfaces in any standalone Linux node (such as a baremetal node, cloud instance or virtual machine) outside the cluster.


 Host endpoints are non-namespaced. So in order to secure host endpoints we'll need to use Calico global network policies. In a similar fashion to how we created the default-app-policy for pods in the previous module which allowed DNS but default denied all other traffic, we’ll create a default-node-policy that allows processes running in the host network namespace to connect to each other, but results in default-deny behavior for any other node connections.
 

             cat <<EOF| calicoctl apply -f -
            ---
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: default-node-policy
            spec:
              selector: has(kubernetes.io/hostname)
              ingress:
              - action: Allow
                protocol: TCP
                source:
                  nets:
                  - 127.0.0.1/32
              - action: Allow
                protocol: UDP
                source:
                  nets:
                  - 127.0.0.1/32
            EOF
 
 

failsafe\
The answer is that Calico has a configurable list of “failsafe” ports which take precedence over any policy. These failsafe ports ensure the connections required for the host networked Kubernetes and Calico control planes processes to function are always allowed (assuming your failsafe ports are correctly configured). This means you don’t have to worry about defining policy rules for these. The default failsafe ports also allow SSH traffic so you can always log into your nodes.


Lock down node port access\
Kube-proxy load balances incoming connections to node ports to the pods backing the corresponding service. This process involves using DNAT (Destination Network Address Translation) to map the connection to the node port to a pod IP address and port.


We can specify who from outside can access.


Course  Week 2  Network Policy for Hosts and NodePorts  Restrict Access to Kubernetes NodePorts



Week 3:-- \
Calico makes it easy to encrypt on the wire in-cluster pod traffic in a Calico cluster using WireGuard. WireGuard utilizes state-of-the-art cryptography and aims to be faster, simpler, leaner, than alternative encryption techniques such as IPsec.


Calico handles all the configuration of WireGuard for you to provide full mesh encryption across all the nodes in your cluster.  WireGuard is included in the latest Linux kernel versions by default, and if running older Linux versions you can easily load it as a kernel module. (Note that if you have some nodes that don’t have WireGuard support, then traffic to/from those specific nodes will be unencrypted.)



kubectl cluster-info dump | grep -m 2 -E "service-cidr|cluster-cidr"



Choosing the best network \

https://docs.projectcalico.org/networking/determine-best-networking




The Summary pods use this service to connect to the Database pods. Kube-proxy uses DNAT to map the Cluster IP to the chosen backing pod.


In summary, for a packet being sent to a clusterIP:

The KUBE-SERVICES chain matches on the clusterIP and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.

The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.

The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

Finally, let's return to host1 to take a look at NodePorts in the next section by using the exit command.




Calico's eBPF dataplane is an alternative to the default standard Linux dataplane (which is iptables based). The eBPF dataplane has a number of advantages:

It scales to higher throughput.
It uses less CPU per GBit.
It has native support for Kubernetes services (without needing kube-proxy) that:
Reduces first packet latency for packets to services.
Preserves external client source IP addresses all the way to the pod.
Supports DSR (Direct Server Return) for more efficient service routing.
Uses less CPU than kube-proxy to keep the dataplane in sync.





To enable Calico eBPF we need to:

Configure Calico so it knows how to connect directly to the API server (rather than relying on kube-proxy to help it connect)
Disable kube-proxy
Configure Calico to switch to the eBPF dataplane



- The Kubernetes network model specifies that pod can communicate with each other directly without NAT. But going via a Kubernetes Service is not classed as direct, and inherently involved at least DNAT as part of load balancing the service to the backing pods.




- Node port services do not normally preserve client source IP address, because return packets need to be routed via the ingress node in order to reverse the DNAT associated with the service load balancing.


https://kubernetes.io/docs/tutorials/services/source-ip/

Packets sent to ClusterIP from within the cluster are never source NAT'd if you're running kube-proxy in iptables mode, (the default). \
Packets sent to Services with Type=NodePort are source NAT'd by default.\
Packets sent to Services with Type=LoadBalancer are source NAT'd by default, because all schedulable Kubernetes nodes in the Ready state are eligible for load-balanced traffic. So if packets arrive at a node without an endpoint, the system proxies it to a node with an endpoint, replacing the source IP on the packet with the IP of the node 
