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
