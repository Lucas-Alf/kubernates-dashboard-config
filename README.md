# Tutorial para instalação do Kubernetes

Para a instalação do cluster Kubernetes foi utilizado os seguintes materiais como base: 
* [Intensivo Kubernetes: O mínimo que um dev precisa saber](https://youtu.be/5unI7VPnASM)
* [Setup Kubernetes Cluster Step By Step](https://youtu.be/UWg3ORRRF60)
* [How To Install Kubernetes Cluster On Ubuntu 16.04](https://www.edureka.co/blog/install-kubernetes-on-ubuntu)

## Requisitos mínimos
É recomendado que o cluster envolva ao menos 3 maquinas, pois o Kubernetes em si também possui um custo computational para realizar a orquestração dos containers.
### Master
* Memoria: 2GB RAM
* Processador: 2xCPU Cores

### Nodos
* Memoria: 1GB RAM
* Processador: 1xCPU Core

## Hardware utilizado
Para o desenvolvimento do cluster foram utilizadas máquinas virtuais disponibilizadas pelo [LARCC](https://larcc.setrem.com.br/).

O cluster foi composto por 4 VMs com a mesma configuração, sendo 1 Master e 3 Nodos.
### Master / Nodos
* Processador: 2xCores
* Memória: 4GB RAM
* Armazenamento: 20GB
* OS: Ubuntu Server 20.04

# Instalação do Kubernetes
As seguintes etapas devem ser realizadas tanto na maquina Master quanto nos Nodos como usuário administrador.

## Entrar como usuário "sudo" e atualizar o repositório de pacotes
```
$ sudo su
# apt-get update
```

## Desabilitar a utilização do espaço de Swap
O espaço de Swap do sistema operacional deve ser desabilitado para a instalação do Kubernetes, ao contrário podem ocorrer erros durante a execução.
```
# swapoff -a
```
Verificar se existem partições de swap no fstab, caso existir devem ser removidas ou comentadas.
```
# nano /etc/fstab
```
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/fstab.png)

## Alterar o Hostname da máquina
O Hostname da máquina será a forma como ela será identificada dentro do cluster, este nome irá aparecer no dashboard e nas consultas por nodos.

Para construir um padrão de nomenclaturas, podemos chamar a máquina Master de "kmaster" e os nodos de "knode0", "knode1", "knode2"...

```
# nano /etc/hostname
```
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/hostname.png)

## Definir o IP como estático
Para que as maquinas do cluster não percam o seu IP caso sejam desligadas ou reiniciadas, é necessário definir o IP como estático.

Atualize o arquivo `/etc/network/interfaces` com uma configuração de IP estático, a formatação pode mudar dependendo da versão do Ubuntu.

![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/static-ip.png)


## Configurar o arquivo de hosts
Para que as maquinas possam se encontrar dentro do cluster, é necessário configurar o arquivo de hosts com os IPs estáticos configurados anteriormente.

* No arquivo de hosts da maquina Master deve haver os IPs de todos os nodos.
* Já nos nodos pode haver somente o IP da maquina master.
```
# nano /etc/hosts
```
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/hostfile.png)

## Instalar o pacote do OpenSSH-Server
```
# sudo apt-get install openssh-server
```

## Instalar os pacotes do Docker
O seguinte comando irá instalar Docker e todas as suas dependencies, podendo demorar um pouco.
```
# sudo apt-get install -y docker.io
```

## Instalar os pacotes do Curl
```
apt-get update && apt-get install -y apt-transport-https curl
```

## Adicionar o repositório do Kubernates ao sistema
```
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
# apt-get update
```

## Instalar os pacotes do kubeadm, Kubelet And Kubectl
```
# apt-get install -y kubelet kubeadm kubectl
```

## Configurar as variáveis de ambiente do Kubernetes
Na próxima etapa será definido o "cgroup-driver" como "cgroupfs" nas variaveis de ambiente do Kubernetes.

Adicione a seguinte linha:
```
Environment=”cgroup-driver=systemd/cgroup-driver=cgroupfs”
```
no arquivo:
```
# nano /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/environment.png)

## Pronto!
A configuração inicial do Kubernates foi realizada, as próximas etapas serão definir uma maquina como Master, no nosso caso o "knode", e adicionar os nodos ao cluster.

# Configuração do Master
As seguintes etapas devem ser realizadas somente na maquina Master.

## Iniciar o cluster da maquina Master
O seguinte comando irá iniciar o cluster, altere o `<ip-address-of-kmaster-vm>` pelo IP da maquina master.
```
# kubeadm init --apiserver-advertise-address=<ip-address-of-kmaster-vm> --pod-network-cidr=192.168.0.0/16
```

Dois resultados deste comando devem ser anotados.
1. Os comandos para copiar o arquivo .config do cluster serão exibidos como:
```
 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

2. O comando utilizado para fazer os nodos se juntarem ao cluster será algo como:
```
 kubeadm join <ip-address-of-kmaster-vm>:port --token ...
```

## Copiar o arquivo .config do cluster
Execute os comandos exibidos anteriormente como um usuário **NÃO ROOT**. 

Pode ser criado um novo usuário somente para o Kubernetes neste caso.
```
# sudo adduser <user-name>
```

Depois troque para o novo usuário, e execute os comandos.
```
# su <user-name>
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Para verificar se o cluster foi iniciado com sucesso, pode executar o seguinte comando e verificar se os pods estão sendo iniciados.
```
$ kubectl get pods -o wide --all-namespace
```

## Instalação da rede interna entre os Pods
No comando anterior deve ser notado que todos os Pods ficaram com o status "Running" menos o "kube-dns", para resolve isso, será instalado a rede intra-pod chamada de [CALICO](https://docs.projectcalico.org).
```
$ kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
$ kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

Após instalação, todos Pods devem ficar com o status "Running", incluindo os novos pods do CALICO.

## Instalação do Kubernates Dashboard
A instalação do [Dashboard](https://github.com/kubernetes/dashboard) deve ser realizada antes dos outros nodos se juntarem ao cluster, ao contrário pode ocorrer do Dashboard ser instalado em um nodo, e não no Master.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
```

Para ter acesso ao Dashboard, execute o seguinte comando:
```
kubectl proxy
```

O Dashboard ficara disponível em:
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Para conseguir logar no Dashboard, vamos criar uma conta de serviço com direitos administrativos ao cluster. 
```
$ kubectl create serviceaccount dashboard -n default
$ kubectl create clusterrolebinding dashboard-admin -n default 
  --clusterrole=cluster-admin 
  --serviceaccount=default:dashboard
```
Agora para conseguir o Token de acesso, execute o seguinte comando:
```
$ kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```
O Token pode ser utilizado para realizar o login no Dashboard.

# Configuração dos nodos
As seguintes etapas devem ser realizadas somente nas máquinas nodos do cluster.

## Adicionando um nodo ao cluster
Para adicionar um novo nodo ao cluster somente é necessário executar o comando `sudo kubeadm join ...` que foi gerado durante a instalação do cluster, na maquina nodo que deseja adicionar.

Ao executar esse comando o nodo irá automaticamente entrar na cluster, e você poderá visualiza-lo no Dashboard.

Pode demorar alguns minutos até o novo nodo ficar utilizável, pois ao entrar no cluster ele irá começar a instalar uma série de dependências no nodo, o progresso da instalação é visível no Dashboard.
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/Dashboard.jpg)

