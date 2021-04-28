# Kubectl Cheatsheet

* use multiple config files: `KUBECONFIG=~/.kube/config:~/.kube/kubconfig2 `
* change default namespace: `kubectl config set-context --current --namespace $new-namespace`
* check previous failed pod log: `kubectl logs --previous <pod_name>`
* create pod in one line: `kubectl run busybox --image=busybox --restart=Never -- tail -f /dev/null`
* create job in one line: `kubectl create job loop --image=busybox -- tail -f /dev/null`
* create cronjob in one line: `kubectl create cronjob --image=busybox --schedule="*/1 * * * *" -- echo "Hello World"`
* get the documentation for resource manifests: `kubectl explain pods`
* create secret without file:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: $(echo -n "s33msi4" | base64 -w0)
  username: $(echo -n "jane" | base64 -w0)
EOF
```
* get secret data in one line: `kubectl get secret mysecret -o go-template='{{range $k,$v := .data}}{{"### "}}{{$k}}{{"\n"}}{{$v|base64decode}}{{"\n\n"}}{{end}}'`
* check diff between current state of cluster and a file: `kubectl diff -f local.yaml`
* check kubectl plugin list: `kubectl plugin list`
* get raw data
```
kubectl get --raw /api/v1/namespaces
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
```