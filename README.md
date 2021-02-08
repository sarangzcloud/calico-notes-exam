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
