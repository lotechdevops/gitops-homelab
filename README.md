# gitops-homelab

Repositório **GitOps** do meu homelab Kubernetes. O [Argo CD](https://argo-cd.readthedocs.io/) lê este repositório e mantém o cluster igual ao que está descrito aqui: o Git é a fonte da verdade.

> Status: **infraestrutura migrada**. Todas as aplicações de infraestrutura da tabela abaixo já são gerenciadas pelo Argo CD, adotadas sem recriar nada do que já rodava.

## Como se encaixa com os outros repositórios

| Repositório | Papel |
|---|---|
| [`devops-infrastructure-as-code`](https://github.com/lotechdevops/devops-infrastructure-as-code) | Cria as VMs (Vagrant) e monta o cluster com **Ansible** |
| **`gitops-homelab`** (este) | Descreve o que **roda dentro** do cluster, de forma declarativa, gerenciado pelo **Argo CD** |
| `gitops-cloud` (futuro) | Mesma ideia para um cluster na nuvem |

O Ansible provisiona o cluster; o Argo CD entrega e mantém as aplicações. **Não aplique os dois no mesmo recurso.**

## Estrutura

```text
gitops-homelab/
├── projects/                  # AppProjects (limites de cada grupo de aplicações)
│   └── infra.yaml
├── apps/                      # uma Application por aplicação
│   ├── nfs-provisioner.yaml
│   ├── metallb.yaml
│   ├── traefik.yaml
│   ├── cert-manager.yaml
│   ├── cert-manager-issuers.yaml
│   ├── sealed-secrets.yaml
│   ├── kube-prometheus-stack.yaml
│   └── observability-config.yaml
└── config/                    # values e manifests de cada aplicação
    ├── nfs-provisioner/       # values do chart
    ├── metallb/               # Kustomize: manifesto oficial (versão fixa) + pool e L2
    ├── traefik/               # values do chart
    ├── cert-manager/          # values do chart
    │   └── issuers/           # ClusterIssuers (Let's Encrypt) + token como SealedSecret
    ├── sealed-secrets/        # values do chart
    ├── kube-prometheus-stack/ # values do chart
    └── observability/         # o que é meu e fica fora do chart
        ├── prometheusRules/   # regras de alerta (CPU, disco, memória)
        ├── alertmanager/      # AlertmanagerConfig (Telegram) + token como SealedSecret
        └── grafana/           # senha de admin como SealedSecret
```

## Aplicações

| Aplicação | Origem | Versão | Sincronização |
|---|---|---|---|
| nfs-provisioner | chart | 4.0.18 | automática, com `prune` |
| MetalLB | Kustomize (manifesto oficial) | v0.14.3 | automática (`selfHeal`) |
| Traefik | chart | 42.2.0 | automática (`selfHeal`) |
| cert-manager | chart | v1.21.3 | automática (`selfHeal`) |
| cert-manager-issuers | pasta `config/cert-manager/issuers` | n/a | automática (`selfHeal`) |
| Sealed Secrets | chart | 2.22.0 | automática (`selfHeal`) |
| kube-prometheus-stack | chart | 87.19.0 | automática (`selfHeal`) |
| observability-config | pasta `config/observability` | n/a | automática, com `prune` |

**Regra de `prune`:** fica **desligado** onde há CRDs, volumes ou o que sustenta outros recursos (certificados, rede, storage, segredos), porque uma remoção por engano apagaria dados ou todos os recursos dependentes. Fica ligado só para recursos simples e recriáveis (regras de alerta, o provisionador NFS).

## Princípios

- **Git é a fonte da verdade.** Mudança no cluster = commit (e pull request).
- **Sem segredos em texto puro.** Credenciais entram como `SealedSecret` (cifrados para o meu cluster) ou ficam fora do Git. O `.gitignore` bloqueia os padrões comuns.
- **Versões fixas** de charts e manifestos (nada de `latest`).
- **Um dono por recurso:** o que está aqui não é gerenciado por Ansible nem por `kubectl` manual.
- **Adoção sem recriar:** cada aplicação que já existia foi migrada comparando o `diff` com o cluster (só podia mudar a marca do Argo CD) antes de sincronizar.

## Segredos

| Segredo | Onde fica | Quem usa |
|---|---|---|
| Token da API do Cloudflare | `config/cert-manager/issuers/` (SealedSecret) | ClusterIssuers (desafio DNS01) |
| Token do bot do Telegram | `config/observability/alertmanager/` (SealedSecret) | Alertmanager |
| Senha do admin do Grafana | `config/observability/grafana/` (SealedSecret) | Grafana |

Os SealedSecrets só podem ser abertos pela **chave privada do controller**, que **não está neste repositório**: o backup dela fica num gerenciador de senhas, fora do cluster e do Git. Sem ela, os SealedSecrets não abrem num cluster novo.

Fora do Git de propósito: a credencial SSH que o Argo CD usa para ler este repositório (é ela que dá acesso aos SealedSecrets).

## Como subir (cluster novo)

1. Cluster pronto (veja o repositório `devops-infrastructure-as-code`).
2. **Restaurar a chave do Sealed Secrets** no namespace `kube-system`, antes de o controller existir (assim ele a reconhece e os SealedSecrets do Git abrem).
3. Instalar o Argo CD (Helm, versão fixa). <!-- AJUSTAR: documentar os values usados -->
4. Conectar este repositório ao Argo CD (`argocd repo add` com a credencial SSH).
5. Aplicar os Projects e as Applications:

   ```bash
   kubectl apply -f projects/
   kubectl apply -f apps/
   ```

6. O Argo CD instala tudo e os Secrets são abertos pelo controller.

## Próximos passos

- [ ] **Application raiz** (app-of-apps): um único `kubectl apply` para subir tudo.
- [ ] Adotar o **`cloudflared`** (túnel do Cloudflare), com o token como SealedSecret.
- [ ] O Argo CD **gerenciando a si mesmo**.
- [ ] Atualizar o MetalLB (hoje v0.14.3) em commit separado.
- [ ] `gitops-cloud`: o mesmo padrão em um cluster na nuvem.

## Documentação

Os conceitos e o passo a passo de Argo CD que usei para aprender estão no meu blog: <https://devopsr66.com>.