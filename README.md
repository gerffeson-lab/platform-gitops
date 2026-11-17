# platform-gitops

Estado desejado do cluster, sincronizado pelo Argo CD. O que está aqui é o que roda.

```
bootstrap/
  argocd/        instala o Argo CD (no OpenShift: Subscription do OpenShift GitOps)
  root.yaml      app-of-apps: o Argo CD passa a gerenciar cluster/
cluster/         time de plataforma
  politicas/     ValidatingAdmissionPolicy (nativas, sem Kyverno)
  produtos/      1 arquivo por produto: namespaces, quota, AppProject, ApplicationSet
produtos/        esteira: manifestos renderizados por serviço e ambiente
  <sigla>-<produto>/<serviço>/<ambiente>/manifests.yaml
kind/            cluster local para testar
```

## Quem escreve o quê

| Pasta | Quem escreve | Quando |
|---|---|---|
| `cluster/politicas/` | Time de plataforma, por PR | Regra nova |
| `cluster/produtos/` | Gerado do catálogo do self-service | Produto novo entra no catálogo |
| `produtos/.../<ambiente>/` | Esteira do serviço (render do chart da plataforma) | Cada deploy |

Ninguém cria `Application` à mão: o `ApplicationSet` do produto acha cada pasta
`produtos/<produto>/<serviço>/<ambiente>` e cria a Application no namespace
`<produto>-<ambiente>`. Serviço novo = primeira vez que a esteira escreve a pasta dele.

## Guardrails

| Regra | Onde |
|---|---|
| Container com limits, até 2 CPU / 4Gi | VAP `recursos-por-container` (barra o Deployment, o erro aparece no sync do Argo CD) e `LimitRange` (barra o pod) |
| Imagem do registry da plataforma, com versão fixa | VAP `imagem-confiavel` |
| Sem root, sem privilégios | Pod Security `restricted` no namespace (no OpenShift: SCC `restricted-v2`) |
| Consumo total do produto | `ResourceQuota` por namespace |
| Produto só implanta no que é dele | `AppProject`: só `<produto>-*`, sem recursos de cluster, sem mexer em quota |

As políticas valem para namespaces com o label `plataforma.acme/produto`.

## Testar localmente (kind)

```bash
kind create cluster --config kind/cluster.yaml
kubectl apply -k bootstrap/argocd --server-side
kubectl -n argocd wait --for=condition=Available deploy --all --timeout=300s
kubectl apply -f bootstrap/root.yaml

# console do Argo CD em https://localhost:8443 (usuário admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8443:443
```

Cluster de 1 nó com o Argo CD: cerca de 2–3 GB de RAM. `kind delete cluster --name plataforma` para liberar.

## No OpenShift do banco

| Aqui | No banco |
|---|---|
| Argo CD da comunidade (`bootstrap/argocd`) | Operator OpenShift GitOps |
| Pod Security `restricted` | SCC `restricted-v2` (já é o padrão) |
| `Service` + port-forward | `Route` gerada pelo chart da plataforma |
| `produtos/` neste repo | Repo GitOps por produto (a decidir: por produto ou por sigla) |
| 1 cluster, namespace por ambiente | Clusters por ambiente, distribuídos pelo ACM (se houver) |
