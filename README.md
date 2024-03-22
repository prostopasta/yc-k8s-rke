### Set k8s for Yandex Cloud use RKE

* Ubuntu v22.04 (2CPU, 40GB HDD, 6Gb RAM)
* k8s v1.27.11

#### Ansible install (terminal_host who ssh connect to VM's cluster)
MacOS
```Bash
brew install ansible
```
Ubuntu
```Bash
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible
```

### Configure terraform provider on host
Create ~/.terraformrc
```
provider_installation { 
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
} 
```


Получить oauth токен 
https://cloud.yandex.ru/ru/docs/cli/operations/authentication/user -
```Bash
yc config set token <token>
yc iam create-token
export YC_TOKEN=<token>
```

Получить варианты images
```Bash
yc compute image list --folder-id standard-images|grep ubuntu-22 
```
Получить  subnet_id
```Bash
 yc vpc subnets list  --cloud-id=<cloud-id> --folder-id=<folder-id>
```

#### Initialisation VM's and Configuration (dynamic inventory Ansible)
```Bash
cd terraform
terraform init
terraform apply
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook -i ../ansible/host.ini ../ansible/playbook.yml

```

### Install RKE (terminal_host who ssh connect to VM's cluster)
MacOS
```Bash

brew install rke
```

Ubuntu
```Bash
curl -s https://api.github.com/repos/rancher/rke/releases/latest | grep download_url | grep amd64 | cut -d '"' -f 4 | wget -qi -
chmod +x rke_linux-amd64
sudo mv rke_linux-amd64 /usr/local/bin/rke
rke --version
rke config --list-version --all
```

### Deploy k8s
```Bash
cd ../rke_config
vm cluster.yml # Replace IP for nodes
# Remove old config cluster
rke up --ignore-docker-version

kubectl --kubeconfig kube_config_cluster.yml get nodes  -o wide  
NAME    STATUS   ROLES                      AGE     VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
node1   Ready    controlplane,etcd,worker   7m2s    v1.27.11   10.128.0.22   <none>        Ubuntu 20.04.3 LTS   5.11.0-43-generic   docker://26.0.0
node2   Ready    worker                     6m55s   v1.27.11   10.128.0.16   <none>        Ubuntu 20.04.3 LTS   5.11.0-43-generic   docker://26.0.0
node3   Ready    worker                     6m55s   v1.27.11   10.128.0.8    <none>        Ubuntu 20.04.3 LTS   5.11.0-43-generic   docker://26.0.0


scp kube_config_cluster.yml <username>@<vm_ip>:~/ & ssh <username>@<vm_ip>
```

### Install Kubectl (master node)
```Bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

### Set kubeconfig
```Bash
mkdir ~/.kube/
cat kube_config_cluster.yml >~/.kube/k8s-hls & export KUBECONFIG=$(find ~/.kube -maxdepth 1 -type f -name '*' | tr "\n" ":")
```


### Test app
```bash
cd ../k8s
export KUBECONFIG=<path_to_project>/rke_config/kube_config_cluster.yml
kubectl apply -f deployment.yaml  
kubectl get pods  -l app=my-test -o wide                                          
NAME                       READY   STATUS    RESTARTS   AGE   IP          NODE    NOMINATED NODE   READINESS GATES
my-test-57fcc94cbb-g5m2d   1/1     Running   0          98s   10.42.2.7   node2   <none>           <none>
my-test-57fcc94cbb-zn4l2   1/1     Running   0          98s   10.42.0.6   node1   <none>           <none>
                                          
kubectl get pods  -l app=my-test  -o custom-columns=POD_IP:.status.podIPs    
POD_IP
[map[ip:10.42.2.7]]
[map[ip:10.42.0.6]]
```

Создадим сервис
```Bash
kubectl apply -f service.yaml            
kubectl get svc my-test                                                                         
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
my-test   ClusterIP   10.43.255.99   <none>        8080/TCP   13s


kubectl describe svc my-test                                                                   
Name:              my-test
Namespace:         default
Labels:            app=my-test
Annotations:       <none>
Selector:          app=my-test
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.43.255.99
IPs:               10.43.255.99
Port:              <unset>  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.42.0.6:8080,10.42.2.7:8080
Session Affinity:  None
Events:            <none>

kubectl get endpointslices -l kubernetes.io/service-name=my-test                        
NAME            ADDRESSTYPE   PORTS   ENDPOINTS             AGE
my-test-b9zh4   IPv4          8080    10.42.2.7,10.42.0.6   71s
```

Пробросим порты
```Bash
kubectl port-forward deployment.apps/my-test 8080:8080
или
kubectl port-forward svc/my-test  8080:8080
```

Добавим ингресс
```Bash
kubectl apply -f ingress.yaml  
kubectl get ingress 
sudo vi /etc/hosts #add ingress ip -> dns name   
# 158.160.50.119 hi.my.test.ru  
curl  hi.my.test.ru                                                                                    
```

### Clear
```Bash
#exit from cluster node
pwd rke_config
rke remove --ignore-docker-version
cd ../terraform  
terraform destroy
```
