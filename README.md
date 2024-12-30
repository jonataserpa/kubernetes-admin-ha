# kubernetes-admin-ha
 Kubernetes com Kubeadm - HA Stacked etcd Topology
 
# Guia de Instalação do Kubernetes com Kubeadm - HA Stacked etcd Topology

Este guia tem como objetivo auxiliar na criação de um cluster Kubernetes com alta disponibilidade (HA) utilizando o kubeadm e a topologia Stacked etcd. Nessa configuração, o etcd é executado diretamente nos nós do control plane, permitindo uma arquitetura simplificada e eficiente para ambientes que necessitam de maior resiliência.

A abordagem de alta disponibilidade com Stacked etcd é ideal para ambientes de produção que precisam minimizar pontos únicos de falha, garantindo maior robustez e continuidade dos serviços mesmo diante de falhas de nós. Este guia aborda desde os requisitos e preparativos até a configuração e inicialização do cluster com alta disponibilidade.

Nessa estrutura teremos mais de um control plane para orquestrar o cluster Kubernetes, então teremos um componente a mais para ser o balanceador de carga entre as requisições para o API Server. Vamos inserir o HAProxy como balanceador de carga. Vai ser da seguinte forma.
---

## Kubeadm - HA Stacked etcd Topology

![Topolgy](https://raw.githubusercontent.com/jonataserpa/kubernetes-admin-ha/refs/heads/main/loadbalancer.png)

### Requisitos

Para criar o cluster Kubernetes, precisamos de no mínimo 1 máquina para o HAProxy e 3 control planes. Os worker nodes depende do que você vai executar. Vou considerar 3 worker nodes nesse laboratório.

- 6 Máquinas Linux (aqui no caso vou utilizar Ubuntu 22.04)
- 2 GB de memória RAM
- 2 CPUs
- Conexão de rede entre as máquinas
- Hostname, endereço MAC e product_uuid únicos pra cada nó.
- Swap desabilitado
- Acesso SSH a todas as máquinas

### Requisitos de Hardware (HAProxy)
- 1 Máquina Linux (aqui no caso vou utilizar Ubuntu 22.04)
- 1 GB de memória RAM
- 1 CPU
- Conexão de rede entre as máquinas
- Hostname, endereço MAC e product_uuid único.
- Acesso SSH a máquina.

### Requisitos de Rede

#### Control Plane
O control plane requer a liberação das seguintes portas:

| Protocol | Direction | Port Range   | Purpose                     | Used By               |
|----------|-----------|--------------|-----------------------------|-----------------------|
| TCP      | Inbound   | 6443         | Kubernetes API server       | All                   |
| TCP      | Inbound   | 2379-2380    | etcd server client API      | kube-apiserver, etcd  |
| TCP      | Inbound   | 10250        | Kubelet API                 | Self, Control plane   |
| TCP      | Inbound   | 10259        | kube-scheduler              | Self                  |
| TCP      | Inbound   | 10257        | kube-controller-manager     | Self                  |

#### Worker Nodes
Os worker nodes requerem a liberação das seguintes portas:

| Protocol | Direction | Port Range   | Purpose                     | Used By             |
|----------|-----------|--------------|-----------------------------|---------------------|
| TCP      | Inbound   | 10250        | Kubelet API                 | Self, Control plane |
| TCP      | Inbound   | 10256        | kube-proxy                  | Self, Load balancers |
| TCP      | Inbound   | 30000-32767  | NodePort Services†          | All                 |

---

## Passos da Instalação

### Para a instalação e criação do cluster Kubernetes, vamos executar as seguintes etapas:

- Instalação e Configuração do HAProxy
- Instalação do Container Runtime (ContainerD)
- Instalação do kubeadm, kubelet e kubectl
- Inicialização do cluster Kubernetes
- Instalação do CNI cálico
- Incluir os worker nodes no cluster Kubernetes

## Instalação e Configuração do HAProxy

O primeiro passo é instalar e configurar o HAProxy.
Instalação do HA Proxy

```bash
sudo apt update && sudo apt install haproxy -y
```

Configuração do HAProxy:
A configuração do HAProxy é feita no arquivo /etc/haproxy/haproxy.cfg Abaixo segue como deve ser feita a configuração

```bash
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # SSL Configuration
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5000ms
    timeout client 60000ms
    timeout server 60000ms

frontend k8s-control-plane
    bind *:6443 # Escuta na porta 6443 em todas as interfaces
    mode tcp # Modo TCP para tráfego binário
    option tcplog # Loga conexões TCP
    default_backend k8s-control-plane # Redireciona para o backend "k8s-control-plane"


backend k8s-control-plane
    mode tcp # Modo TCP
    balance roundrobin # Distribuição de carga (round-robin)
    option tcp-check # Habilita checagem de saúde com TCP
    server control-plane-1 [IP_DO_CONTROL_PLANE_01]:6443 check inter 2000 fall 3 rise 2  # Servidor 1
    server control-plane-2 [IP_DO_CONTROL_PLANE_02]:6443 check inter 2000 fall 3 rise 2  # Servidor 2
    server control-plane-3 [IP_DO_CONTROL_PLANE_03]:6443 check inter 2000 fall 3 rise 2  # Servidor 3
```

Depois reinicie o HAProxy com o comando:

```bash
sudo systemctl restart haproxy && \
sudo systemctl enable haproxy && \ 
sudo systemctl status haproxy
```
Validação do HAProxy

Após configurar o HAProxy, valide que ele está funcionando corretamente:
Verificar se está escutando na porta 6443

```bash
sudo apt install net-tools -y
sudo netstat -tuln | grep 6443
```

## Instalação do Container Runtime

O Container Runtime é a base para executar contêineres no Kubernetes. O ContainerD é uma escolha popular devido à sua simplicidade, desempenho e suporte oficial do Kubernetes.
OBS: Essa etapa deve ser executada em todas as máquinas que vão fazer parte do cluster Kubernetes.

## Instalação dos módulos de Kernel do Linux
Para seguir com a instalação, primeiro é preciso habilitar 2 módulos no kernel do Linux:

- overlay ⇒ Usado pra unir camadas de file system (falo isso no curso de Docker)
- br_netfilter ⇒ Modulo de rede usado pra garantir a comunicação dos containers e do Kubernetes

Comandos para habilitar:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Ajustes de definições do Kernel:
```bash
# Configuração dos parâmetros do sysctl, fica mantido mesmo com reebot da máquina.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Aplica as definições do sysctl sem reiniciar a máquina
sudo sysctl --system
```
net.bridge.bridge-nf-call-iptables = 1 ⇒ Ativo as redes bridges pra passarem pelo iptables e assim as regras de firewall vão passar por essas redes também.

net.ipv4.ip_forward = 1 ⇒ Habilita o encaminhamento de pacotes IPv4 no sistema. Isso é essencial para que o host funcione como um roteador, encaminhando pacotes de rede de uma interface para outra.

net.bridge.bridge-nf-call-ip6tables = 1 ⇒ Similar ao bridge-nf-call-iptables, mas para tráfego IPv6.

## Instalação do ContainerD

OBS: A partir da versão 1.26 do Kubernetes, foi removido o suporte ao CRI v1alpha2 e ao Containerd 1.5. E até o momento que escrevo esse guia, o repositório oficial do Ubuntu não tem o Containerd 1.6, então precisamos usar o repositório do Docker pra instalar o ContainerD.

Kubernetes v1.26: Electrifying
https://kubernetes.io/blog/2022/12/09/kubernetes-v1-26-release/#cri-v1alpha2-removed

```bash
# Instalação de pré requisitos
sudo apt-get update -y
sudo apt-get install ca-certificates curl gnupg --yes
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Configurando o repositório
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt update -y && sudo apt install containerd.io -y
```

## Configuração padrão do Containerd
```bash
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml
```
Alterar o arquivo de configuração pra configurar o systemd cgroup driver.
Sem isso o Containerd não gerencia corretamente os recursos computacionais e vai reiniciar em loop
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

#Alterar a imagem do sandbox
sudo sed -i 's|sandbox_image = ".*"|sandbox_image = "registry.k8s.io/pause:3.10"|' /etc/containerd/config.toml
```

Agora é preciso reiniciar o containerd
```bash
sudo systemctl restart containerd
```

## Instalação do kubeadm, kubelet and kubectl
Com o containerd instalado, agora é preciso instalar o kubeadm, kubelet e o kubectl.
Instalação dos pacotes necessários

```bash
sudo apt-get update && \\
sudo apt-get install -y apt-transport-https ca-certificates curl
```

Download da chave pública do Repositório do Kubernetes
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Adicionando o repositório apt do Kubernetes
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Atualização do repositório apt e instalação das ferramentas
```bash
sudo apt-get update && \\
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Iniciação e Configuração do Cluster Kubernetes
A seguir, apresentamos os passos necessários para inicializar o cluster Kubernetes utilizando a topologia HA Stacked etcd. Este processo inclui a configuração inicial do Control Plane, a adição de nós adicionais do Control Plane e a inclusão de Worker Nodes no cluster.

### Iniciando o Cluster Kubernetes
A inicialização do cluster deve ser feita no primeiro nó do Control Plane. Esse comando configura o Kubernetes API Server e os demais componentes principais.

Comando para inicialização
No primeiro nó do Control Plane, execute o comando abaixo para iniciar o cluster:
```bash
sudo kubeadm init \
    --control-plane-endpoint=[IP_PRIVADO_LOAD_BALANCER]:6443 \
    --upload-certs \
    --apiserver-cert-extra-sans=[IP_PUBLICO_LOAD_BALANCER] \
    --pod-network-cidr=192.168.0.0/16
```

Parâmetros:

--control-plane-endpoint => Define o endereço do balanceador de carga para o cluster. Este será o ponto de entrada único para a comunicação com o Kubernetes API Server.
--upload-certs => Permite o compartilhamento seguro dos certificados necessários para adicionar nós adicionais do Control Plane.
--apiserver-cert-extra-sans => Especifica IPs ou domínios adicionais para serem incluídos no certificado do Kubernetes API Server.
--pod-network-cidr => Define o intervalo de IPs que será utilizado pela rede de Pods no cluster. Após executar o comando, você verá instruções sobre como configurar o cliente kubectl e os

### Fases do Kubeadm init

Durante o processo de inicialização do cluster, alguns estágios são executados:

preflight ⇒ Valida o sistema e verifica se é possível fazer a instalação. Ele pode exibir alertas ou erros, no caso de erro, ele sai da inicialização.

kubelet-start ⇒ Essa fase escreve o arquivo de configuração do kubelet e o arquivo de ambiente e, em seguida, iniciará o kubelet.

certs ⇒ Gera uma autoridade de certificação auto assinada para cada componente do Kubernetes. Isso garante a segurança na comunicação com o cluster.

kubeconfig ⇒ Gera o kubeconfig no diretório /etc/kubernetes e os arquivos utilizados pelo kubelet, controller-manager e o scheduler pra conectar no api-server.

kubelet-start, control-plane e etcd ⇒ Configura o kubelet pra executar os pods com o api-server, controller-manager, o scheduller e o etcd. Depois inicia o kubelet.

mark-control-plane ⇒ Aplica labels e tains no control plane pra garantir que não vai ser executado nenhum pod dentro dele.

bootstrap-token ⇒ Gera tokens de bootstrap usados para adicionar um nó a um cluster.

kubelet-finalize ⇒ Use a seguinte fase para atualizar as configurações relevantes para o kubelet após o bootstrap TLS.

addons ⇒ Adiciona o CoreDNS e o kube-proxy

## Configuração do kubectl
Com o Kubernetes iniciado, agora é preciso configurar o kubectl para se comunicar com o Api Server:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Execute o comando para listar os nodes
```bash
kubectl get nodes
```

## Adicionando mais Control Planes
Depois que o primeiro nó do Control Plane for inicializado, os nós adicionais do Control Plane podem ser adicionados ao cluster. Certifique-se de que os nós adicionais tenham o mesmo ambiente configurado.

Comando para adição de nós do Control Plane Execute o comando de join no nó adicional:
```bash
kubeadm join [IP_PRIVADO_LOAD_BALANCER]:6443 \
    --token [TOKEN] \
    --discovery-token-ca-cert-hash [CERT_HASH] \
    --control-plane \
    --certificate-key [CHAVE_CERTIFICADO]
```

Parâmetros:
--control-plane => Indica que o nó será configurado como parte do Control Plane.
--certificate-key => Fornece a chave necessária para compartilhar os certificados entre os nós do Control Plane. Se você não possui o comando completo, consulte a próxima seção para regenerar o comando necessário.

## Regerando o Comando de Join
Caso você tenha perdido o comando de join para adicionar novos nós ao Control Plane, ele pode ser regenerado com o seguinte comando no nó principal do Control Plane:

```bash
echo $(sudo kubeadm token create --print-join-command) \
    --control-plane --certificate-key $(sudo kubeadm init phase upload-certs --upload-certs | grep -vw -e certificate -e Namespace)
```
Esse comando gera automaticamente o token, o hash do certificado e a chave necessária para adicionar novos nós ao Control Plane.

## Adicionando Worker Nodes ao Cluster

Depois que o Control Plane estiver configurado, você pode adicionar Worker Nodes ao cluster. Esses nós executam os containers e fornecem os recursos de computação para suas aplicações.

Comando para inclusão dos Worker Nodes Execute o comando abaixo nos Worker Nodes:

```bash
kubeadm token create --print-join-command
```

Isso irá gerar o comando necessário para adicionar os nós. Em seguida, execute o comando gerado:

```bash
kubeadm join [IP_PRIVADO_LOAD_BALANCER]:6443 \
    --token [TOKEN] \
    --discovery-token-ca-cert-hash [CERT_HASH]
```
Parâmetros:
--token => Token de autenticação gerado pelo Control Plane para permitir a conexão.
--discovery-token-ca-cert-hash => Hash do certificado utilizado para validar a comunicação com o Control Plane.

## Joins dos Worker Nodes
Agora, o cluster Kubernetes tem apenas o control plane fazendo parte dele, então você deve executam o comando de join nos worker nodes para incluir eles no cluster. Mas antes, você precisa do comando. Então execute o comando abaixo no control plane e o comando de resultado execute nos worker nodes:

```bash
kubeadm token create --print-join-command
```

## Configurações Extras
Após a instalação e inicialização básica do cluster Kubernetes, algumas configurações adicionais são necessárias para otimizar e organizar seu cluster. Este capítulo aborda as etapas de instalação do CNI (Container Network Interface) e a organização do cluster, incluindo a atribuição de rótulos (labels) aos nós para facilitar a identificação e o gerenciamento.

## Instalação do CNI (Container Network Interface)
O Kubernetes precisa de um plugin CNI para gerenciar a comunicação de rede entre os pods e os nós. O Calico é uma das opções mais populares e robustas para gerenciar redes de containers, oferecendo recursos como redes definidas por software (SDN), segurança com políticas de rede e alta performance.

Execute os comandos abaixo para instalar o Calico no cluster:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml
```
O que esses comandos fazem?

Tigera Operator: O primeiro comando instala o operador do Calico, que é responsável por gerenciar e configurar os componentes do Calico no cluster. Custom Resources: O segundo comando configura os recursos personalizados que definem como o Calico deve operar no cluster, incluindo configurações de rede e políticas. Dicas importantes: Validação: Após a instalação, valide se o Calico foi instalado corretamente com o comando:
```bash
kubectl get pods -n calico-system
```

Certifique-se de que todos os pods relacionados ao Calico estão com o status Running.

## Organizando o seu cluster
Para facilitar o gerenciamento e a automação de tarefas no cluster, é importante usar rótulos (labels) nos nós. Rótulos são pares chave-valor que identificam atributos ou funções específicas dos nós, como sua função no cluster (Control Plane, etcd, Worker).

Rótulos para os nós do Control Plane
Os nós do Control Plane desempenham funções críticas, incluindo a execução do API Server, Scheduler e Controller Manager. Para identificá-los claramente, aplique o rótulo control-plane:
```bash
kubectl label node cp-1 node-role.kubernetes.io/control-plane=control-plane
kubectl label node cp-2 node-role.kubernetes.io/control-plane=control-plane
kubectl label node cp-3 node-role.kubernetes.io/control-plane=control-plane
```

Rótulos para os nós do etcd

Os nós que hospedam o etcd, o banco de dados distribuído usado pelo Kubernetes, também devem ser rotulados. Isso ajuda a diferenciar e gerenciar esses nós com facilidade:
```bash
kubectl label node cp-1 node-role.kubernetes.io/etcd=etcd
kubectl label node cp-2 node-role.kubernetes.io/etcd=etcd
kubectl label node cp-3 node-role.kubernetes.io/etcd=etcd
```
Rótulos para os Worker Nodes
Os Worker Nodes são responsáveis por hospedar os pods e executar as cargas de trabalho. Use o rótulo worker para identificar esses nós:
```bash
kubectl label node wn-1 node-role.kubernetes.io/worker=worker
kubectl label node wn-2 node-role.kubernetes.io/worker=worker
kubectl label node wn-3 node-role.kubernetes.io/worker=worker
```

Rodando a aplicação
Teste o cluster Kubernetes usando o manifesto abaixo:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

Referencias para o material:

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

https://kubernetes.io/docs/reference/networking/ports-and-protocols/

https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/
