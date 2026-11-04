# Lab: Argo CD a gerir o Crossplane por GitOps, com sync waves e floci (k3d)

Um cluster k3d onde o **Argo CD gere a plataforma Crossplane inteira a partir do Git**: a instalação do Crossplane, o provider, as functions, a ProviderConfig, o XRD, a Composition e os XRs. Cada camada é uma `Application`, ordenadas com **sync waves**. No fim há uma [variante opcional com `ApplicationSet`](#variante-applicationset-nos-consumers). O que o Crossplane provisiona são **buckets S3 no floci** (a AWS local). Os pacotes do Crossplane (provider e functions) vêm de um **registry OCI**, que serve só para distribuir pacotes.

Baseado na doc oficial:

- [Argo CD → Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Argo CD → Cluster Bootstrapping (App of Apps)](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [Argo CD → Manage Argo CD Using Argo CD](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#manage-argo-cd-using-argo-cd), [Chart Helm `argo-cd` (argo-helm)](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd), [Multiple sources (valores Helm vindos do Git)](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/#helm-value-files-from-external-git-repository)
- [Argo CD → Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
- [Argo CD → ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/), [Git generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/), [Progressive Syncs](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Progressive-Syncs/)
- [Argo CD → Sync Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/), [Health → Argo CD App](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#argocd-app), [Directory apps](https://argo-cd.readthedocs.io/en/stable/user-guide/directory/)
- [Crossplane → Configuring Crossplane with Argo CD](https://docs.crossplane.io/latest/guides/crossplane-with-argo-cd/)
- [Crossplane → Get Started With Composition](https://docs.crossplane.io/latest/get-started/get-started-with-composition/), [Compositions](https://docs.crossplane.io/latest/composition/compositions/), [Providers](https://docs.crossplane.io/latest/packages/providers/), [Functions](https://docs.crossplane.io/latest/packages/functions/)
- [Crossplane → Configurations](https://docs.crossplane.io/latest/packages/configurations/), [Image Configs](https://docs.crossplane.io/latest/packages/image-configs/)

E nos teus labs anteriores: `Lab de duas compositions.md` (ligação ao floci, `ClusterProviderConfig`, Composition de buckets).

> [!tip] Conceito-chave
> Fazes dois `kubectl apply`: um para instalar o Argo e outro para o `root`. A partir daí **a fonte da verdade é o Git**. O Argo põe o cluster igual ao repo e o Crossplane põe a "infraestrutura" (os buckets) igual aos XRs.

> [!warning] Estado deste guia
> Os manifestos foram validados offline (schemas dos CRDs, `crossplane composition render`, `helm template`). **O fluxo completo ainda não foi corrido de ponta a ponta.** Vê a secção [O que foi testado](#o-que-foi-testado) e os avisos "por testar" ao longo do guia.

---

# Como interpretei o enunciado

O enunciado deixa algumas coisas em aberto. Estas são as escolhas deste guia:

| Ponto do enunciado | Neste lab |
| --- | --- |
| Repo Git privado | O Argo recebe uma credencial: uma **deploy key SSH** só de leitura. Ver [Repo privado](#repo-privado-dar-acesso-ao-argo) |
| Registry OCI "ainda não definido" | Os pacotes vêm do `xpkg.crossplane.io` e do `xpkg.upbound.io`, registries OCI públicos. Para trocar de registry, vê a secção [Trocar de registry](#trocar-de-registry) |
| O registry não é alvo de provisionamento | O que se provisiona são **buckets S3 no floci**, a AWS local que já usaste noutros labs. Corre fora do cluster, no teu PC |
| Providers + ProviderConfigs | `provider-aws-s3` + uma `ClusterProviderConfig` que aponta para o floci, com as chaves `test`/`test` |
| Claims | No Crossplane v2 os XRs são namespaced e substituem os claims. Os claims só existem para XRDs `scope: LegacyCluster` (compatibilidade com a v1). Aqui usa-se um **XR namespaced** (`StorageBucket`) |
| Várias `Application`s (uma por camada) | Uma `Application` por camada, ordenadas com **sync waves**. Variante opcional: um [ApplicationSet](#variante-applicationset-nos-consumers) para gerir várias pastas de `consumers/` |
| Configuration packages | Ficam como passo opcional no fim (`crossplane xpkg build/push`) |

---

# A ideia

```
Git (repo público)
 │
 ▼
root                  (aplicado à mão, uma vez; lê a pasta apps/)
 ├─ argocd            wave 0   argocd/                     o próprio Argo (chart Helm)
 ├─ crossplane        wave 1   chart Helm crossplane 2.4.3
 ├─ packages          wave 2   platform/packages/          Provider + Functions (do registry OCI)
 ├─ provider-configs  wave 3   platform/provider-configs/  ClusterProviderConfig (floci)
 ├─ apis              wave 3   apis/                       XRD + Composition
 └─ consumers         wave 4   consumers/                  XRs (StorageBucket)
                                   │
                                   ▼  Crossplane compõe
                               Bucket (S3)  →  bucket no floci
```

As camadas do enunciado:

| Camada | Apps |
| --- | --- |
| Platform | `crossplane`, `packages`, `provider-configs` |
| API | `apis` |
| Consumer | `consumers` |

## Sync waves: a ordem

Uma sync wave é um **número numa annotation** (`argocd.argoproj.io/sync-wave: "2"`). O Argo aplica primeiro os objetos de wave mais baixa e só avança para a seguinte quando os da wave atual estão `Healthy`. Aqui decidem a ordem de instalação:

| Wave | App | Precisa de |
| --- | --- | --- |
| 0 | `argocd` | nada (é instalado pelo bootstrap) |
| 1 | `crossplane` | nada. Cria os CRDs `Provider`, `Function`, `CompositeResourceDefinition`, `Composition` |
| 2 | `packages` | os CRDs do Crossplane |
| 3 | `provider-configs` | o CRD `ClusterProviderConfig`, que **só existe depois de o provider estar instalado** |
| 3 | `apis` | os CRDs do Crossplane e a function instalada |
| 4 | `consumers` | o CRD `StorageBucket`, que é criado pelo XRD |

O `root` só passa à wave seguinte quando as apps da wave atual estão `Healthy`.

> [!note] E o ApplicationSet?
> Este guia usa só waves, com uma `Application` por camada. O ApplicationSet faz outra coisa: gera `Application`s a partir de pastas. Há uma [variante opcional](#variante-applicationset-nos-consumers) no fim, para veres a diferença.

---

# Fase 0: cluster e floci

Pré-requisitos: o `Setup WSL do zero.md` feito, **incluindo os sysctls**. Sem eles o `k3d cluster create` fica pendurado. Também precisas de `kubectl`, `git`, `helm`, `aws` e da CLI `crossplane` (para diagnóstico e, opcionalmente, para [desenvolver a API com `crossplane project`](#opcional-desenvolver-a-api-com-crossplane-project)). O lab corre sem a CLI.

```bash
k3d cluster create xp-argo --k3s-arg "--disable=traefik@server:0"

kubectl config current-context    # k3d-xp-argo
kubectl get nodes                 # Ready
```

> [!note] Sem sudo para os sysctls?
> Cria o cluster com `--k3s-arg "--kubelet-arg=feature-gates=KubeletInUserNamespace=true@server:0"`, que também resolve.

## Arrancar o floci

O floci corre no podman do root, fora do cluster:

```bash
sudo podman start floci floci-aws-ui

alias awsf='aws --profile floci --endpoint-url http://localhost:4566'
awsf s3 ls
```

Se o `awsf s3 ls` responde (mesmo sem listar nada), o floci está de pé. O profile `floci` está no `AMBIENTE.md`.

De **dentro do cluster** o floci está em `169.254.1.2.nip.io:4566` (o teu PC visto de dentro dos containers; o `.nip.io` resolve sempre para o IP que vem antes dele). Do PC é `localhost:4566`. Confirma que o cluster lá chega:

```bash
kubectl run floci-check --rm -i --restart=Never --image=docker.io/curlimages/curl -n default \
  -- curl -s -o /dev/null -w '%{http_code}\n' http://169.254.1.2.nip.io:4566/
```

Qualquer código HTTP serve. Um timeout ou `Could not resolve host` quer dizer que o cluster não chega ao floci.

> [!warning] Por testar neste cluster
> Os labs anteriores usaram este endereço num cluster `kind`. Aqui o cluster é um k3d. Corre o teste acima antes de continuar.

## DNS do cluster (só se precisares)

Se vires `no such host` nos eventos do provider ou da function, aplica este bloco:

```bash
kubectl get cm coredns -n kube-system -o yaml \
  | sed 's#forward . /etc/resolv.conf#forward . 1.1.1.1 8.8.8.8#' \
  | kubectl replace -f -

kubectl rollout restart deploy/coredns -n kube-system

kubectl get cm coredns -n kube-system -o jsonpath='{.data.Corefile}' | grep forward
```

Esperado: `forward . 1.1.1.1 8.8.8.8`.

O que faz: o `sed` troca no Corefile o `forward . /etc/resolv.conf` (o CoreDNS pergunta ao DNS do podman) por `forward . 1.1.1.1 8.8.8.8` (pergunta diretamente à Cloudflare e à Google). O `replace` grava a ConfigMap, o `rollout restart` reinicia o CoreDNS, e o último comando mostra a linha para confirmares.

> [!note] É um workaround, não um passo garantido
> O problema descrito (programas em Go a verem zero respostas porque a resposta em cache vem fora da secção ANSWER) vem de um lab anterior. Fiz 30 queries ao CoreDNS deste cluster, sem o fix, e todas tiveram resposta, por isso **não consegui reproduzi-lo**. Aplica-o só se aparecer o erro.
>
> O k3s repõe o Corefile original quando o cluster reinicia. Depois de um `k3d cluster stop/start` tens de repetir este bloco.

---

# Fase 1: o repo Git

## Criar o repo

No GitHub: **New repository** → `gitops-crossplane-lab`, **Private** (ou Public), sem README.

> [!note] Privado ou público?
> Um repo **público** é lido pelo Argo sem credenciais, com o URL HTTPS. Um repo **privado** obriga a dar uma credencial ao Argo. Este guia usa o URL SSH e uma *deploy key*, que funciona nos dois casos. O passo a passo está em [Repo privado](#repo-privado-dar-acesso-ao-argo).

> [!warning] O `repoURL` não é o teu `git remote`
> O `git remote` pode usar um alias do teu `~/.ssh/config` (por exemplo `git@github-vln:...`). Esse alias só existe no teu PC: o Argo corre dentro do cluster e não o conhece. Nos manifestos usa sempre o host verdadeiro: `git@github.com:<o-teu-user>/gitops-crossplane-lab.git`.

```bash
git clone git@github.com:<o-teu-user>/gitops-crossplane-lab.git
cd gitops-crossplane-lab

mkdir -p bootstrap argocd apps platform/packages platform/provider-configs apis/bucket consumers/default
```

Se nunca fizeste commits nesta máquina, configura antes a tua identidade de Git (`git config --global user.name` e `user.email`).

## Estrutura final

```
gitops-crossplane-lab/
├── bootstrap/
│   └── root.yaml                    app of apps (aplicado à mão, uma vez)
├── argocd/                          o próprio Argo CD
│   └── values.yaml                  valores do chart Helm do Argo
├── apps/                            uma Application por camada
│   ├── argocd.yaml
│   ├── crossplane.yaml
│   ├── packages.yaml
│   ├── provider-configs.yaml
│   ├── apis.yaml
│   └── consumers.yaml
├── platform/                        camada de plataforma
│   ├── packages/
│   │   ├── provider-aws-s3.yaml
│   │   ├── function-kcl.yaml
│   │   └── function-auto-ready.yaml
│   └── provider-configs/
│       └── aws.yaml
├── apis/                            camada de API (uma pasta por API)
│   └── bucket/
│       ├── definition.yaml
│       └── composition.yaml
└── consumers/                       camada de consumo (uma pasta por namespace)
    └── default/
        └── my-bucket.yaml
```

Neste lab o único segredo (as chaves `test`/`test` do floci) **não vai para o Git**: crias a Secret à mão no bootstrap.

> [!note] De onde vêm os ficheiros
> Cada ficheiro começa com comentários `SOURCE:` (a doc ou o exemplo de onde vem, com link) e `ADAPTED:` (o que foi mudado). Não há nenhum gerador oficial para esta estrutura: o layout é uma escolha deste guia, montada a partir de exemplos oficiais.

## `argocd/`: o próprio Argo CD

```bash
cat > argocd/values.yaml <<'EOF'
# SOURCE: Argo CD Helm chart 'argo-cd' (argo-helm), values reference:
#   https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd
#   Health check (the Lua script is the one from the Argo CD docs):
#   https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#argocd-app
# ADAPTED: the chart defaults are kept; the only setting changed is the health check below.
# WHY: Argo CD 3.x has no built-in health check for Application; without it the sync waves
#   of the 'root' app do not wait for each other.
# USED TWICE: `helm upgrade --install ... -f argocd/values.yaml` in the bootstrap, and then the
#   'argocd' Application reads this same file from Git (apps/argocd.yaml, 'ref: values').
configs:
  cm:
    resource.customizations.health.argoproj.io_Application: |
      hs = {}
      hs.status = "Progressing"
      hs.message = ""
      if obj.status ~= nil then
        if obj.status.health ~= nil then
          hs.status = obj.status.health.status
          if obj.status.health.message ~= nil then
            hs.message = obj.status.health.message
          end
        end
      end
      return hs
EOF
```

O Argo CD é instalado com o **chart Helm oficial da comunidade** (`argo-cd`, do repo `argo-helm`). Este ficheiro são os *values* do chart: só mudámos uma coisa em relação aos defaults.

- O chart `10.11.6` instala o Argo CD `v3.5.6`. A versão fica fixa, por isso um upgrade do Argo é só um commit.
- O ficheiro é usado **duas vezes**. No bootstrap passas tu com `helm upgrade --install ... -f argocd/values.yaml`, e depois a app `argocd` lê este mesmo ficheiro do Git (segunda fonte, `ref: values`). É assim que o Argo passa a gerir-se a si próprio com os mesmos valores.
- A única mudança é o health check de `Application` (`configs.cm`), copiado da doc ([Health → Argo CD App](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#argocd-app)). O Argo 3.5.6 não traz esse check de origem. **Sem ele, as waves do `root` não esperam por nada.**
- O chart já marca os CRDs com `ServerSideApply=true` quando é o Argo a instalá-los (está no README do chart). Os CRDs do Argo são grandes demais para o apply normal.

> [!note] Porquê Helm e não Kustomize?
> A doc "Manage Argo CD Using Argo CD" sugere Kustomize sobre o `install.yaml` oficial. Aqui usa-se Helm por escolha do lab: a configuração fica num `values.yaml` em vez de patches, e é o mesmo método que o Crossplane usa. Os dois funcionam. Ver o FAQ.

## `bootstrap/root.yaml`: a app das apps

```bash
cat > bootstrap/root.yaml <<'EOF'
# SOURCE: Argo CD, App of Apps pattern
#   https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern-alternative
# ADAPTED: path 'apps', targetRevision HEAD, automated sync with prune + selfHeal.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: REPO_URL
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

- `path: apps`: o `root` aplica tudo o que está nessa pasta (as 6 `Application`s).
- `automated` sincroniza sozinho quando o Git muda. `prune` apaga o que saiu do Git e `selfHeal` desfaz o que alguém mudou à mão no cluster.
- `REPO_URL` é substituído no fim por um `sed`.

## `apps/`: uma `Application` por camada

```bash
cat > apps/argocd.yaml <<'EOF'
# SOURCE: Argo CD Helm chart 'argo-cd' (argo-helm): https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd
#   Helm values kept in Git (multiple sources, 'ref'):
#   https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/#helm-value-files-from-external-git-repository
#   Argo CD managing itself: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#manage-argo-cd-using-argo-cd
#   Sync waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
# ADAPTED: wave 0; chart pinned to 10.11.6, which installs Argo CD v3.5.6; the values come from this
#   repo through the second source ('ref: values' -> $values); releaseName matches the name used in
#   the bootstrap (`helm upgrade --install argocd ...`) so Argo takes over the same objects;
#   ServerSideApply=true as the self-management docs require.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  sources:
  - repoURL: https://argoproj.github.io/argo-helm
    chart: argo-cd
    targetRevision: 10.11.6
    helm:
      releaseName: argocd
      valueFiles:
      - $values/argocd/values.yaml
  - repoURL: REPO_URL
    targetRevision: HEAD
    ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - ServerSideApply=true
EOF

cat > apps/crossplane.yaml <<'EOF'
# SOURCE: official Crossplane Helm chart (repo https://charts.crossplane.io/stable, version 2.4.3)
#   Crossplane + Argo CD guide: https://docs.crossplane.io/latest/guides/crossplane-with-argo-cd/
#   Sync waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
# ADAPTED: wave 1; CreateNamespace=true; ServerSideApply=true kept because the lab was tested that way.
#   (The chart ships no CRDs: Crossplane's init container creates them when it starts.)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    targetRevision: 2.4.3
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
EOF

cat > apps/packages.yaml <<'EOF'
# SOURCE: App of Apps pattern (https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
#   Sync waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
#   directory.recurse: https://argo-cd.readthedocs.io/en/stable/user-guide/directory/
# ADAPTED: wave 2 (needs the Crossplane CRDs); recurse so subfolders are read.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: packages
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: REPO_URL
    targetRevision: HEAD
    path: platform/packages
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

cat > apps/provider-configs.yaml <<'EOF'
# SOURCE: App of Apps pattern (https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
#   Sync waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
#   directory.recurse: https://argo-cd.readthedocs.io/en/stable/user-guide/directory/
# ADAPTED: wave 3 (the ClusterProviderConfig CRD only exists once the provider is installed).
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: provider-configs
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: REPO_URL
    targetRevision: HEAD
    path: platform/provider-configs
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

cat > apps/apis.yaml <<'EOF'
# SOURCE: App of Apps pattern (https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
#   Sync waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
#   directory.recurse: https://argo-cd.readthedocs.io/en/stable/user-guide/directory/
# ADAPTED: wave 3 (XRD + Composition need the Crossplane CRDs and the function); 'apis' with recurse covers every API folder, e.g. apis/bucket/.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apis
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: REPO_URL
    targetRevision: HEAD
    path: apis
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

cat > apps/consumers.yaml <<'EOF'
# SOURCE: App of Apps pattern (https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
#   Sync waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
#   directory.recurse: https://argo-cd.readthedocs.io/en/stable/user-guide/directory/
# ADAPTED: wave 4 (the StorageBucket CRD is created by the XRD, so this goes last); destination
#   namespace is default, which is where consumers/default/my-bucket.yaml lives.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: consumers
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "4"
spec:
  project: default
  source:
    repoURL: REPO_URL
    targetRevision: HEAD
    path: consumers
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

O que importa:

- `sync-wave` define a ordem da tabela da secção "A ideia". Só tem efeito dentro do `root`, que é quem cria estas apps.
- `argocd`: **é o Argo a gerir-se a ele próprio**. Tem duas fontes (`sources`): o chart `argo-cd` no repo Helm da comunidade, e o teu repo Git só para ler o `values.yaml` (`ref: values`, usado como `$values/argocd/values.yaml`). O `releaseName: argocd` é o mesmo nome do `helm upgrade --install` do bootstrap. O `ServerSideApply=true` é obrigatório (doc do "Manage Argo CD Using Argo CD").
- `crossplane`: a fonte é o chart Helm oficial e não o repo. O Argo faz `helm template` e aplica o resultado. O chart não traz CRDs: é o init container do Crossplane que os cria quando arranca.
- `directory.recurse: true`: **sem isto o Argo só lê os ficheiros que estão na raiz do `path`** e ignora as subpastas (`apis/bucket/`, `consumers/default/`). A app aparece `Synced/Healthy`, mas vazia.

## `platform/packages/`: provider e functions do registry OCI

```bash
cat > platform/packages/provider-aws-s3.yaml <<'EOF'
# SOURCE: Crossplane, 'Providers' (installing a provider package)
#   https://docs.crossplane.io/latest/packages/providers/
# ADAPTED: package xpkg.upbound.io/upbound/provider-aws-s3:v2.9.3, the same one used in the
#   'Lab de duas compositions'. It installs the S3 Bucket managed resources; the
#   provider-family-aws package comes in automatically as a dependency.
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v2.9.3
EOF

cat > platform/packages/function-kcl.yaml <<'EOF'
# SOURCE: Crossplane, 'Get Started With Composition' (installing function-kcl)
#   https://docs.crossplane.io/latest/get-started/get-started-with-composition/
#   Same manifest as https://docs.crossplane.io/latest/manifests/get-started/composition/fn-kcl.yaml
# ADAPTED: simpler object name (function-kcl). Version v0.11.2 is the one the Crossplane docs use.
# WHY: runs the KCL code written inside the Composition (apis/bucket/composition.yaml).
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-kcl
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-kcl:v0.11.2
EOF

cat > platform/packages/function-auto-ready.yaml <<'EOF'
# SOURCE: Crossplane, 'Get Started With Composition' (function-auto-ready)
#   https://docs.crossplane.io/latest/get-started/get-started-with-composition/
# ADAPTED: version v0.7.1, the same one used in the 'Lab de duas compositions'.
# WHY: marks the XR as Ready once all its composed resources are ready.
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.7.1
EOF
```

- **Quem descarrega os pacotes é o Crossplane, não o Argo.** O Argo só aplica os objetos `Provider` e `Function`. O Crossplane lê o `spec.package`, faz pull do registry OCI e instala o pacote.
- O `provider-aws-s3` traz os recursos de S3 (`Bucket`, etc.). O `provider-family-aws`, que traz a `ClusterProviderConfig`, vem sozinho como dependência.
- `function-kcl` executa o código KCL que está dentro da Composition (cria os recursos a partir do XR) e `function-auto-ready` marca o XR como `READY` quando os recursos compostos estão prontos.
- O provider AWS vem do `xpkg.upbound.io` e as functions do `xpkg.crossplane.io`.

## `platform/provider-configs/`

```bash
cat > platform/provider-configs/aws.yaml <<'EOF'
# SOURCE: this user's own 'Lab de duas compositions' (tested against floci), file
#   '01-providerconfig.yaml'; the fields are defined by the provider's CRD (validated below).
# ADAPTED: endpoint points at floci (the local AWS) running on your PC, seen from inside the
#   cluster as 169.254.1.2.nip.io. The Secret 'aws-secret' is NOT in Git: you create it by hand
#   during the bootstrap (it only holds the dummy keys 'test'/'test').
# NEVER use real AWS credentials here: a wrong endpoint would send requests to real AWS.
apiVersion: aws.m.upbound.io/v1beta1
kind: ClusterProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
  endpoint:
    hostnameImmutable: true
    services: [s3, s3control, sts]
    url:
      type: Static
      static: http://169.254.1.2.nip.io:4566
  skip_credentials_validation: true
  skip_metadata_api_check: true
  skip_requesting_account_id: true
  s3_use_path_style: true
EOF
```

- `aws.m.upbound.io` é a API namespaced (v2) do provider. O `.m.` quer dizer *namespaced managed resources*.
- `ClusterProviderConfig` serve para todos os namespaces. A alternativa, `ProviderConfig`, é namespaced e obriga a ter uma por namespace.
- `endpoint.url.static`: o floci visto de dentro do cluster. Os `skip_*` impedem o provider de validar a conta ou as credenciais na AWS verdadeira, e `s3_use_path_style` usa `host/bucket` em vez de `bucket.host`.
- `secretRef`: a Secret `aws-secret`, que crias no bootstrap.

> [!danger] Nunca uses credenciais AWS reais
> Se o endpoint estiver mal configurado, os pedidos podem chegar à AWS verdadeira.

## `apis/bucket/`: o XRD e a Composition

```bash
cat > apis/bucket/definition.yaml <<'EOF'
# SOURCE: Crossplane, 'Get Started With Composition' (v2 XRD, scope Namespaced)
#   https://docs.crossplane.io/latest/get-started/get-started-with-composition/
# ADAPTED: a new API 'StorageBucket' (platform.example.org/v1alpha1) with one field, 'region'.
#   Whoever asks for a bucket only writes the region; how the bucket is made is the platform's job.
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: storagebuckets.platform.example.org
spec:
  scope: Namespaced
  group: platform.example.org
  names:
    kind: StorageBucket
    plural: storagebuckets
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                description: AWS region of the bucket.
                type: string
                default: us-east-1
EOF

cat > apis/bucket/composition.yaml <<'EOF'
# SOURCE: Crossplane, 'Get Started With Composition' (KCL variant, inline KCL source)
#   https://docs.crossplane.io/latest/get-started/get-started-with-composition/
#   Same structure as https://docs.crossplane.io/latest/manifests/get-started/composition/composition-kcl.yaml
#   Compositions: https://docs.crossplane.io/latest/composition/compositions/
# ADAPTED: the official example builds a Deployment and a Service; here the KCL builds one S3 Bucket
#   (s3.aws.m.upbound.io, from the 'Lab de duas compositions'). The XR name becomes the bucket name in
#   floci (crossplane.io/external-name), and the XR's 'region' becomes the Bucket's region.
# HOW IT READS: `option("params").oxr` is the XR the user created. `items` is the list of resources
#   Crossplane must create. The 'krm.kcl.dev/composition-resource-name' annotation is the stable name of
#   each composed resource. No metadata.namespace is set: the Bucket inherits the XR's namespace.
#   The second step (function-auto-ready) marks the XR Ready when the Bucket is.
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: storagebucket-aws
spec:
  compositeTypeRef:
    apiVersion: platform.example.org/v1alpha1
    kind: StorageBucket
  mode: Pipeline
  pipeline:
  - step: kcl
    functionRef:
      name: function-kcl
    input:
      apiVersion: krm.kcl.dev/v1alpha1
      kind: KCLInput
      spec:
        source: |
          oxr = option("params").oxr

          _bucket = {
            apiVersion = "s3.aws.m.upbound.io/v1beta1"
            kind = "Bucket"
            metadata.annotations = {
              "krm.kcl.dev/composition-resource-name" = "bucket"
              "crossplane.io/external-name" = oxr.metadata.name
            }
            spec = {
              providerConfigRef = {kind = "ClusterProviderConfig", name = "default"}
              forProvider = {
                region = oxr.spec.region
                tags = {"managed-by" = "argocd"}
              }
            }
          }

          items = [_bucket]
  - step: auto-ready
    functionRef:
      name: function-auto-ready
EOF
```

- O XRD cria uma API nova, `StorageBucket` (`platform.example.org/v1alpha1`), **namespaced** (`scope: Namespaced`). Quem a usa só escreve a `region`.
- A Composition cria um `Bucket` S3 (`s3.aws.m.upbound.io`). O `metadata.name` do XR passa a ser o `external-name`, por isso o bucket no floci tem o nome do XR.
- `mode: Pipeline` é o único modo no Crossplane v2. A composição é feita por functions: primeiro a `kcl`, depois a `auto-ready`.
- O código **KCL** está escrito dentro da própria Composition (`input.spec.source`), sem build nem registry. É o mesmo formato do exemplo oficial do "Get Started". Lê-se assim:
  - `option("params").oxr` é o XR que o utilizador criou (o `my-bucket`).
  - `items = [_bucket]` é a lista de recursos que o Crossplane tem de criar. Aqui, um `Bucket`.
  - `krm.kcl.dev/composition-resource-name` é o nome estável do recurso composto. O Crossplane usa-o para saber que o `Bucket` de agora é o mesmo do reconcile anterior.
  - `crossplane.io/external-name = oxr.metadata.name` faz o bucket no floci ter o nome do XR.
  - `region = oxr.spec.region` copia a região do pedido para o `Bucket`.
  - Não há `metadata.namespace`: o `Bucket` herda o namespace do XR.
- A function `auto-ready` marca o `Bucket` e o XR como prontos quando o `Bucket` tem `Ready=True`.
- A tag `managed-by: argocd` só serve para se ver no floci quem criou o bucket.

> [!note] Porque KCL e não patch-and-transform?
> O KCL é uma linguagem pequena para descrever recursos, e a Crossplane lista-a como boa opção quando a composição tem lógica dinâmica. Aqui o código é mínimo (copiar dois campos), mas já é código, e a Composition fica um único ficheiro. Confirmei que gera um `Bucket` **idêntico** ao da versão anterior com `function-patch-and-transform`. No teu `Crossplane v2.md` o KCL vive numa função dentro de um project (`functions/.../main.k`); aqui está inline, o que dispensa o `build` e o `push` do pacote.

## `consumers/default/`: o XR

```bash
cat > consumers/default/my-bucket.yaml <<'EOF'
# SOURCE: this lab's own example: a 'StorageBucket' XR that satisfies the XRD in apis/bucket/definition.yaml.
#   XR format: https://docs.crossplane.io/latest/composition/composite-resources/
# WHAT HAPPENS: Crossplane picks the only compatible Composition and creates an S3 Bucket in floci.
apiVersion: platform.example.org/v1alpha1
kind: StorageBucket
metadata:
  name: my-bucket
  namespace: default
spec:
  region: us-east-1
EOF
```

## URL do repo e push

```bash
REPO=git@github.com:<o-teu-user>/gitops-crossplane-lab.git
sed -i "s#REPO_URL#$REPO#" bootstrap/root.yaml apps/*.yaml

grep -rn REPO_URL bootstrap apps || echo "ok: nenhum REPO_URL por substituir"
grep -h repoURL bootstrap/root.yaml apps/*.yaml | sort | uniq -c
```

Esperado: o primeiro comando diz `ok`. No segundo, 6 linhas com o teu repo (as 5 apps `root`, `packages`, `provider-configs`, `apis` e `consumers`, mais a segunda fonte da `argocd`, que só lê os valores) e 2 de charts: `https://charts.crossplane.io/stable` e `https://argoproj.github.io/argo-helm`. O `uniq -c` pode mostrá-las em grupos separados, porque a indentação é diferente.

> [!note] Repo público?
> Se for público, podes usar `REPO=https://github.com/<o-teu-user>/gitops-crossplane-lab.git` e saltar a secção seguinte.

```bash
git add -A
git commit -m "argocd + crossplane + floci"
git push
```

---

# Repo privado: dar acesso ao Argo

O Argo lê o repo do Git para saber o que aplicar. Se o repo é privado, tem de ter uma **credencial**. Sem ela, as apps ficam `Unknown` com `repository not found` ou `authentication required`.

## As opções

A doc do Argo ([Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)) suporta, entre outras:

| Opção | URL | Credencial | Quando usar |
| --- | --- | --- | --- |
| **SSH deploy key** (a deste guia) | `git@github.com:...` | Chave privada, **só de leitura e só para este repo** | Um repo. É o menor privilégio |
| HTTPS + token | `https://github.com/...` | Username e token (PAT) | Quando o SSH está bloqueado. Usa um token só com leitura |
| GitHub App | `https://github.com/...` | App id + chave privada | Muitos repos / equipas |

## Passo a passo com uma deploy key

**1. Gera uma chave só para o Argo** (sem passphrase, o Argo não tem onde a pedir):

```bash
ssh-keygen -t ed25519 -C "argocd gitops-crossplane-lab" -f ~/.ssh/argocd-gitops-lab -N ""
```

> [!warning] Não uses a tua chave pessoal
> A tua chave pessoal dá acesso a **todos** os teus repos, com escrita. Se a pusesses no cluster, quem lá chegasse ficava com tudo. Uma deploy key só lê este repo.

**2. Regista a chave pública no GitHub:** no repo → **Settings** → **Deploy keys** → **Add deploy key**. Dá-lhe um título, cola o resultado de `cat ~/.ssh/argocd-gitops-lab.pub` e **não** marques *Allow write access*.

**3. Testa a chave no teu PC**, antes de ir para o cluster. O `-F /dev/null` ignora o teu `~/.ssh/config`, para garantir que é mesmo esta chave que está a ser usada:

```bash
GIT_SSH_COMMAND="ssh -F /dev/null -i ~/.ssh/argocd-gitops-lab -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new" \
  git ls-remote git@github.com:<o-teu-user>/gitops-crossplane-lab.git
```

Se não der erro de autenticação, a chave funciona (num repo vazio o comando não lista nada).

**4. Dá a chave ao Argo**, numa Secret no namespace `argocd`. É a forma declarativa da doc: uma Secret com a label `argocd.argoproj.io/secret-type: repository`. Faz-se depois de o Argo estar instalado (Fase 2):

```bash
kubectl create secret generic repo-gitops-lab -n argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:<o-teu-user>/gitops-crossplane-lab.git \
  --from-file=sshPrivateKey=$HOME/.ssh/argocd-gitops-lab

kubectl label secret repo-gitops-lab -n argocd argocd.argoproj.io/secret-type=repository
```

- `type=git` e `url` dizem ao Argo de que repo é a credencial. O `url` tem de bater com o `repoURL` dos manifestos.
- `sshPrivateKey` é a chave privada.
- A label é o que faz o Argo tratar a Secret como credencial de repositório.
- O Argo traz por omissão as chaves de host do GitHub (`known_hosts`), por isso não tens de as adicionar.

**5. Confirma:** na UI do Argo, **Settings** → **Repositories**, o repo deve aparecer com *Connection Status: Successful*.

> [!danger] A chave privada nunca vai para o Git
> A Secret cria-se à mão no bootstrap, tal como a do floci. Para guardar segredos no Git de forma segura existem o Sealed Secrets e o External Secrets Operator.

## Alternativa: HTTPS + token

Se preferires HTTPS, usa `https://github.com/<o-teu-user>/gitops-crossplane-lab.git` no `REPO` do sed e esta Secret:

```bash
kubectl create secret generic repo-gitops-lab -n argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/<o-teu-user>/gitops-crossplane-lab.git \
  --from-literal=username=<o-teu-user> \
  --from-literal=password=<TOKEN>

kubectl label secret repo-gitops-lab -n argocd argocd.argoproj.io/secret-type=repository
```

O token deve ser só de leitura e limitado a este repo (no GitHub, um *fine-grained personal access token* com permissão `Contents: Read-only`).

## Os charts Helm não precisam de credencial

O `argoproj.github.io/argo-helm` e o `charts.crossplane.io/stable` são públicos. A credencial só é preciso para o teu repo.

---

# Fase 2: bootstrap

São os **únicos** comandos "à mão" do lab (mais as Secrets):

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd --version 10.11.6 \
  -n argocd --create-namespace -f argocd/values.yaml --wait --timeout 5m
```

- `helm upgrade --install` instala se não existir e atualiza se já existir, por isso podes repetir o comando.
- `--version 10.11.6` é a versão do chart que instala o Argo CD `v3.5.6`.
- `-f argocd/values.yaml` é o mesmo ficheiro que a app `argocd` vai ler do Git, depois de assumir.
- `--create-namespace` cria o namespace `argocd`. O `--wait` espera que os pods fiquem prontos.

> [!warning] Já tens um Argo instalado de outra forma?
> Se instalaste o Argo com `kubectl apply` (por exemplo na versão anterior deste lab), o Helm recusa-se a adotar esses objetos. Apaga primeiro o namespace e os CRDs: `kubectl delete namespace argocd` e `kubectl delete crd applications.argoproj.io applicationsets.argoproj.io appprojects.argoproj.io`.

Se o repo é **privado**, dá agora a credencial ao Argo (passo 4 de [Repo privado](#repo-privado-dar-acesso-ao-argo)). Tem de ser antes do `root`.

A Secret com as chaves do floci **não vai para o Git**, por isso cria-se à mão. O namespace tem de existir antes:

```bash
kubectl create namespace crossplane-system

kubectl create secret generic aws-secret -n crossplane-system \
  --from-literal=creds='[default]
aws_access_key_id = test
aws_secret_access_key = test'
```

Agora passa o resto para o Argo:

```bash
kubectl apply -f bootstrap/root.yaml

kubectl get applications -n argocd -w
```

As apps vão aparecendo, uma wave de cada vez: `argocd`, `crossplane`, `packages`, `provider-configs` e `apis`, e por fim a `consumers`.

> [!note] `Degraded` no início é normal
> A app `packages` passa por `Degraded` enquanto o Crossplane descarrega e arranca o provider. O `root` fica à espera na wave 2 e avança sozinho quando a `packages` passa a `Healthy`.

> [!warning] Por testar
> Não corri este fluxo de ponta a ponta, por isso não tenho tempos nem outputs reais para te mostrar. Se alguma app ficar `OutOfSync` e parada, vê a tabela [Se correr mal](#se-correr-mal).

## A UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo

kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abre `https://localhost:8080` no browser do Windows e aceita o certificado self-signed. O utilizador é `admin` e a password é a de cima. Vês o `root` com as 6 apps por baixo. Dentro da `consumers` aparece o XR e, por baixo dele, o `Bucket` que o Crossplane compôs.

---

# Fase 3: validar

Noutro terminal:

```bash
# Argo
kubectl get applications -n argocd

# Plataforma
kubectl get pods -n crossplane-system
kubectl get providers,functions
kubectl get clusterproviderconfigs.aws.m.upbound.io

# API
kubectl get xrd
kubectl get compositions

# Consumo
kubectl get storagebuckets.platform.example.org -n default
kubectl get buckets.s3.aws.m.upbound.io -n default

# O resultado, no floci
awsf s3 ls
```

Esperado:

- as apps `Synced` e `Healthy`;
- providers e functions com `INSTALLED=True` e `HEALTHY=True`;
- o XRD com `ESTABLISHED=True`;
- o `my-bucket` com `SYNCED=True` e `READY=True`;
- o `Bucket` com `READY=True`;
- o `awsf s3 ls` a listar um bucket chamado `my-bucket`.

A árvore do XR, com a CLI do Crossplane:

```bash
crossplane resource trace storagebucket.platform.example.org/my-bucket -n default
```

E a tag que a Composition pôs no bucket:

```bash
awsf s3api get-bucket-tagging --bucket my-bucket
```

---

# Fase 4: GitOps a funcionar

O Argo vai ao Git de ~3 em ~3 minutos. Para não esperar, usa o botão **Refresh** na UI ou este comando:

```bash
kubectl annotate application <app> -n argocd argocd.argoproj.io/refresh=normal --overwrite
```

> [!warning] Ainda por testar
> Estes passos ainda não foram corridos neste lab. Se algo não acontecer, não é erro teu: regista o que viste.

## Uma app nova é um ficheiro novo

```bash
sed 's/my-bucket/other-bucket/' consumers/default/my-bucket.yaml > consumers/default/other-bucket.yaml
git add -A && git commit -m "other-bucket" && git push
kubectl annotate application consumers -n argocd argocd.argoproj.io/refresh=normal --overwrite

kubectl get storagebuckets.platform.example.org -n default
awsf s3 ls
```

## Apagar é tirar do Git

```bash
git rm consumers/default/other-bucket.yaml
git commit -m "remove other-bucket" && git push
kubectl annotate application consumers -n argocd argocd.argoproj.io/refresh=normal --overwrite

kubectl get storagebuckets.platform.example.org -n default
awsf s3 ls
```

O `prune` apaga o XR `other-bucket`. O Crossplane apaga o `Bucket` dele e o bucket desaparece do floci.

> [!warning] Não apagues o último ficheiro de uma pasta
> O Git não guarda pastas vazias. Se tirares o único XR de `consumers/`, a pasta desaparece do repo e a app dá `app path does not exist`. Por segurança o Argo **não apaga nada** nesse caso. Mesmo com a pasta lá (por exemplo com um `.gitkeep`), o sync automático recusa deixar a app vazia, a não ser que ponhas `syncPolicy.automated.allowEmpty: true`.

## Mudanças à mão

```bash
kubectl patch storagebuckets.platform.example.org my-bucket -n default --type merge -p '{"spec":{"region":"eu-west-1"}}'
kubectl get storagebuckets.platform.example.org my-bucket -n default -o jsonpath='{.spec.region}{"\n"}'
```

Repete o `get` passados uns segundos e o valor volta a `us-east-1`, o do Git. É o `selfHeal`. A mudança de região num bucket já criado pode dar erro no Crossplane até ser revertida.

## O Crossplane repõe o que apagas no floci

Isto é o Crossplane, não o Argo:

```bash
awsf s3 rb s3://my-bucket
awsf s3 ls
```

Passados uns segundos o Crossplane deteta que o bucket desapareceu e volta a criá-lo. Já o testaste no `Lab de duas compositions`.

## O Argo muda-se a ele próprio

Acrescenta esta chave ao `argocd/values.yaml`, por baixo de `configs.cm`, ao lado do health check:

```yaml
configs:
  cm:
    ui.bannercontent: "Gerido pelo Argo CD a partir do Git"
```

```bash
git commit -am "banner" && git push
kubectl annotate application argocd -n argocd argocd.argoproj.io/refresh=normal --overwrite
```

Recarrega a UI e aparece o banner. A configuração do Argo mudou por commit, aplicada pelo próprio Argo.

---

# Variante: ApplicationSet nos `consumers`

> [!warning] Variante opcional, não testada
> O guia principal usa uma `Application` por camada. Esta secção mostra como trocar **só a `consumers`** por um `ApplicationSet`. Só validei o YAML contra o CRD do Argo, não o corri.

**O que é.** Um `ApplicationSet` gera `Application`s a partir de um modelo. Com o *Git directory generator*, cria uma `Application` por cada pasta que encontra no repo. As waves e o ApplicationSet fazem coisas diferentes: as waves decidem a **ordem**, o ApplicationSet decide **quem cria as `Application`s**.

**O que muda.** A `consumers` deixa de ser uma `Application` única que lê `consumers/` inteira. Passa a haver uma `Application` por cada pasta (`consumers/default` vira `consumers-default`), com o namespace de destino igual ao nome da pasta. Criar `consumers/team-b/` passa a criar uma app nova, sem escreveres nenhum ficheiro de `Application`.

**Como trocar.** Substitui o conteúdo de `apps/consumers.yaml` por isto (mantém o nome do ficheiro, para o `root` continuar a lê-lo):

```yaml
# SOURCE: Argo CD ApplicationSet, Git directory generator
#   https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/
#   Sync waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
# ADAPTED: ONE ApplicationSet replaces a hand-written Application per consumer folder. The generator
#   creates one Application for every folder under consumers/ (consumers/default -> 'consumers-default'),
#   so a new folder (e.g. consumers/team-b/) becomes a new app with no extra file.
# WHY ONLY HERE: the platform pieces (packages, provider-configs, apis) are few, fixed and must be
#   applied in order, so they stay plain Applications with sync waves. The consumers are the folders
#   that multiply, and nothing depends on them, so they are the ones an ApplicationSet manages.
# WAVE 4: the 'root' applies this ApplicationSet last, after the CRDs of the earlier waves exist.
#   Do not count on the 'root' waiting for the Applications this ApplicationSet generates; that is
#   fine here because nothing comes after them.
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: consumers
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "4"
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: REPO_URL
      revision: HEAD
      directories:
      - path: consumers/*
  template:
    metadata:
      name: 'consumers-{{ .path.basename }}'
    spec:
      project: default
      source:
        repoURL: REPO_URL
        targetRevision: HEAD
        path: '{{ .path.path }}'
      destination:
        server: https://kubernetes.default.svc
        # consumers/<name> deploys into the namespace <name>
        namespace: '{{ .path.basename }}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
        retry:
          limit: 5
          backoff:
            duration: 10s
            factor: 2
            maxDuration: 3m
```

**Porque só os `consumers`.** As peças da plataforma (`packages`, `provider-configs`, `apis`) são poucas, fixas e têm de ser aplicadas por ordem, por isso ficam como `Application`s com wave. Os `consumers` são as pastas que se multiplicam, e nada depende deles, por isso é onde o ApplicationSet compensa.

> [!note] As waves não ordenam as apps geradas
> As waves só ordenam objetos que uma sync do Argo aplica. As `Application`s que o ApplicationSet gera são criadas pelo controller dele, por isso as waves **não as ordenam entre si**. Aqui não interessa, porque todas dependem só das waves anteriores. Para ordenar apps geradas existe o `RollingSync` (Progressive Syncs), que está em Beta e, segundo a doc, desativa o `automated` nas apps geradas. Este lab não o usa.
>
> Também não confirmei se o `root` espera que as apps geradas fiquem `Healthy` antes de passar à wave seguinte. Como a wave 4 é a última, não afeta o lab.

**Experimenta.** Depois de trocar e fazer push, cria um namespace de consumo novo, só com um ficheiro:

```bash
mkdir -p consumers/team-b
sed 's/my-bucket/team-b-bucket/; s/namespace: default/namespace: team-b/' consumers/default/my-bucket.yaml > consumers/team-b/bucket.yaml
git add -A && git commit -m "team-b" && git push

kubectl get applications -n argocd -w
```

Esperado: aparece a `consumers-team-b` sem teres escrito nenhuma `Application`. O generator volta a ler o Git de tempos a tempos; para não esperar: `kubectl annotate applicationset consumers -n argocd argocd.argoproj.io/refresh=normal --overwrite`.

Com a versão do guia principal (uma só `Application` com `directory.recurse`), uma pasta nova dentro de `consumers/` seria lida pela mesma app, mas o namespace `team-b` teria de existir antes.

---

# Opcional: desenvolver a API com `crossplane project`

A CLI do Crossplane tem um fluxo de **projects** (`crossplane project ...`) que serve para escrever e testar o XRD e a Composition no teu PC, antes de os pores no Git. Já o usaste no `Lab de duas compositions`.

> [!danger] Não uses o `crossplane project run` neste lab
> O `project run` cria o **seu próprio cluster KIND**, com um registry local, e instala lá o Crossplane e a Configuration do project. Neste lab quem instala o Crossplane e tudo o resto é o Argo, no k3d. Se usasses o `run` terias uma segunda plataforma, fora do GitOps. Aqui o project serve só de **bancada de trabalho offline**.

O fluxo é: desenvolves a API no project, testas com o `render`, e **copias os dois ficheiros** para `apis/bucket/` do repo. O Argo põe-nos no cluster.

## 1. Criar o project e as dependências

```bash
crossplane project init bucket-api
cd bucket-api

crossplane dependency add xpkg.upbound.io/upbound/provider-aws-s3:v2.9.3
crossplane dependency add xpkg.crossplane.io/crossplane-contrib/function-kcl:v0.11.2
crossplane dependency add xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.7.1
```

- O `project init` cria a estrutura (`apis/`, `examples/`, `functions/`, ...) e o `crossplane-project.yaml`.
- Cada `dependency add` regista o pacote no `crossplane-project.yaml` e dá-te os schemas, que servem para o `render` e para o editor.

> [!note] `no such host` ao descarregar
> Se o `dependency add` falhar com `lookup ... on 10.255.255.254:53: no such host`, é o DNS do WSL (o mesmo do `Lab de duas compositions`): repete o comando.

## 2. O XR de exemplo e o XRD

```bash
mkdir -p examples/storagebuckets

cat > examples/storagebuckets/my-bucket.yaml <<'EOF'
apiVersion: platform.example.org/v1alpha1
kind: StorageBucket
metadata:
  name: my-bucket
  namespace: default
spec:
  region: us-east-1
EOF

crossplane xrd generate examples/storagebuckets/my-bucket.yaml
```

O `xrd generate` lê o XR e **infere** o XRD, que fica em `apis/storagebuckets/definition.yaml`: o kind `StorageBucket`, o `scope: Namespaced` e o campo `spec.region` como `string`.

O XRD gerado é mais básico do que o do repo. Falta-lhe o `default: us-east-1` e a `description` do campo `region`. Acrescenta-os à mão, ou usa o `apis/bucket/definition.yaml` do repo como ponto de partida.

## 3. A Composition

Põe a Composition em `apis/storagebuckets/composition.yaml`. O mais simples é copiar a do repo (`apis/bucket/composition.yaml`). O `crossplane composition generate apis/storagebuckets/definition.yaml` também gera um esqueleto, mas, segundo a ajuda da CLI, só traz o passo `function-auto-ready`, por isso acabas a escrever o resto à mão.

## 4. Testar offline com o `render`

O `render` precisa de saber que functions usar. Cria um ficheiro com as mesmas `Function`s do lab:

```bash
cat > functions.yaml <<'EOF'
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-kcl
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-kcl:v0.11.2
---
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.7.1
EOF

crossplane composition render examples/storagebuckets/my-bucket.yaml apis/storagebuckets/composition.yaml functions.yaml
```

Esperado: o XR e um `Bucket` (`s3.aws.m.upbound.io`) com `crossplane.io/external-name: my-bucket` e a tag `managed-by: argocd`. Se trocares a `region` do XR, o `Bucket` muda com ela. O `render` corre as functions num container, por isso precisa de um Docker/Podman acessível (`DOCKER_HOST`).

> [!warning] Os nomes das functions têm de bater certo
> Dentro de um project, as dependências ficam com nomes como `crossplane-contrib-function-kcl` (é o nome que a doc oficial usa nas Compositions). A Composition deste lab usa `function-kcl` e `function-auto-ready`, que são os nomes dos objetos em `platform/packages/`. Por isso passa o `functions.yaml` ao `render`, como acima. **Não testei** o `render` sem esse ficheiro (a usar as dependências do project), porque o DNS do WSL falhou nas tentativas.

## 5. Levar para o repo

```bash
cp apis/storagebuckets/definition.yaml  ../gitops-crossplane-lab/apis/bucket/definition.yaml
cp apis/storagebuckets/composition.yaml ../gitops-crossplane-lab/apis/bucket/composition.yaml

cd ../gitops-crossplane-lab
git add -A && git commit -m "bucket API" && git push
```

O Argo vê a mudança em `apis/bucket/` e atualiza o XRD e a Composition. Os ficheiros do repo têm os comentários `SOURCE:`/`ADAPTED:` no topo, por isso mantém-nos se copiares por cima.

## E o `project build` e o `project push`?

Constroem uma Configuration (um pacote OCI com o XRD e a Composition) e enviam-na para um registry. Isso encaixa na secção [Os teus próprios pacotes (Configuration)](#os-teus-próprios-pacotes-configuration) de "Trocar de registry". **Não os testei.**

---

# Trocar de registry

Neste lab os pacotes vêm do `xpkg.crossplane.io` e do `xpkg.upbound.io`. Como o registry final ainda não está decidido, estas são as opções que a doc do Crossplane dá. Escolhe a que se aplicar.

> [!note] Esta secção não foi testada
> Nada do que está aqui (push, `ImageConfig`, instalar uma Configuration) foi corrido neste lab.

## Outro registry público

Basta mudar o `spec.package` (por exemplo `ghcr.io/<org>/provider-x:v1.0.0`) e fazer commit. Qualquer registry OCI serve (Harbor, GHCR, Docker Hub…).

## Registry privado (credenciais)

Uma `ImageConfig` diz ao Crossplane que credencial usar para os pacotes com um certo prefixo:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: ImageConfig
metadata:
  name: private-registry
spec:
  matchImages:
  - type: Prefix
    prefix: registry.example.com/minha-org/
  registry:
    authentication:
      pullSecretRef:
        name: private-registry-credentials
```

A Secret tem de ser do tipo `kubernetes.io/dockerconfigjson` e ficar no `crossplane-system`. **A Secret não vai para o Git**: cria-a à mão no bootstrap, ou usa Sealed Secrets ou o External Secrets Operator. A `ImageConfig` pode ir para `platform/packages/`.

## Mirror (puxar tudo de outro sítio)

Para continuares a escrever `xpkg.crossplane.io/...` nos manifests mas o pull ser feito ao teu registry:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: ImageConfig
metadata:
  name: mirror
spec:
  matchImages:
  - prefix: xpkg.crossplane.io
  rewriteImage:
    prefix: registry.example.com
```

## Os teus próprios pacotes (Configuration)

Uma Configuration empacota o XRD e a Composition num pacote OCI. Os passos:

1. Cria uma pasta **que nenhuma app do Argo leia** (ou seja, fora de `platform/`, `apis/` e `consumers/`), por exemplo `packages-src/platform-bucket/`, com cópias do `definition.yaml` e do `composition.yaml` e com este `crossplane.yaml`:

   ```yaml
   apiVersion: meta.pkg.crossplane.io/v1
   kind: Configuration
   metadata:
     name: platform-bucket
   spec:
     crossplane:
       version: ">=v2.0.0"
     dependsOn:
     - apiVersion: pkg.crossplane.io/v1
       kind: Provider
       package: xpkg.upbound.io/upbound/provider-aws-s3
       version: ">=v2.9.0"
     - apiVersion: pkg.crossplane.io/v1
       kind: Function
       package: xpkg.crossplane.io/crossplane-contrib/function-kcl
       version: ">=v0.11.0"
     - apiVersion: pkg.crossplane.io/v1
       kind: Function
       package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready
       version: ">=v0.7.0"
   ```

2. Faz o build e o push. O push usa as credenciais do docker config, por isso o login é feito para esse ficheiro:

   ```bash
   crossplane xpkg build --package-root=packages-src/platform-bucket --package-file=platform-bucket.xpkg
   podman login --authfile ~/.docker/config.json <registry>
   crossplane xpkg push -f platform-bucket.xpkg <registry>/<org>/platform-bucket:v0.1.0
   ```

3. Em `platform/packages/`, um objeto que instala o pacote:

   ```yaml
   apiVersion: pkg.crossplane.io/v1
   kind: Configuration
   metadata:
     name: platform-bucket
   spec:
     package: <registry>/<org>/platform-bucket:v0.1.0
   ```

> [!danger] Cuidados com a Configuration
> - O `crossplane.yaml` (`meta.pkg.crossplane.io`) não é um recurso do cluster. Se ficar numa pasta que o Argo sincroniza, o sync falha.
> - O mesmo XRD não pode vir de dois sítios ao mesmo tempo (da app `apis` e da Configuration).
> - **Tirar o XRD do Git apaga o XRD, e apagar um XRD apaga todos os XRs desse tipo** e, a seguir, os buckets. Para mudar de "YAML no Git" para "Configuration", faz-o num cluster novo, ou protege antes o XRD com a annotation `argocd.argoproj.io/sync-options: Prune=false`.

---

# Boas práticas

| Prática | Porquê |
| --- | --- |
| Versões fixas em tudo (chart do Argo, chart do Crossplane, tags dos pacotes) | Um upgrade é um commit, e um rollback é um `git revert` |
| Uma `Application` por camada, ordenadas com sync waves | O CRD de cada camada já existe quando a seguinte é aplicada, e a ordem vê-se na UI |
| Health check de `Application` nos `values` do Argo (`configs.cm`) | Sem ele as waves do app of apps não esperam umas pelas outras |
| `ServerSideApply=true` na app do Argo | A doc de self-management pede-o |
| `directory.recurse: true` nas apps de pastas | Para poderes organizar as pastas por API |
| Tracking por annotation | É o que a doc do Crossplane pede, e já é o default desde o Argo CD 3.0 |
| Credencial do repo privado: uma deploy key só de leitura, numa Secret criada à mão | A chave nunca vai para o Git, e se fugir só dá leitura a este repo |
| Nada de secrets no Git | A Secret do floci é criada à mão. Com uma cloud a sério, usa External Secrets ou Sealed Secrets |
| Cuidado com o `prune` em XRDs e no Crossplane | Apagar o XRD apaga os XRs e apagar o Crossplane apaga os CRDs. Considera `Prune=false` nesses recursos |
| A saúde do XR confirma-se no Crossplane | O Argo não conhece o tipo `StorageBucket`, por isso a app `consumers` pode aparecer `Healthy` mesmo com o XR `READY=False`. Confirma com `crossplane resource trace` |
| Com muitos managed resources | A doc do Crossplane sugere esconder os `ProviderConfigUsage` (`resource.exclusions`) e subir o `ARGOCD_K8S_CLIENT_QPS` para 300. Atenção: `resource.exclusions` **substitui** a lista por omissão do Argo, por isso tens de a copiar e acrescentar |

---

# Se correr mal

| Sintoma | Causa / solução |
| --- | --- |
| `k3d cluster create` fica preso em `Injecting records ... CoreDNS` | O k3s morreu: faltam os sysctls do `Setup WSL do zero.md` (ou a feature gate). `k3d cluster delete xp-argo` e cria de novo |
| `no such host` nos eventos do provider ou da function (`kubectl get events -n crossplane-system`) | Falta a correção do DNS, ou o cluster reiniciou e o k3s repôs o Corefile. Aplica a secção "DNS do cluster" |
| Provider ou function `HEALTHY=False` | `kubectl describe providerrevision` / `functionrevision`. Normalmente é DNS, um nome de pacote errado ou um registry privado sem credenciais |
| App `apis` ou `consumers` `Synced/Healthy`, mas sem nada no cluster | Falta o `directory.recurse: true`, ou a pasta está vazia |
| `consumers` com `no matches for kind "StorageBucket"` | Correu antes de o XRD estar estabelecido. Confirma o health check (`kubectl get cm argocd-cm -n argocd -o yaml \| grep argoproj.io_Application`) e `kubectl get xrd` |
| `provider-configs` com `no matches for kind "ClusterProviderConfig"` | Correu antes do provider estar instalado. Confirma o health check: `kubectl get cm argocd-cm -n argocd -o yaml \| grep argoproj.io_Application` e que o provider está `HEALTHY` |
| `Bucket` com `InvalidAccessKeyId` ou `expecting 'eu-west-2'` | O pedido foi para a AWS verdadeira. Confirma que `s3` está em `services` na `ClusterProviderConfig` |
| `Bucket` sem ficar pronto, com timeout ou `connection refused` | O floci não está a correr (`sudo podman start floci`) ou o cluster não lhe chega (repete o teste do `floci-check` da Fase 0) |
| `Bucket` sem ficar pronto e sem erro | `kubectl describe bucket.s3.aws.m.upbound.io <nome> -n default` e vê os eventos |
| `kubectl get storagebuckets` diz `the server doesn't have a resource type` | Escreve o nome por extenso: `storagebuckets.platform.example.org` |
| App `Unknown` com `repository not found` ou `authentication required` | O repo é privado e falta a credencial (ver [Repo privado](#repo-privado-dar-acesso-ao-argo)), ou o `repoURL` está errado. Confirma com o `grep repoURL` |
| Erro de SSH nos logs do `argocd-repo-server` (`kubectl logs deploy/argocd-repo-server -n argocd`) | A deploy key não está registada no repo certo, ou o `url` da Secret não bate com o `repoURL` dos manifestos. Repete o teste do passo 3 |
| `helm upgrade --install` falha com `invalid ownership metadata` | Já havia um Argo instalado de outra forma. Apaga o namespace `argocd` e os CRDs do Argo (ver aviso da Fase 2) |
| Fiz push e nada mudou | Ainda não passaram os ~3 min. Faz refresh |

---

# Limpeza

Apaga primeiro os buckets pelo Crossplane, para não ficarem órfãos no floci. Remove os ficheiros de `consumers/` do Git e dá push, ou apaga os XRs diretamente:

```bash
kubectl delete storagebuckets.platform.example.org --all -A
kubectl wait --for=delete buckets.s3.aws.m.upbound.io --all -A --timeout=3m
awsf s3 ls
```

Se o Argo recriar os XRs porque ainda estão no Git, apaga o `root` primeiro: `kubectl delete application root -n argocd`.

Depois o cluster:

```bash
k3d cluster delete xp-argo
```

O repo no GitHub fica. Se o cluster desaparecer antes de os `Bucket`s serem apagados, os buckets ficam órfãos no floci. Apaga-os à mão com `awsf s3 rb s3://<nome>`. Para repetir o lab: cluster, floci, bootstrap e `root`.

---

# Memory Trick

| Conceito | Significado |
| --- | --- |
| `Application` | "Põe esta pasta (ou chart) do Git neste cluster/namespace" |
| `root` (app of apps) | Uma `Application` cuja pasta só tem `Application`s |
| `sync-wave` | Ordem: a wave seguinte só começa quando a anterior está `Healthy` |
| `automated` + `prune` + `selfHeal` | Aplica o Git, apaga o que saiu do Git, desfaz mudanças feitas à mão |
| `Provider` / `Function` | "Crossplane, vai buscar este pacote ao registry OCI e instala-o" |
| XRD → Composition → XR | A API da plataforma → como é implementada → um pedido de quem a usa |

> [!tip] A regra principal
> **Bootstrap = `helm upgrade --install argocd` + as Secrets (repo e floci) + `kubectl apply -f bootstrap/root.yaml`. Tudo o resto = `git push`.**

---

# O que foi testado

Validado a 2026-11-03, **offline e no cluster, mas sem o fluxo completo**. Versões: Argo CD v3.5.6, chart do Crossplane 2.4.3, `provider-aws-s3` v2.9.3, `function-kcl` v0.11.2, `function-auto-ready` v0.7.1, Crossplane CLI v2.5.0, k3d com k3s v1.35.5.

Validado:

- as versões do Argo, do chart e dos três pacotes existem;
- o XR `my-bucket` cumpre o schema do XRD;
- a `ClusterProviderConfig` cumpre o CRD do `provider-family-aws`;
- o `Provider`, as `Function`s, o XRD e a Composition cumprem os CRDs do Crossplane;
- `crossplane composition render` com as functions reais (`function-kcl` e `function-auto-ready`) gera o `Bucket` (com o `external-name`, a região do XR e a tag). É **idêntico** ao `Bucket` que a versão anterior (`function-patch-and-transform`) gerava, e esse cumpria o schema do `provider-aws-s3`. Com `region: eu-west-1` e outro nome, o `Bucket` acompanha;
- o `root` e as 6 `Application`s cumprem os CRDs do Argo;
- o chart `argo-cd` 10.11.6 renderiza (`helm template`) o Argo CD `v3.5.6`, com o health check no `argocd-cm`, os CRDs, e a Secret `argocd-secret` sem dados;
- o `argocd-ssh-known-hosts-cm` do chart inclui o `github.com`;
- numa pasta temporária: `crossplane project init`, `crossplane dependency add` (os 3 pacotes), `crossplane xrd generate` e o `crossplane composition render` com um ficheiro de functions, dentro do project (a repetir quando o DNS do WSL falhou).

**Não testado:**

- o bootstrap completo: o `helm upgrade --install` e o `root`;
- o Argo a assumir a gestão de si próprio a partir do chart Helm (a app `argocd` com duas fontes);
- a deploy key e a Secret do repo privado (precisam da tua conta do GitHub);
- a ordem das waves, da 0 à 4;
- que o cluster k3d alcança o floci em `169.254.1.2.nip.io:4566`;
- a criação do bucket no floci a partir do GitOps;
- o `prune` e o `selfHeal`;
- o `crossplane composition generate`, o `project build`/`push` e o `render` dentro do project sem ficheiro de functions;
- a variante com ApplicationSet (só validei o YAML contra o CRD do Argo);
- a secção "Trocar de registry".

A versão anterior deste lab (com `provider-kubernetes` e sem ApplicationSet) diz ter sido testada a 2026-10-30. O esqueleto do `root`, do `argocd` e do `crossplane` é o mesmo, mas **não voltei a correr essa parte**.
