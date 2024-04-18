# Prometheus + Grafana Monitoring Tools on Kubernetes
## 1) Install Heml
We will use Helm because it will make things easier.

```bash
sudo apt-get gpg curl -y
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

Verify Helm installation
```bash
 helm -version
```

Output:
```bash
version.BuildInfo{Version:"v3.14.2", GitCommit:"c309b6f0ff63856811846ce18f3bdc93d2b4d54b", GitTreeState:"clean", GoVersion:"go1.21.7"}
```

## 2) Install Prometheus using Helm

### 2.1) 
Commands were taken from here: [Prometheus ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/prometheus)


<details>
  <summary><i>Cinder CSI Plugin</i></summary>
 
_This part is optional. I needed to do this because I am using OpenStack to deploy my Kubernetes Nodes._

Git clone repository for `cinder-csi-plugin`. Follow the guide here [cinder-csi-plugin](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md)
```bash
git clone https://github.com/kubernetes/cloud-provider-openstack.git
```

Change to directory:
```bash
cd /cloud-provider-openstack/manifests/cinder-csi-plugin
```

Create file `cloud.conf`
```bash
[Global]
username = YOUR_USER
password = YOUR_PASSWORD
domain-name = default
auth-url = https://YOUR_DU_URL/keystone/v3
tenant-id = YOUR_TENANT_ID
region = YOUR_REGION
```

Encode file `cloud.conf` to Base64
```bash
cat cloud.conf | base64 |tr -d '\n'

Output:
W0dsb2JhbF0Kddfgkjhdkfjhg1pbgpwYXNzd29yZCA9IGV3UkRManJuWWJYZlBjNkxhV1RubzFyc1FxRlZzdFZuekFobFRodWYKZG9tYWluLW5hbWUgPSBEZWZhdWx0CmF1dGgtdXJsID0gaHR0cDovLzE5Mi4xNjguMi44OD.....vbk9uZQo=
```

Copy the contents of the encoded OpenStack RC configuration and add the string into the data field of the `csi-secret-cinderplugin.yaml` file
```bash
# This YAML file contains secret objects,
# which are necessary to run csi cinder plugin.

kind: Secret
apiVersion: v1
metadata:
  name: cloud-config
  namespace: kube-system
data:
  cloud.conf: W0dsb2JhbF0KdXNlcm5hbWUgP        <=== Change this!
```

Kubectl apply everything in the directory
```bash
kubectl apply -f .
```
Verify
```bash
kubectl get pods -n kube-system

Output:
NAME                                           READY   STATUS    RESTARTS      AGE
coredns-85b955d87b-585nd                       1/1     Running   1 (43m ago)   46m
coredns-85b955d87b-w4bc2                       1/1     Running   1 (43m ago)   46m
csi-cinder-controllerplugin-646bdf9885-fvfgz   6/6     Running   1 (46s ago)   107s              <=== This
csi-cinder-nodeplugin-hptwq                    3/3     Running   0             107s              <=== This
csi-cinder-nodeplugin-jqhrg                    3/3     Running   1 (67s ago)   107s              <=== This
csi-cinder-nodeplugin-z67px                    3/3     Running   1 (68s ago)   107s              <=== This
kube-apiserver-control-1                       1/1     Running   0             19m
kube-controller-manager-control-1              1/1     Running   2 (19m ago)   19m
kube-flannel-96wjz                             1/1     Running   0             18m
kube-flannel-gkzwm                             1/1     Running   1 (43m ago)   45m
kube-flannel-mkctv                             1/1     Running   1 (43m ago)   45m
kube-proxy-8d7ct                               1/1     Running   0             43m
kube-proxy-dtlw4                               1/1     Running   1 (43m ago)   45m
kube-proxy-vmbfb                               1/1     Running   0             18m
kube-scheduler-control-1                       1/1     Running   2 (19m ago)   19m
```

### _Dont forget to check persistent volume claim and StorageClass when you run into Prometheus volume errors_

<hr/>
<hr/>
</details>

Add Repo
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Output:
```bash
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Helm Install. We will create them in a different namespace for easy management. `--namespace monitoring --create-namespace`
```bash
helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace
```

If installation was successful, you will get the following

Output:
```bash
...
Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus-pushgateway,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```

### 2.2) Let's expose the service `prometheus-server` to a NodePort Service so we can access it in the web
Check all SVC
```bash
kubectl get svc -n monitoring

Output:
service/kubernetes                            ClusterIP   10.96.0.1      <none>        443/TCP        23m
service/prometheus-alertmanager               ClusterIP   10.111.162.3   <none>        9093/TCP       9m49s
service/prometheus-alertmanager-headless      ClusterIP   None           <none>        9093/TCP       9m49s
service/prometheus-kube-state-metrics         ClusterIP   10.103.2.1     <none>        8080/TCP       9m49s
service/prometheus-prometheus-node-exporter   ClusterIP   10.97.16.97    <none>        9100/TCP       9m49s
service/prometheus-prometheus-pushgateway     ClusterIP   10.102.82.45   <none>        9091/TCP       9m49s
service/prometheus-server                     ClusterIP   10.96.2.25     <none>        80/TCP         9m49s
```

We expose the target port `9090`

```bash
kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-ext
```
We now have a newly added service

```bash
kubectl get svc -n monitoring

...
...
...
service/prometheus-server                     ClusterIP   10.96.2.25     <none>        80/TCP         9m49s
service/prometheus-server-ext                 NodePort    10.99.0.210    <none>        80:30136/TCP   3s                  <== Verify this
```

We enter http://<Node IP>:<PORT>. For me it is  `http://192.168.2.242:30136/`

<p align="left">
  <img width="80%" height="80%" src="https://github.com/famasboy888/Prometheus_Grafana_kubernetes/assets/23441168/3004d59f-4852-42ed-8842-aadfcdc718eb">
</p>

## 3) Install Grafana using Helm

### 3.1) Add Grafana Repo

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
Output:
```bash
...Successfully got an update from the "grafana" chart repository
Update Complete. ⎈Happy Helming!⎈
```
