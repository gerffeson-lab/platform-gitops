# platform-gitops

Manifestos renderizados dos serviços, escritos **só pela esteira** e sincronizados pelo
Argo CD. O que está aqui é o que roda. Não edite à mão: o próximo deploy sobrescreve.

```
manifests/
  <sigla>-<produto>/
    <serviço>/          um diretório por repositório de serviço
      deployment.yaml
      service.yaml
      ...
```

| Branch | Ambiente | Quem escreve |
|---|---|---|
| `develop` | dev | esteira, em push na `develop` do serviço |
| `release` | hml | esteira, em push numa `release/*` do serviço |
| `main` | prd | esteira, em push na `main` do serviço |

As branches nunca são mescladas entre si: cada uma é o estado de um ambiente. Promover é
fazer o merge no repositório do serviço (GitFlow); a esteira dele grava aqui.

Cada deploy substitui a pasta inteira do serviço (o que saiu do chart some daqui e o
Argo CD remove do cluster). O commit leva o serviço, a imagem e o link da execução.

Quem lê estas pastas, cria namespaces e define permissões é a configuração do cluster, em
[platform-cluster-config](https://github.com/gerffeson-lab/platform-cluster-config).
Os manifestos saem do chart da plataforma, em
[platform-helm-charts](https://github.com/gerffeson-lab/platform-helm-charts), com os
values de `deploy/` do serviço.
