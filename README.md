# platform-app-base

Golden path do homelab: a estrutura k8s que qualquer microserviço consome via
Kustomize Remote Component, sem copiar manifests. A plataforma controla
security context, probes, resources e observabilidade; o serviço só parametriza
(nome, hostname, imagem, limites).

## Fragments (Components opt-in)

`components/` fatia o golden path em Components coesos, cada um mapeando uma
decisão real de quem cria o serviço:

| Fragment | Recursos | Quando |
|---|---|---|
| `workload` | Deployment + Service | sempre — o irredutível |
| `metrics` | ServiceMonitor | default-on (golden path injeta observabilidade) |
| `http` | Ingress + Certificate | só se expõe HTTP externo |
| `availability` | HPA + PDB | se quer autoscaling / disruption budget |

Cada fragment carrega **só os replacements dos recursos que ele define** — o
replacement de `hostname` vive no `http`, então não fica órfão num serviço
interno que não expõe HTTP.

### Como consumir

A kustomization do repo de deploy (`gitops-<app>`) compõe os fragments que
precisa e fornece os valores via `app-config` (ConfigMap) + bloco `images`:

```yaml
components:
  - https://github.com/TourinhoM/platform-app-base//components/workload?ref=main
  - https://github.com/TourinhoM/platform-app-base//components/metrics?ref=main
  - https://github.com/TourinhoM/platform-app-base//components/http?ref=main   # opcional

configMapGenerator:
  - name: app-config
    literals:
      - name=meu-servico
      - hostname=meu-servico.local   # só se incluir http

images:
  - name: app-image
    newName: ghcr.io/tourinho-labs/meu-servico
    newTag: latest
```

O scaffolder `new-service-full` (platform-templates) gera essa composição
automaticamente a partir dos toggles `exposeHttp` / `autoscaling`.

## `base/` — bundle legado (frozen)

`base/` é o Component monolítico antigo (todos os 7 recursos juntos). Mantido
**só** para os consumidores que ainda pinam `//base?ref=main` (apps fixture
pré-fragments). Não recebe mudanças. Será removido no cutover, quando os
fixtures forem aposentados. Serviços novos usam `components/*`, não `base/`.

## Lint (CI)

`base/` e `components/` são `kind: Component` — não buildam standalone. O
`lint-k8s` (org-ci-platform) renderiza overlays de teste em `tests/` e roda
kubeconform + kube-linter + polaris sobre o resultado:

| Overlay | Compõe | Valida |
|---|---|---|
| `tests/minimal` | workload + metrics | serviço interno renderiza sem o `http` |
| `tests/full` | os 4 fragments | composição completa exposta |
| `tests/legacy` | `base/` | bundle legado enquanto vivo (removível no cutover) |

Os overlays de `tests/` não são deployados — não há Application apontando pra
eles. Existem só como harness de lint na fonte, uma vez, em vez de relintar a
mesma estrutura em cada `gitops-<app>`.

## Limitações conhecidas

### Hoje, dentro do escopo atual

- **Duplicação temporária** entre `base/` e `components/`: os mesmos manifests
  vivem nos dois durante a transição. Colapsa quando `base/` for removido no
  cutover.
- **`?ref=main` (não tag).** Consumidores seguem o `main` do app-base — uma
  mudança no fragment chega sem o app pinar. Versionamento por tag semântica é
  um passo futuro (decisão #7 do PLATFORM_PLAN).

### Se a stack mudar

- **porta 3000 / probe em `/health` / `/metrics`** são convenção fixa do
  `workload` e `metrics`. Serviço que fuja disso precisa de patch no próprio
  `gitops-<app>` (a base não parametriza porta hoje).
