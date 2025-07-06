# Guia de Execução – Guess Game no Kubernetes

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

Por esse motivo, para acessar o frontend, utilizei o comando `minikube service frontend -n guess-game --url` que cria um túnel e fornece a URL temporária para acessar o serviço. Como a porta exposta pode mudar a cada execução, optei por manter o backend como ClusterIP e utilizar o NGINX como proxy reverso, seguindo a mesma abordagem usada na Prática 01 com Docker Compose. O problema dessa abordagem, é que o backend fica dependente do container do front estar funcionando corretamente, o que nessa prática tbm não seria um problema, já que não é necessário o acesso da api externa, apenas pelo frontend.

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