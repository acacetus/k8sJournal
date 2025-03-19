# k8sJournal
Bare Metal 2 Node Cluster with MetalLB and basic monitoring

Start with the standard deployment, including prereqs (install, fw conf, etc):
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
Most guides provide initial sudo swapoff -a command.
Ensure you make the process endure through reboots verify swap is off:
sudo vi /etc/fstab
-comment out swap line
If swap is enabled kubelet will not start properly Init the cluster on your control node.

Start your cluster
This pod-network-cidr is the default for flannel CNI, if using a different CNI plugin adjust appropriately
kubeadm init --pod-network-cidr=10.244.0.0/16 --service-dns-domain=k8s.acacetus.local
READ THE PRINTED info on completion.
There are 2 important pieces:
1) Kube config:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
2) Cluster join string (save this for later)
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

Install your CNI plugin, in this case we are using flannel:
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

Ensure you allow required ports for flannel in your firewall:
https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md
"When using udp backend, flannel uses UDP port 8285 for sending encapsulated packets.
When using vxlan backend, kernel uses UDP port 8472 for sending encapsulated packets."
If you don't you can end up with services accessible from External IPs, that can't communicate with each other pod to pod.

Apply desired taint(s) and label(s)
I want to be able to use the control node for both the control plane as well as pod assignment
kubectl label node master-node kubernetes.io/role=worker

Adjust the taint on the control node to prevent tolerance failure for additional jobs (https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-
toleration/)

kubectl taint nodes master-node node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes master-node key1=value1:PreferNoSchedule

Configure and deploy MetalLB
Validate strictArp adjusted in kube-proxy per-reqs (https://metallb.universe.tf/installation/)
Apply metallb to assign external available IP addresses for access to services
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
Configure "external IP" address list: https://metallb.universe.tf/configuration/_advanced_ipaddresspool_configuration/
My basic ipexample.yaml:
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
name: first-pool
namespace: metallb-system
spec:
addresses:
- 192.168.0.1-192.168.0.254

kubectl apply -f ipexample.yaml

Setup L2 advertisement so you don't have to do crazy routing shenanigans manually (which is kind of the point)
Example L2 Advert yaml:
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
name: mainl2advert
namespace: metallb-system
spec:
ipAddressPools:
- first-pool
nodeSelectors:
- matchLabels:
kubernetes.io/hostname: master-node
- matchLabels:
kubernetes.io/hostname: worker-node
kubectl apply -f mlb-l2advert.yaml
Ensure you can advertise services on your control-plane node:

kubectl label node master-node node.kubernetes.io/exclude-from-external-load-balancers-
Verify you have metallb ports open on your firewall (for L2 mode like our example):

https://metallb.universe.tf/#requirements
Ports: 7946/TCP, 7946/UDP

Verify all pods are up and ready
You will be doing this often and this shows you pods, services, deployments, daemonsets, replicasets, you can also ad the -o wide flag for additional
information:
kubectl get all --all-namespaces
Go back and find your join string from the cluster init. If you didn't save it you can use: kubectl create token --print-join-command.
Join your second node to the cluster.
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
On the control node, verify the new node is present:
kubectl get nodes
Ensure the newly added node is ready for new items (pods, deployment, services, etc)
kubectl label node worker-node kubernetes.io/role=worker

Deploy prometheus for monitoring
I like this walk through as it walks through the different parts involved:
https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/ which references this git https://github.com/techiescamp/kubernetes-prometheus
The basic steps are as follows:
1) Create namespace
2) Create rbac role
3) config map for base container hat has scrape conf and alerting rules
4) create deployment (This can be adjusted for persistent storage, and is recommended for prod)
The guide noted walks through these in separate steps, but looking at them I don't see any reason you couldn't put this all into a single yaml (the metallb
master does a bunch of them all at once including ns, rbac, deployment as an example)
Get a view of your current cluster:
kubectl get all --all-namespaces
Find your prometheus service and find your external IP:

NAMESPACE NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
default service/kubernetes ClusterIP <provided> <none> 443/TCP 41h
kube-system service/kube-dns ClusterIP <provided> <none> 53/UDP,53/TCP,9153/TCP 41h
metallb-system service/metallb-webhook-service ClusterIP <provided> <none> 443/TCP 40h
prometheus service/prometheus-service LoadBalancer <provided> <THIS IP> 80:31275/TCP 18h
Connect to the IP from a device which has a route to it from a browser: http://<service_external_ip>/

Success!!!

