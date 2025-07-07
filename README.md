# Guia de Execução – Guess Game no Kubernetes

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

 
## Ordem da execução dos comandos

Todos os recursos desta aplicação estão sob o namespace `guess-game`. Portanto, ao utilizar o `kubectl` para inspecionar recursos como `pods`, `deployments`, `HPA`, `services` ou `PVCs`, deve incluir a flag `-n guess-game1`.

Exemplo: `kubectl -n guess-game get pods`.

1. Navegar até o diretório do projeto

    Abra o terminal e vá até o local onde você extraiu os arquivos ou clonou o repositório.

2. Criar o namespace da aplicação

    ```bash
    kubectl apply -f namespace.yaml
    ```

3. Criar volumes persistentes do banco de dados PostgreSQL

    ```bash
    kubectl apply -f postgresdb-pv.yaml
    kubectl apply -f postgresdb-pvc.yaml
    ```

    Utilizei a política de acesso `ReadWriteMany` para permitir leitura e escrita por múltiplos nós.

4. Deploy do banco de dados PostgreSQL

    ```bash
    kubectl apply -f postgresdb-deployment.yaml
    kubectl apply -f postgresdb-service.yaml
    ```

    O Service foi configurado como ClusterIP, permitindo que o backend acesse o banco internamente dentro do cluster.

5. Deploy do backend (Flask) com HPA

    ```bash
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
    kubectl apply -f frontend-secret.yaml
    ```

    O secret nesse caso é o arquivo de configuração do nginx.conf, que foi configurado para tanto servir front estatico como proxy reverso para backend, abaixo explico melhor a decisão de usar. Caso precise de alguma alteração, basta alterar no secret o arquivo dentro, mas primeiro precisa converter em base64, por isso deixei ele alocado no projeto também.

8. Conferir se tudo subiu corretamente

    ```bash
    kubectl -n guess-game get deployment
    kubectl -n guess-game get pods
    kubectl -n guess-game get services
    kubectl -n guess-game get secret
    kubectl -n guess-game get pvc
    kubectl get pv
    ```

7. Acessar frontend

   Como o frontend está exposto via `NodePort` na porta `30080`, basta acessá-lo diretamente por: `http://<IP do Cluster>:30080`.

    Como usei o minikube, usei o comando: `minikube service frontend -n guess-game --url` que fará um tunnel e listará a URL para abrir no navegador. Esse comando criará um túnel e exibirá a URL que pode ser acessada diretamente no navegador. Caso esteja usando o driver Hyper-V (o que não foi o meu caso), `minikube ip`, utilizar esse ip + a porta, ou seja: `http://<IP em minikube ip>:30080`.

## Considerações finais

Para este projeto, utilizei o Minikube em conjunto com o WSL. Como estou usando o Windows 11 Home, não tenho suporte ao Hyper-V, portanto precisei usar o WSL. Nessa configuração, o Minikube não é executado como uma VM, mas sim como um container Docker, o que pode dificultar um pouco a exposição dos serviços via NodePort.

Por esse motivo, para acessar o frontend, utilizei o comando `minikube service frontend -n guess-game --url` que cria um túnel e fornece a URL temporária para acessar o serviço. Como a porta exposta pode mudar a cada execução, optei por manter o backend como ClusterIP e utilizar o NGINX como proxy reverso, seguindo a mesma abordagem usada na Prática 01 com Docker Compose. O problema dessa abordagem, é que o backend e o frontend ficam dependentes entre si, o que nessa prática acredito que não seria um problema, já que não é necessário o acesso da api externa, apenas pelo frontend, e para a aplicação estar funcionando, ambos precisam existir (em um ambiente produtivo essa prática não seria recomendada).

No entanto, existem outras alternativas. Eu poderia ter exposto o backend também como NodePort e, durante o build da imagem do frontend, definido a variável de ambiente REACT_APP_BACKEND_URL apontando para: `http://<IP do cluster>:<porta exposta no service backend nodeport>`.

Entretanto, devido à aleatoriedade da porta no Minikube com túnel, preferi manter o backend como ClusterIP e delegar ao NGINX a função de redirecionar as requisições para o backend e facilitar os testes locais. Dessa forma, na variável REACT_APP_BACKEND_URL, utilizei apenas "/api" durante o build do frontend.

Além disso, enfrentei dificuldades para configurar corretamente o Ingress Controller, e por isso decidi não utilizá-lo, evitando a necessidade de configurar entradas no /etc/hosts. Apesar disso, reconheço que a abordagem ideal seria com o uso de Ingress, o que permitiria expor tanto o frontend quanto o backend sob o mesmo host, diferenciando apenas pelo path, como por exemplo:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guess-game-ingress
  namespace: guess-game
spec:
  rules:
    - host: guessgame.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 5000
```
