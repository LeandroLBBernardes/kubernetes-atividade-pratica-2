# Guia de Execução – Guess Game no Kubernetes

## Pré-requisitos

Certifique-se de possuir ao menos um cluster Kubernetes válido configurado. Algumas opções que você pode utilizar são:

* **Docker Engine**
* **Minikube**
* **Microk8s**

# Sumário

1. [Descrição dos Componentes](#descrição-dos-componentes)  
2. [Ordem da Execução dos Comandos](#ordem-da-execução-dos-comandos)  
3. [Considerações Finais](#considerações-finais)

## Descrição dos componentes

Namespace:
  - namespace.yaml

O recurso `namespace` define um ambiente isolado dentro do cluster Kubernetes chamado `guess-game`. Esse namespace serve para organizar e separar os recursos do projeto, como os deployments, services e secrets.

---

Backend:
  - backend-deployment.yaml
  - backend-service.yaml
  - backend-hpa.yaml

Para o `deployment` do `backend` a imagem utilizada para o container foi a `leandrolbbernardes/pucminas:backend-flask-guess-game` que contém a aplicação desenvolvida em Flask. As variáveis de ambiente definidas configuram a aplicação para o ambiente de produção e fornecem os dados de conexão com o banco de dados PostgreSQL.

No deployment, o Init Container wait-for-postgres tem a função de aguardar até que o serviço do banco de dados PostgreSQL (postgresdb) esteja acessível na porta 5432, garantindo que a aplicação principal só inicie quando o banco estiver pronto para receber conexões e assim evitar erros de subida como `CrashLoopBackOff`.

O container também define limites e requisições de recursos, assegurando que ele solicite no mínimo 10Mi de memória e 10m de CPU para iniciar, com um limite máximo de 256Mi de memória e 250m de CPU. 

O livenessProbe foi utilizado para verificar se o container continua funcionando corretamente ao longo do tempo. Foi configurada uma sonda TCP na porta 5000, que será ativada após um delay inicial de 60 segundos.

**initialDelaySeconds: 60**
Aguarda 60 segundos após o início do container antes de iniciar as verificações.

**periodSeconds: 10**
Realiza uma verificação a cada 10 segundos.

**timeoutSeconds: 5**
Considera a sonda com falha se a resposta demorar mais de 5 segundos.

**failureThreshold: 5**
O container será reiniciado se a sonda falhar 5 vezes consecutivas.

O `service` do `backend` é do tipo `ClusterIP`, utilizado para expor e facilitar o acesso ao pod do backend dentro do cluster Kubernetes. Sendo o NGINX responsável pelo proxy reverso.

O `HPA` está configurado para ajustar dinamicamente o número de réplicas entre 1 e 3, conforme a demanda. A métrica utilizada é a utilização de CPU, com um alvo de 50% de uso médio entre os pods. Quando o consumo de CPU ultrapassar esse valor, o Kubernetes criará novas réplicas do pod para distribuir a carga. Necessita do metrics-server.

---

Frontend:
  - frontend-deployment.yaml
  - frontend-service.yaml
  - frontend-secret.yaml

Para o `frontend` a a imagem utilizada para o container foi a `leandrolbbernardes/pucminas:frontend-react-guess-game`, as especificações de recursos são os mesmo que do backend. Além disso, faz o uso de um volume baseado em secret, que permite a injeção de um arquivo de configuração do NGINX personalizado. O que torna o NGINX configurável sem a necessidade de reconstruir a imagem.

O `service` foi configurado como NodePort. Neste caso, a porta exposta é a 30080, que redireciona para a porta 80 do container onde o servidor NGINX está escutando. Foi excolhido NodePort para facilitar o acesso externo sem precisar de ingress ou port-foward.

O `secret` do frontend armazena de forma o conteúdo do arquivo nginx.conf, necessário para a configuração do NGINX utilizado no frontend. O Secret é do tipo Opaque e permite armazenar dados arbitrários codificados em base64. O conteúdo do nginx.conf é montado no container do frontend através de um volume baseado em Secret, substituindo o arquivo padrão de configuração do NGINX. 

---

Banco de dados:
  - postgresdb-deployment.yaml
  - postgresdb-service.yaml
  - postgresdb-pv.yaml
  - postgresdb-pvc.yaml

O `deployment` do banco de dados PostgreSQL utiliza a imagem oficial `postgres:latest` e está configurado para rodar uma única réplica no namespace guess-game. As variáveis de ambiente definem o usuário, a senha e o nome do bancos. Os recursos solicitados e os limites são os mesmos do backend e frontend. Além disso, o volume postgresdb-storage é montado no caminho /var/lib/postgresql/data para persistência dos dados, que são armazenados em um volume persistente ligado a um PersistentVolumeClaim.

O PersistentVolume, `postgresdb-pv`, utiliza um caminho local no host /data/postgres, com capacidade de 1Gi e política de retenção Retain, preservando os dados mesmo se o volume for removido. Tanto o PV quanto o PersistentVolumeClaim `postgresdb-pvc` possuem o campo storageClassName vazio para indicar que são volumes estáticos, que são pré-configurados manualmente e não provisionados dinamicamente pelo cluster. O modo de acesso configurado é ReadWriteMany, permitindo que múltiplos pods possam acessar o volume simultaneamente para leitura e escrita.

O `service` do banco de dados é do tipo ClusterIP, expondo internamente a aplicação na porta 5432.

---

Ingress:
  - guess-game-ingress.yaml

O recurso Ingress é responsável por expor os serviços do cluster Kubernetes para fora, permitindo que os usuários acessem as aplicações utilizando um único ponto de entrada, geralmente com base em nomes de host e caminhos. Neste caso, o Ingress está configurado para o namespace guess-game e utiliza a classe de Ingress do NGINX.

A anotação nginx.ingress.kubernetes.io/rewrite-target: /$1 é usada para reescrever a URL antes que ela seja encaminhada ao serviço de backend. O valor /$1 corresponde ao primeiro grupo de captura das expressões regulares definidas nos paths, permitindo remover prefixos da URL externa ao encaminhar a requisição para o serviço correto. Isso ajuda a poder usar rotas como /maker em outra aba.

Foi separado em dois paths usando o mesmo host: `guessgame.local`, esses paths foram para separar o tráfego do frontend para backend. (exige ingress controller) 

 
## Ordem da execução dos comandos

Todos os recursos desta aplicação estão sob o namespace `guess-game`. Portanto, ao utilizar o `kubectl` para inspecionar recursos como `pods`, `deployments`, `HPA`, `services` ou `PVCs`, deve incluir a flag `-n guess-game1`.

Exemplo: `kubectl -n guess-game get pods`.

1. Extrair os arquivos da pasta `k8s-manifest`, navegar até o diretório onde foi feita a extração

    Abra o terminal e vá até o local onde você extraiu os arquivos ou clonou o repositório.

2. Criar o namespace da aplicação

    ```bash
    kubectl apply -f namespace.yaml
    ```

3. Criar volumes persistentes do banco de dados PostgreSQL

    ```bash
    kubectl apply -f postgresdb-pv.yaml

    # espere até que tenha subido corretamente o pv
    kubectl get pv

    # apos subir o pv corretamente
    kubectl apply -f postgresdb-pvc.yaml

    # aguarde até o status ficar: Bound
    kubectl -n guess-game get pvc
    ```

    Utilizei a política de acesso `ReadWriteOnce` para permitir leitura e escrita.

4. Deploy do banco de dados PostgreSQL

    ```bash
    kubectl apply -f postgresdb-deployment.yaml
    kubectl apply -f postgresdb-service.yaml
    ```

    O Service foi configurado como ClusterIP, permitindo que o backend acesse o banco internamente dentro do cluster.

5. Deploy do backend (Flask) com HPA

    ```bash
    # garanta que o banco tenha subido corretamente primeiro, o código flask apita erro caso não tenha subido o banco
    kubectl -n guess-game get pods

    # caso nao tenha subido, o init containers se encarregara de somente subir o container principal
    # assim que o servico do banco estiver de pe
    kubectl apply -f backend-deployment.yaml
    kubectl apply -f backend-service.yaml
    kubectl apply -f backend-hpa.yaml
    ```

    OBS: Para o HPA funcionar, o `metrics-server` tem de estar no cluster.

    ```bash
    kubectl get deployment metrics-server -n kube-system
    ```

6. Deploy do frontend (Node + NGINX)

    ```bash
    kubectl apply -f frontend-deployment.yaml
    kubectl apply -f frontend-service.yaml
    ```

8. Conferir se tudo subiu corretamente

    ```bash
    kubectl -n guess-game get deployment
    kubectl -n guess-game get pods
    kubectl -n guess-game get services
    kubectl -n guess-game get secret
    kubectl -n guess-game get pvc
    kubectl get pv
    ```

9. Subir ingress (caso tenha ingress controller)

    ```bash
      kubectl apply -f guess-game-ingress.yaml

      # conferir se subiu corretamente
      kubectl -n guess-game get ingress
    ```

7. Acessar frontend

    Como o frontend está exposto via `NodePort` na porta `30080`, basta acessá-lo diretamente por: `http://<IP do Cluster>:30080`.

    OBS: Como usei o minikube, usei o comando: `minikube service frontend -n guess-game --url` que fará um tunnel e listará a URL para abrir no navegador. Esse comando criará um túnel e exibirá a URL que pode ser acessada diretamente no navegador.

    Caso opte por acessar a aplicação via Ingress, isso também é possível. Para isso, adicione uma entrada no arquivo `/etc/hosts` do seu sistema, apontando o domínio com o nome do host do ingress para o IP do seu cluster. Exemplo:

    ```
    <ip do cluster>   guessgame.local
    ```

    Após isso, você poderá acessar a aplicação normalmente pelo navegador, usando o endereço: `http://guessgame.local`

    OBS: Certifique-se de que o Ingress Controller (como o NGINX Ingress) esteja corretamente instalado e funcionando no cluster.

## Considerações finais

Para este projeto, adotei duas abordagens complementares para garantir o funcionamento adequado da aplicação: NodePort e Ingress.

### Primeira abordagem: NodePort

Nessa abordagem, o frontend atua como um proxy reverso tanto para os arquivos estáticos (HTML, CSS, JS) quanto para o serviço do backend. Isso gera um acoplamento direto entre as duas aplicações, tornando o backend acessível somente quando o frontend estiver em funcionamento. Ou seja, a API se torna dependente da aplicação de frontend, o que não é ideal do ponto de vista arquitetural, especialmente em ambientes de produção.

### Segunda abordagem: Ingress

Por isso, como forma de desacoplar os componentes e permitir mais flexibilidade, implementei também a exposição via Ingress. Essa abordagem permite isolar o funcionamento de cada serviço e acessá-los de forma independente, com rotas bem definidas. O Ingress também facilita o roteamento baseado em path, como por exemplo:

```arduino
http://guessgame.local/        → Frontend  
http://guessgame.local/api     → Backend
```

Para que isso funcione, é necessário adicionar uma entrada no arquivo /etc/hosts, apontando o domínio para o IP do cluster. Exemplo:

```
<IP_DO_CLUSTER>   guessgame.local
```

Assim, o navegador poderá resolver corretamente os domínios definidos no Ingress Controller (como o NGINX Ingress).

### Atualização dos componentes

1. Atualize o código-fonte do serviço conforme necessário.

2. Reconstrua a imagem Docker e envie-a para o Docker Hub, pode ser a mesma tag, ou um incremento caso use CI/CD e queira manter um histórico.

3. Reaplique o manifesto .yaml do Deployment correspondente com o comando `kubectl apply -f ...`

Componentes sem ser deployment podem ser atualizados seguindo a mesma lógica usando o `kubectl apply -f ...`