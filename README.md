# k8s-init
## k8sのinitをするにあたっての下準備をansibleで極力自動化したのでメモ
- ```/etc/netplan/99-config.yaml```を書き込んで```netplan apply```
- ```ssh-keygen -f ~/.ssh/id_rsa_k8s```で鍵作成
  - パスフレーズは何も入力せずEnter  
- ```ssh-copy-id -i ~/.ssh/id_rsa_k8s.pub [リモートユーザー]@[リモートサーバーのホスト名]```で公開鍵転送
- ```.ssh/config```を書き換え
- ```ansible-playbook kubeinit.yaml --ask-become-pass```を実行

### setup(master)
```
sudo kubeadm init --control-plane-endpoint=[master ip]:6443 --pod-network-cidr=10.0.0.0/16 --upload-certs
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubeadm token create --print-join-command
```
### calico(master)
```
kubectl apply -f calico.yaml
```
ここまで来たらworkerにtoken貼り付けてクラスタに追加する
### helm(master)
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm
```
### nfs-subdir-external-provisioner
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -n kube-system --set nfs.server=[nfs server ip] --set nfs.path=[folder path] --set storageClass.defaultClass=true --set nfs.mountOptions={"nfsvers=4.0"}
```
### metallb
```
kubectl create namespace metallb-system
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system
kubectl apply -f ipaddresspool.yaml
```
### argocd
```
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install -n argocd argocd argo/argo-cd --set server.service.type="LoadBalancer" --set server.service.loadBalancerIP="[argo cd access ip]"
kubectl -n argocd get secret/argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