Deploy grafana for visualization of the data in Prometheus:
https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/
create namespace
default yaml deployment (pvs, deploy, svc)
0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/2 nodes are available: 2 Preemption is not helpful for
scheduling.
need a persistent volume for the claim to consume (I have an nfs volume on a 7tb raid on the worker node)
sudo mount -t nfs 10.10.1.76:/run/media/acacetus/bigraid/prometheus_persist prom_persist
add to fstab (for permanence):
10.10.1.76:/run/media/acacetus/bigraid/prometheus_persist /home/acacetus/prom_persist nfs defaults 0 0
apiVersion: v1
kind: PersistentVolume
metadata:
name: grafana-pv-home
labels:
type: local
spec:
accessModes:
- ReadWriteOnce
capacity:
storage: 1Gi
hostPath:
path: "/home/acacetus/prom_persist/grafana"
adjusted mount path to persistent storage:
volumeMounts:
- mountPath: /home/acacetus/prom_persist/grafana
name: grafana-pv
Create prometheus rules for monitoring This will allow monitoring your pods/services/deployments/nodes, pretty much anything that has the ability (which it
seems is the vast majority of prepackaged stuff at this point.
Configuring Logging for Grafana to ingest:
Grafana inherently supports Loki/Promtail (they are part of the grafana environment). Loki is the data source application, promtail collects the
logs. Promtail can be configured with a number of listeners (log reads, syslog ingest, journald reads, among many others)

This is recommended at the Node level, and to prevent permissions issues to read logs (they require sudo perms min to read) I opted to configure syslog
to just double up the write to a different port configured as a promtail listener.
[acacetus@master-node ~]$ cat /etc/promtail/config.yml
# This minimal config scrape only single log file.
# Primarily used in rpm/deb packaging where promtail service can be started during system init process.
# And too much scraping during init process can overload the complete system.
# https://github.com/grafana/loki/issues/11398
server:
http_listen_port: 9080
grpc_listen_port: 9095
positions:
filename: /tmp/positions.yaml
clients:
- url: http://localhost:3100/loki/api/v1/push
scrape_configs:
- job_name: syslog
syslog:
listen_address: 127.0.0.1:5140
relabel_configs:
- source_labels: [__syslog_message_severity]
target_label: severity
- source_labels: [__syslog_message_facility]
target_label: facility
- source_labels: [__syslog_message_hostname]
target_label: host
- source_labels: [__syslog_message_app_name]
target_label: app
Make sure there were no errors for promtail startup, resolve as necessary
Now we need to configure rsyslog on the node to forward to promtail. Create a file /etc/rsyslog.d/25-promtail.conf with the following contents:
*.* action(type="omfwd" protocol="tcp"
target="127.0.0.1" port="5140"
Template="RSYSLOG_SyslogProtocol23Format"
TCP_Framing="octet-counted")
systemctl restart rsyslog
journalctl -eu rsyslog
Initially, I was running into issues I thought were fw related again but there were no block messages. After digging through the logs I was running into a
permissions issue, but it was different than previous (example from a search):
setroubleshoot[31103]: SELinux is preventing /usr/sbin/rsyslogd from name_connect access on the tcp_socket port 10601.#012#012***** Plugin
connect_ports (92.2 confidence) suggests *********************#012#012If you want to allow /usr/sbin/rsyslogd to connect to network port 10601#012Then
you need to modify the port type.#012Do#012# semanage port -a -t PORT_TYPE -p tcp 10601#012 where PORT_TYPE is one of the following:
dns_port_t, dnssec_port_t, http_port_t, kerberos_port_t, mysqld_port_t, ocsp_port_t, postgresql_port_t, rsh_port_t, syslog_tls_port_t, syslogd_port_t,
wap_wsp_port_t.#012#012***** Plugin catchall_boolean (7.83 confidence) suggests ******************#012#012If you want to
Pretty straight forward that I would need to adjust SELinux to allow the new syslog port (I obviously was using 5140, so I adjusted accordingly):
semanage port -a -t syslogd_port_t -p tcp 10601
Now we have the data source hooked up to Grafana at the node IP and port 3100 (loki) and 9080 (promtail), just remember to include these in your fw
config since we are polling the node
And Grafana is ingesting logs (Yay!) Duplicate the process on the worker node
Dec 17 10:29:06 worker-node loki[619617]: failed parsing config: /etc/loki/config.yml: yaml: unmarshal errors: Dec 17 10:29:06 worker-node loki
[619617]: line 41: field enabled not found in type aggregation.Config
>>> There was a bad field that appeared to be still present in the raw config (bonus nested enabled flag)
pattern_ingester:
enabled: true
metric_aggregation:
enabled: true <<<<
loki_address: localhost:3100
Instead of:
pattern_ingester:
enabled: true
metric_aggregation:
loki_address: localhost:3100

We have node logs, now we need Pod logs:

Due to the fact that I am doing this both for an eventual home lab as well as a learning tool, I am
going to use a different (also native module) for grafana called Alloy. Additionally, the
recommendation for using Helm for the install process I haven't run into yet.

Install helm on the control node
https://helm.sh/docs/intro/install/ (this is a raw script, so as anything being run on your device
you should be sure you trust it):
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh

