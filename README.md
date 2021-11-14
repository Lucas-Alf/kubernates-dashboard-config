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

O cluster foi composto por 4 VMs com a mesma configuração, sendo 1 Master e 2 Nodos.
### Master / Nodos
* Processador: 2xCores
* Memória: 2GB RAM
* Armazenamento: 20GB
* OS: Ubuntu Server 20.04

# Instalação do Kubernetes
As seguintes etapas devem ser realizadas tanto na maquina Master quanto nos Nodos como usuário administrador.

## Atualizar o repositório de pacotes
```
sudo apt update
```

## Desabilitar a utilização do espaço de Swap
O espaço de Swap do sistema operacional deve ser desabilitado para a instalação do Kubernetes, ao contrário podem ocorrer erros durante a execução.
```
swapoff -a
```
Verificar se existem partições de swap no fstab, caso existir devem ser removidas ou comentadas.
```
nano /etc/fstab
```
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/fstab.png)

## Alterar o Hostname da máquina
O Hostname da máquina será a forma como ela será identificada dentro do cluster, este nome irá aparecer no dashboard e nas consultas por nodos.

Para construir um padrão de nomenclaturas, podemos chamar a máquina Master de "kmaster" e os nodos de "knode0", "knode1", "knode2"...

```
nano /etc/hostname
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
nano /etc/hosts
```
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/hostfile.png)

## Instalar o pacote do OpenSSH-Server
```
sudo apt-get install openssh-server
```

## Instalar os pacotes do Docker
O seguinte comando irá instalar Docker e todas as suas dependencies, podendo demorar um pouco.
```
sudo apt-get install -y docker.io
```

## Instalar os pacotes do Curl
```
apt-get update && apt-get install -y apt-transport-https curl
```

## Adicionar o repositório do Kubernates ao sistema
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```
```
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
apt-get update
```

## Instalar os pacotes do kubeadm, Kubelet And Kubectl
```
apt-get install -y kubelet kubeadm kubectl
```

## Configurar o Docker para utilizar o systemd como padrão
```
sudo nano /etc/docker/daemon.json
```
Adicionar o seguinte conteudo:
```
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

Reiniciar o serviço do docker
```
sudo systemctl restart docker
```

## Reiniciar a máquina
Para garantir que as configurações realizadas no namespace e variaveis de ambiente foram aplicadas, temos que reiniciar a máquina.
```
sudo reboot
```

## Pronto!
A configuração inicial do Kubernates foi realizada, as próximas etapas serão definir uma maquina como Master, no nosso caso o "knode", e adicionar os nodos ao cluster.

# Configuração do Master
As seguintes etapas devem ser realizadas somente na maquina Master.

## Iniciar o cluster da maquina Master
O seguinte comando irá iniciar o cluster, altere o `<ip-address-of-kmaster-vm>` pelo IP da maquina master.
```
kubeadm init --apiserver-advertise-address=<ip-address-of-kmaster-vm> --pod-network-cidr=192.168.0.0/16
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
Deve ser criado um novo usuário somente para o Kubernetes neste caso.
```
sudo adduser k8s
usermod -aG sudo k8s
```

Depois troque para o novo usuário, e execute os comandos.
```
su k8s
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Para verificar se o cluster foi iniciado com sucesso, pode executar o seguinte comando e verificar se os pods estão sendo iniciados.
```
kubectl get pods -o wide -A
```

## Instalação da rede interna entre os Pods
No comando anterior deve ser notado que todos os Pods ficaram com o status "Running" menos o "kube-dns", para resolve isso, será instalado a rede intra-pod chamada de [CALICO](https://docs.projectcalico.org).
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

Após instalação, todos Pods devem ficar com o status "Running", incluindo os novos pods do CALICO.

## Instalação do Kubernates Dashboard
A instalação do [Dashboard](https://github.com/kubernetes/dashboard) deve ser realizada antes dos outros nodos se juntarem ao cluster, ao contrário pode ocorrer do Dashboard ser instalado em um nodo, e não no Master.
```
git clone https://github.com/Lucas-Alf/kubernates-dashboard-config.git
```
```
cd kubernates-dashboard-config
kubectl apply -f alternative.yaml
```

Para conseguir logar no Dashboard sem senha precisamos executar: 
```
kubectl create clusterrolebinding dashboard-admin -n default --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:kubernetes-dashboard
```

# Configuração dos nodos
As seguintes etapas devem ser realizadas somente nas máquinas nodos do cluster.

## Adicionando um nodo ao cluster
Para adicionar um novo nodo ao cluster somente é necessário executar o comando `sudo kubeadm join ...` que foi gerado durante a instalação do cluster, na maquina nodo que deseja adicionar.

Ao executar esse comando o nodo irá automaticamente entrar na cluster, e você poderá visualiza-lo no Dashboard.

Pode demorar alguns minutos até o novo nodo ficar utilizável, pois ao entrar no cluster ele irá começar a instalar uma série de dependências no nodo, o progresso da instalação é visível no Dashboard.
![](https://github.com/RSD-II/lucasalf/blob/main/Kubernetes/Dashboard.jpg)

# Acessando o Dashboard no LARCC
Para acessar o dashboard hospedado na VM dentro do LARCC, é necessário utilizar um serviço de relay.
1. Criar uma conta em: https://webhookrelay.com/
2. Instalar o webhook relay na máquina master:
```
curl https://my.webhookrelay.com/webhookrelay/downloads/install-cli.sh | bash
```
3. Criar um novo token em: https://my.webhookrelay.com/tokens
4. Logar com token na maquina master:
```
relay login -k meutoken -s meusecret
```
5. Iniciar um proxy com o kubectl em um segundo terminal
```
kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 8001:80 --address 0.0.0.0
```
6. Inicializar o Relay:
```
relay connect localhost:8001
```
7. Acessar a URL gerada pelo Relay.

