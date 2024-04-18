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

Commands were taken from here: [Prometheus ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/prometheus)

This part is optional. I needed to do this because I am using OpenStack to deploy my Kubernetes Nodes.
<details>
  <summary><i>Cinder CSI Plugin</i></summary>


```bash


```

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

### 2.1) 