Install into kubernetes:
https://grafana.com/docs/alloy/latest/set-up/install/kubernetes/
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
create alloy ns:
kubectl create namespace alloy
helm install --namespace alloy alloy grafana/alloy
verify pods are up and running: kubectl get pods -n alloy
Configure the values file (I'm going to use the default for now for the sake of the order):
wget https://raw.githubusercontent.com/grafana/alloy/main/operations/helm/charts/alloy/values.
yaml
helm upgrade -n alloy alloy grafana/alloy -f values.yaml
Add logging agent(s), here is the default:
alloy:
configMap:
content: |-
// Write your Alloy config here:
logging { <<<<
level = "info" <<<<
format = "logfmt" <<<<
} <<<<
Push the change:
helm upgrade -n alloy alloy grafana/alloy -f values.yaml

Enable pod logging, we are opting for using the control node local loki instance:
https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/
Please note if you aren't using a separate configMap file, the comments will cause errors.

discovery.kubernetes "pod" {
role = "pod"
}
discovery.relabel "pod_logs" {
targets = discovery.kubernetes.pod.targets
rule {
source_labels = ["__meta_kubernetes_namespace"]
action = "replace"
target_label = "namespace"
}
rule {
source_labels = ["__meta_kubernetes_pod_name"]
action = "replace"
target_label = "pod"
}
rule {
source_labels = ["__meta_kubernetes_pod_container_name"]
action = "replace"
target_label = "container"
}
rule {
source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
action = "replace"
target_label = "app"
}
rule {
source_labels = ["__meta_kubernetes_namespace",
"__meta_kubernetes_pod_container_name"]
action = "replace"
target_label = "job"
separator = "/"
replacement = "$1"
}
rule {
source_labels = ["__meta_kubernetes_pod_uid",
"__meta_kubernetes_pod_container_name"]
action = "replace"
target_label = "__path__"
separator = "/"
replacement = "/var/log/pods/*$1/*.log"
}
rule {
source_labels = ["__meta_kubernetes_pod_container_id"]
action = "replace"
target_label = "container_runtime"
regex = "^(\\S+):\\/\\/.+$"
replacement = "$1"
}
}
loki.source.kubernetes "pod_logs" {

targets = discovery.relabel.pod_logs.output
forward_to = [loki.process.pod_logs.receiver]
}
loki.process "pod_logs" {
stage.static_labels {
values = {
cluster = "kubernetes",
}
}
forward_to = [loki.write.default.receiver]
}
loki.write "default" {
endpoint {
url = "http://10.10.1.225:3100/loki/api/v1/push"
}
}
These logs will be under your control node datasource, but they will be labeled per pod type so
you can delineate

Troubleshooting:
During my initial conf I got a bit ahead of myself and didn't understand the requirements for flannel fully and dug myself into a hole by getting inventive with
my address spacing.
Here are some examples of the errors I saw as a result (kubectl describe pod -n <namespace> <pod_name> and/or kubectl logs -n <namespace>
<pod_name>):
Failed calling webhook, failing closed validate.nginx.ingress.kubernetes.io: failed calling webhook "validate.nginx.ingress.kubernetes.io": failed to call
webhook: Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1/ingresses?timeout=10s": dial tcp 172.2.0.57:443: connect:
connection refused
Error from server (InternalError): error when creating "ipaddresspool.yaml": Internal error occurred: failed calling webhook "ipaddresspoolvalidationwebhook
.metallb.io": failed to call webhook: Post "https://metallb-webhook-service.metallb-system.svc:443/validate-metallb-io-v1beta1-ipaddresspool?timeout=10s":
dial tcp 10.110.237.241:443: connect: connection refused Warning FailedCreatePodSandBox 53s
kubelet Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox
"956f971e137050742e59acaa4164405ee563f801ba7a6b15c5b71bbc890e900d": plugin type="flannel" failed (add): failed to set bridge addr: "cni0" already
has an IP address different from 10.244.1.1/24
Note all of these are network related.
The first you can see the IP attempting to be connected to is outside of the pod-cidr (and service cidr, but that is bonus info you can look up if interested)
The second one does appear to have a reasonable service IP (default for kubernetes is 10.96.0.0/12 from kubeadm init), but that turned to be inline with
the cause for the 3rd as well, and had the only good symptom in the 1st.
My 2nd node had some leftover configuration left over from my early IP addressing shenanigans and the cni0 bridge was still in the 172.x.x.x range
I also had an issue with the firewall breaking functionality as my nodes didn't have several ports open:
Controlled By: DaemonSet/speaker
Containers:
speaker:
Container ID:
Image: quay.io/metallb/speaker:v0.14.8
Image ID:
Ports: 7472/TCP, 7946/TCP, 7946/UDP
Host Ports: 7472/TCP, 7946/TCP, 7946/UDP

Permissions:
[acacetus@master-node ~]$ kubectl logs grafana-69d855495d-nk58j -n grafana
GF_PATHS_DATA='/var/lib/grafana' is not writable.
You may have issues with file permissions, more information here: http://docs.grafana.org/installation/docker/#migrate-to-v51-or-later
mkdir: can't create directory '/var/lib/grafana/plugins': Permission denied
sudo mkdir /var/lib/grafana
sudo chown $(id -u):$(id -g) /var/lib/grafana
adjust perms to the kubectl user (default 1000)
securityContext:
fsGroup: 1000
runAsUser: 1000
still no joy, not 100 why the dir is properly chowned why it was tripping up, but I had the pv anyway so I switched over to that for my path as seen in the
guide
PV ends up in released state after removing previous attempts at grafana deployment pv needs to be deleted and restarted kubectl delete pv <name> OR
kubectl delete -f example-pv.yaml (where example-pv.yaml is your persistent volume yaml)
logger=tsdb.prometheus endpoint=checkHealth pluginId=prometheus dsName=prometheus dsUID=ee4u7pbiyy0aod uname=admin t=2024-11-24T16:15:
03.789273996Z level=warn msg="Failed to get prometheus buildinfo" err="error querying resource: Get \"http://prometheus-service.prometheus/api/v1
/status/buildinfo\": dial tcp 10.109.128.79:80: i/o timeout"
logger=tsdb.prometheus endpoint=checkHealth pluginId=prometheus dsName=prometheus dsUID=ee4u7pbiyy0aod uname=admin t=2024-11-24T16:15:
03.789310732Z level=warn msg="Failed to get prometheus heuristics" err="failed to get buildinfo: error querying resource: Get \"http://prometheus-service.
prometheus/api/v1/status/buildinfo\": dial tcp 10.109.128.79:80: i/o timeout"
logger=context userId=1 orgId=1 uname=admin t=2024-11-24T16:15:03.789369817Z level=info msg="Request Completed" method=GET path=/api
/datasources/uid/ee4u7pbiyy0aod/health status=400 remote_addr=10.244.1.1 time_ms=20003 duration=20.003739593s size=211 referer=http://10.
10.10.2:3000/connections/datasources/edit/ee4u7pbiyy0aod handler=/api/datasources/uid/:uid/health status_source=server
This was a result of not having the necessary udp ports open for flannel to send tunneled packets between nodes.
It resulted in the worker node not being able to contact the DNS server and two service pods (prometheus and grafana) not being able to talk to each
other:
https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md
"When using udp backend, flannel uses UDP port 8285 for sending encapsulated packets. When using vxlan backend, kernel uses UDP port 8472 for
sending encapsulated packets."
Network troubleshooting:
https://github.com/nicolaka/netshoot
I would suggest adding the plugin as it makes life WAY easier spinning up sidecar debug pods

OMG FIREWALL ISSUES
-verify functionality of service via ephemeral pod:
https://github.com/nicolaka/netshoot
use the cluster IP and the appropriate service port (in this case it was prometheus and 9090)
-check host logs for firewall blocks /var/log/messages
-I had to turn on fw logging for firewalld, it was enabled by default for ufw (I used one on each node to become familar with them) -found that there were
blocks recorded in both.
-disabled the fw on the node where traffic would be ingressing (the arp was reporting to my master-node)
-functionality was still not as expected -disabled fw on the peer node
-functionality was restored
-given the nature of the internal k8s traffic I adjusted firewalld to put the cni and flannel interfaces in the trusted zone (note that this isn't optimum, but I
don't have any route to these except through the node physical interface on my network)
-started both fw's back up and functionality was present
-obviously this could likely be tuned more specifically, but for my purposes I will likely leave this as is for the time being. I suspect cilium would be
functional for this purpose
