# Como realizar deploy do blackbox-exporter no kubernetes

O objetivo dessa doc é realizar o deploy do prometheus **blackbox-exporter** em um cluster kubernetes de uma maneira simples e fácil.

# Pre-requisitos

 - docker
 - minikube
 - kubectl
 - helm

## Preparando o ambiente

Primeiro é necessário criar um cluster no minikube:

`minikube start --cpus 8 --memory 12384mb --disk-size='200000mb' --nodes 1`

> Observação: a quantidade de memória e cpus vai de acordo com o recurso
> do seu computador.

Criando o namespace monitoring:

`kubectl create namespace monitoring`

Instalando a stack kube-prometheus-stack com Helm:

 1. Adicionando o repositório helm da stack

```console
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

 2. Atualizando o repositório

`helm repo update`

 3. Realizando deploy da stack

Para esse deploy o values da stack foi modificado adicionando o seguinte bloco:

```yaml
    additionalScrapeConfigs:
      - job_name: blackbox
        metrics_path: /probe
        params:
          module: [http_2xx]
        static_configs:
          - targets:
            - https://www.google.com
        relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: prometheus-blackbox-exporter.monitoring:9115
```
O arquivo yaml que tá no diretório **kube-stack-prometheus** deste repositório.
```
helm upgrade --install kube-stack-prometheus prometheus-community/kube-prometheus-stack -f kube-stack-prometheus/values.yaml -n monitoring
```
É importante, após o deploy, verificar se os pods estão OK (com status running):

`kubectl get pods -n monitoring`

 4. Acessando a interface do prometheus
```
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-kube-prometheus 9090:9090
```
Só ir no navegador e acessar http://localhost:9090
 
 5. Acessando o Grafana
```
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-grafana 8080:80
```
Só ir no navegador e acessar http://localhost:8080

Usuário: admin

Senha: prom-operator

## Prometheus blackbox-exporter

Antes de realizar o deploy do blackbox-exporter é necessário realizar algumas alterações no *values.yaml*. Assim como o *values.yaml* da stack foi modificado o do blackbox também está com algumas modifições:

```yaml
config:
  modules:
    http_2xx:
      prober: http
      timeout: 5s
      http:
        valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
        # A modificao acontece aqui
        valid_status_codes: [200, 403]
        ##############################
        follow_redirects: true
        preferred_ip_protocol: "ip4"
```

> Lembrando que é possível configurar/modificar os values de acordo com a suas necessidades.

 - Deploy
```console
helm install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter -f blackbox-exporter/values.yaml -n monitoring
```

Depois do deploy feito é interessante visualizar as métricas em um dashboard, para isso vamos acessar o Grafana novamente:
```
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-grafana 8080:80
```
Só ir no navegador e acessar http://localhost:8080

Usuário: admin

Senha: prom-operator

Ir na parte de Dashboards > Import e inserir o código do dashboard que é: **7587**.

Por fim o resultado final é este:

![Dashboard](https://lh3.googleusercontent.com/i-H9-Sz20Ypw4KMH1weYztglWrSuCmIUcFnwg87T-hPgEHXhQQ9yIF3aCcLKLpDaalbmE-XIC-C-8Ti8jqw-ad35-_Br7SwquA05yF58kxqpMIBsZX3gB8mycZTCeOu8c5N0WOq8V-iYRr_Sbvho9r-tgQAELeoc8eUvfPDCt119j71oY00faGUwgEjThaKF1YlJjjqLqkh3pmK-d9y9ORd-4l0SMZBEtgznBJho38O2ijg-m114f6P8F3lK7OxBx-w7yWI2loUZ8-JZ0OudrREFUtUfhiW6kxlhx1WdKRKPuvqgqM9jYZMbBw510Q9hDBIz5iY5Fx89o4SPzk9qJahfvySSi8fYSXVcFGYTP1UzTpeaDSMGpTmPfEkqSz6zZkgZV8CjcxaJ8wYWOZnQIBC8uLRX3po599mKGYOAQYQ-6y-YA8Ydkdw_QrrPEiFc4gPN2Z4VzoLgWLncaZhPxEbcapnHz3rm3Y5SwWdb8JfxnaFJoTIzZuwxgWPPNJEDGoXgULScK6-NnyehwiL71E2IpeHHKJlmV_N5HqqiW2CRlcfHnwf7i-MIf21LbgWL2RMeyTf0boW4YdexFJ9sUgL3DO4hzx3fr2Ecy9XH5fpXAIdN4P99KFtK-Es6H5-HbDHxJyw_STpMsmH8TvrTBabLisejgcKficlzt1P0pvVPEiIzUqPz1gH0yYEhrsxDPRMtJfIvmDji9fuaasHQftEhMPe5oazvWUZjGHP1sUOE6Rf4lDfeZvGj31Z7xyQ=w1920-h883-no?authuser=0)
