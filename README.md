# gitops-homelab

Repositório **GitOps** do meu homelab Kubernetes. O [Argo CD](https://argo-cd.readthedocs.io/) lê este repositório e mantém o cluster igual ao que está descrito aqui: o Git é a fonte da verdade.

> Status: **em construção**. A tabela de aplicações abaixo mostra o que já está migrado.

## Como se encaixa com os outros repositórios

| Repositório | Papel |
|---|---|
| [`devops-infrastructure-as-code`](https://github.com/lotechdevops/devops-infrastructure-as-code) | Cria as VMs (Vagrant) e monta o cluster e os serviços com **Ansible** |
| **`gitops-homelab`** (este) | Descreve o que **roda dentro** do cluster, de forma declarativa, gerenciado pelo **Argo CD** |
| `gitops-cloud` (futuro) | Mesma ideia para um cluster na nuvem |

São dois caminhos para o mesmo resultado: o Ansible instala imperativamente, o Argo CD mantém declarativamente. **Não aplique os dois no mesmo cluster.**

## Estrutura

```text
gitops-homelab/
├── bootstrap/                # values do Argo CD e a Application raiz
├── projects/                 # AppProjects (limites de cada grupo de aplicações)
├── apps/                     # uma Application por aplicação
└── config/                   # values e manifests de cada aplicação
    ├── metallb/
    ├── cert-manager/
    ├── kube-prometheus-stack/
    └── observability/
        ├── alertmanager/
        └── prometheusRules/
```

## Aplicações

| Aplicação | Origem | Status |
|---|---|---|
| MetalLB | chart + `config/metallb` | a migrar |
| cert-manager (Let's Encrypt) | chart + `config/cert-manager` | a migrar |
| kube-prometheus-stack | chart + `config/kube-prometheus-stack` | a migrar |
| Regras de alerta e Alertmanager | `config/observability` | a migrar |

## Princípios

- **Git é a fonte da verdade.** Mudança no cluster = commit (e pull request).
- **Sem segredos em texto puro.** Credenciais entram como `SealedSecret` (cifrados para o meu cluster) ou ficam fora do Git. O `.gitignore` bloqueia os padrões comuns.
- **Versões fixas** de charts e imagens (nada de `latest`).
- **Um dono por recurso:** o que está aqui não é gerenciado por Ansible nem por `kubectl` manual.

## Como subir

1. Cluster pronto (veja o repositório `devops-infrastructure-as-code`).
2. Instalar o Argo CD com os values de `bootstrap/` (Helm, versão fixa).
3. Aplicar a Application raiz (`bootstrap/root-app.yaml`).
4. O Argo CD cria os Projects e as Applications a partir deste repositório.

> Os comandos exatos serão documentados conforme cada etapa for validada no cluster.

## Documentação

Os conceitos e o passo a passo de Argo CD que usei para aprender estão no meu blog: <https://devopsr66.com>.
