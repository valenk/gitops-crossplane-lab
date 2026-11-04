# FAQ: Lab de Argo CD + Crossplane + floci

Perguntas e respostas curtas sobre o lab. O guia completo está em [`Lab de ArgoCD.md`](Lab%20de%20ArgoCD.md).

> [!warning] Estado
> Os manifestos foram validados offline, mas **o lab ainda não foi corrido de ponta a ponta**. Vê "O que foi testado" no fim do guia.

---

# O que acontece neste lab, em 7 passos

1. Crias um cluster k3d e arrancas o floci (a AWS local, fora do cluster).
2. Instalas o **Argo CD** com o chart Helm (`helm upgrade --install`), e crias à mão duas Secrets: a credencial do repo (se for privado) e as chaves `test`/`test` do floci.
3. Aplicas o `bootstrap/root.yaml`. É a **única** `Application` escrita para ti aplicares à mão. A partir daqui o Git é a fonte da verdade.
4. O `root` lê a pasta `apps/` e instala, por esta ordem (**sync waves**): o Argo a gerir-se a si próprio (0), o Crossplane (1), os pacotes (2), a config do provider e as APIs (3), e por fim os consumidores (4).
5. Cada passo só começa quando o anterior está `Healthy`, por isso os CRDs de que cada camada precisa já existem.
6. O Crossplane descarrega o provider e as functions, o Argo aplica o XRD e a Composition, e o teu pedido `StorageBucket` (`consumers/default/my-bucket.yaml`) é composto num `Bucket` S3.
7. O provider cria o bucket **no floci**. Confirmas com `awsf s3 ls`.

Daí para a frente, mudar alguma coisa é `git push`, não `kubectl apply`.

**Numa frase:** o Git é a fonte da verdade, o Argo põe o cluster igual ao Git, e o Crossplane transforma um pedido simples (`StorageBucket` com uma região) num recurso real (um bucket).

---

# Os ficheiros e as pastas

## Tinha de escrever os ficheiros todos à mão?
Não. O guia dá cada ficheiro como um bloco `cat > ficheiro <<'EOF'` para copiares e colares no terminal. Não há nenhum `init` oficial que gere esta estrutura: o layout é uma escolha do guia, montada a partir de exemplos oficiais.

## De onde vêm os YAMLs?
De exemplos e docs oficiais, adaptados. Cada ficheiro começa com `# SOURCE:` (link da origem) e `# ADAPTED:` (o que foi mudado).

| Ficheiros | Origem |
| --- | --- |
| `bootstrap/root.yaml`, `apps/*.yaml` | Argo CD, padrão App of Apps ([doc](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)) |
| `argocd/values.yaml`, `apps/argocd.yaml` | Chart Helm [`argo-cd`](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd), [Manage Argo CD Using Argo CD](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#manage-argo-cd-using-argo-cd), [Multiple sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/#helm-value-files-from-external-git-repository) e [Health → Argo CD App](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#argocd-app) (o Lua é o da doc) |
| `platform/packages/*` | Crossplane, [Providers](https://docs.crossplane.io/latest/packages/providers/) e [Functions](https://docs.crossplane.io/latest/packages/functions/) |
| `platform/provider-configs/aws.yaml` | O teu `Lab de duas compositions` (testado contra o floci) |
| `apis/bucket/*` | Crossplane, [Get Started With Composition](https://docs.crossplane.io/latest/get-started/get-started-with-composition/), e o formato KCL inline do [`composition-kcl.yaml`](https://docs.crossplane.io/latest/manifests/get-started/composition/composition-kcl.yaml) oficial |
| `consumers/default/my-bucket.yaml` | Exemplo próprio deste lab |

## Foram validados?
Sim, offline: os schemas dos CRDs reais (`crossplane resource validate`), o `crossplane composition render` com as functions reais, e o `helm template` do Argo. Isso confirma que a **sintaxe e os campos estão certos**. Não confirma o comportamento em runtime.

## As pastas estão como se faz normalmente?
É um layout razoável e comum, mas **não há um padrão oficial** (a própria doc do Argo diz que não há um padrão único). Bate com a prática: `bootstrap/` + `apps/` (App of Apps), e uma divisão em plataforma, API e consumidores. Um repo real costuma ter mais: vários ambientes (dev/prod), `AppProject` próprio, e plataforma e consumidores em repos separados.

## O que faz cada pasta?
| Pasta | O que é |
| --- | --- |
| `bootstrap/` | O `root`, aplicado à mão uma vez |
| `argocd/` | Os `values.yaml` do chart Helm do Argo (com o health check das `Application`s) |
| `apps/` | O que o `root` lê: uma `Application` por camada (`argocd`, `crossplane`, `packages`, `provider-configs`, `apis`, `consumers`) |
| `platform/packages/` | `Provider` e `Function`s que o Crossplane descarrega |
| `platform/provider-configs/` | Como o provider fala com o floci |
| `apis/bucket/` | O XRD (a API `StorageBucket`) e a Composition (como se cria o bucket) |
| `consumers/<namespace>/` | Os pedidos de quem usa a API (os XRs) |

---

# Sync waves e ApplicationSet

## O que são sync waves?
Um número numa annotation (`argocd.argoproj.io/sync-wave: "2"`). Quando o Argo aplica vários objetos, aplica primeiro os de wave mais baixa e só avança quando esses estão `Healthy`. Aqui servem para a ordem de instalação: o Argo, depois o Crossplane (que cria os CRDs), depois o resto.

## São obrigatórias?
Não. Servem para tornar a ordem **garantida**. Sem elas, tudo era criado ao mesmo tempo e as apps que dependem de CRDs ainda inexistentes falhavam e voltavam a tentar. Provavelmente funcionava, mas **não o testei**. O lab usa-as porque a ordem importa.

## O que é um ApplicationSet?
Um único ficheiro que **gera várias `Application`s** a partir de um modelo, por exemplo uma por cada pasta de `consumers/`. O lab principal **não o usa**: tem uma `Application` por camada. Há uma variante opcional no fim do guia.

## As waves e o ApplicationSet fazem a mesma coisa?
Não, fazem coisas diferentes:

| Ferramenta | Responde a | Exemplo |
| --- | --- | --- |
| Sync waves | "Em que **ordem**?" | `crossplane` (1) antes de `packages` (2) |
| ApplicationSet | "Quem **cria** as Applications?" | Uma app por cada pasta de `consumers/` |

Não são alternativas, mas interferem uma na outra: as waves só ordenam objetos que uma sync do Argo aplica, e as `Application`s geradas por um ApplicationSet são criadas pelo controller dele. Por isso **as waves não ordenam as apps geradas entre si**. É a diferença que vale a pena ver ao comparar as duas versões do lab.

## Porque é que o lab começa só com waves?
Para perceberes primeiro o que a ordem dá e o que cada app precisa. Depois podes experimentar a variante com ApplicationSet e ver o que muda (uma app nova por pasta, o namespace do consumidor, e o facto de as waves já não ordenarem as apps geradas).

## Porque é que os pacotes, a config e as APIs não seriam um ApplicationSet?
São poucos, fixos e têm de ser aplicados por ordem. Para ordenar apps geradas existe o `RollingSync` (Progressive Syncs), que está em Beta e, segundo a doc do Argo, **desativa o `automated`** nas apps geradas. O lab não o usa.

---

# Crossplane, floci e a CLI

## Posso provisionar no floci em vez de pods no cluster?
Sim, e é o que o lab faz. A Composition cria um `Bucket` S3 (`provider-aws-s3`), e a `ClusterProviderConfig` aponta para o floci. O floci corre fora do cluster e é visto de dentro como `169.254.1.2.nip.io:4566`.

## O lab usa a CLI do Crossplane?
Para correr, não: precisa só de `kubectl`, `git`, `k3d` e `aws`. A CLI aparece no guia em dois sítios opcionais: `crossplane resource trace` (diagnóstico) e a secção "Opcional: desenvolver a API com `crossplane project`".

## Posso usar o `crossplane project`?
Para **desenvolver** a API, sim: `project init`, `dependency add`, `xrd generate` e `composition render` testam o XRD e a Composition offline. Depois copias os dois ficheiros para `apis/bucket/`. O que **não** deves usar é o `crossplane project run`: cria o seu próprio cluster KIND e instala lá o Crossplane, em conflito com o Argo.

---

## O que é a function KCL e o que faz a Composition?
A Composition é a receita: "quando alguém pedir um `StorageBucket`, cria um `Bucket` S3". Quem a executa é a **function** `function-kcl`, que corre o código KCL escrito dentro da Composition. O código tem 3 ideias: lê o XR (`option("params").oxr`), monta o `Bucket` copiando o nome e a região, e devolve-o em `items`. Depois a `function-auto-ready` marca o XR como pronto.

## Porque KCL e não patch-and-transform?
O patch-and-transform descreve a Composition só em YAML (copia campos do XR para os recursos). O KCL descreve-a em código. Para o que este lab faz dão o mesmo resultado, e confirmei que o `Bucket` gerado é **idêntico**. O KCL escolheu-se por ser a opção para composições com lógica dinâmica e por fazer uma Composition com um único ficheiro. A versão com patch-and-transform está no `Lab de duas compositions`.

## O KCL está num project ou inline?
Inline, dentro da Composition (`input.spec.source`), tal como no exemplo oficial. No teu `Crossplane v2.md` o KCL estava numa função dentro de um project (`main.k`), que precisa de `build` e `push`. Inline é mais simples para o Argo, porque tudo vive no Git.

---

# Comandos que parecem estranhos

## Porque é que os comandos do CoreDNS existem? Preciso deles?
O `sed` troca no Corefile o `forward . /etc/resolv.conf` (o CoreDNS pergunta ao DNS do podman) por `forward . 1.1.1.1 8.8.8.8` (pergunta diretamente à Cloudflare e à Google). O `kubectl replace` grava a ConfigMap, e o `rollout restart` reinicia o CoreDNS.

É um **workaround** de um lab anterior: programas em Go (o Crossplane, o Argo) podiam ver zero respostas DNS e dar `no such host`. Fiz 30 queries ao CoreDNS deste cluster sem o fix e todas tiveram resposta, por isso **não consegui reproduzir o problema**. Aplica-o só se aparecer o erro. O k3s repõe o Corefile quando o cluster reinicia, e é preciso repetir.

## Porque é que o Argo é instalado com Helm e não com Kustomize?
Porque o lab escolheu Helm. Os dois funcionam:

| | Helm (o do lab) | Kustomize |
| --- | --- | --- |
| Configuração | Um `values.yaml` | Patches sobre o `install.yaml` oficial |
| Versão | A do chart (`10.11.6` = Argo CD `v3.5.6`) | A do `install.yaml` |
| Bootstrap | `helm upgrade --install` | `kubectl apply -k` |
| Na doc oficial do Argo | Chart da comunidade (`argo-helm`) | É o método que a doc "Manage Argo CD Using Argo CD" sugere |

O chart do Argo tem suporte para ser o próprio Argo a instalá-lo (marca os CRDs com `ServerSideApply=true`, segundo o README do chart). O Crossplane também usa Helm, por isso o lab fica com o mesmo método nos dois.

## Porque é que os valores do Argo são usados duas vezes?
Uma à mão, no `helm upgrade --install ... -f argocd/values.yaml`, para o Argo existir. Outra pela app `argocd`, que lê esse mesmo ficheiro do Git (segunda fonte, `ref: values`) e passa a gerir o Argo. É o padrão da doc "Manage Argo CD Using Argo CD". **Não testei** a passagem de testemunho do Helm para o Argo.

## Para que serve o health check de `Application`?
É uma chave do `argocd-cm`, posta em `configs.cm` do `values.yaml`. O Argo 3.5.6 não avalia a saúde das `Application`s de origem. Sem este check (copiado da doc), as sync waves do `root` não esperavam umas pelas outras.

## Pus um XR noutro namespace e não funciona (`namespaces "..." not found`)
O namespace tem de existir antes. A app `consumers` só cria o namespace de destino (`default`), e só com `CreateNamespace=true`, não o que está no `metadata.namespace` do XR. Põe um `Namespace` no Git, ao lado do XR, com a annotation `argocd.argoproj.io/sync-wave: "-1"` para ser criado primeiro. O Argo repete a sync sozinho, por isso assim que o namespace existir, o XR é aplicado.

## Criei um recurso à mão com `kubectl apply` e o Argo não o apagou. Porquê?
Porque o Argo só gere o que vem do Git. Cada recurso que aplica leva a annotation `argocd.argoproj.io/tracking-id`. O `prune` só apaga recursos rastreados que saíram do Git, e o `selfHeal` só desfaz mudanças nesses recursos. Um recurso criado à mão não tem a annotation, por isso o Argo ignora-o (vimos isto com um segundo `StorageBucket`). O bucket no floci foi criado na mesma, porque quem o cria é o Crossplane, que não sabe nada do Argo. Para o Argo o gerir, põe o ficheiro no Git. Para o limpar, `kubectl delete`.

## Porque é que a Secret do floci não vai para o Git?
Segue a regra de nunca pôr secrets no Git. As chaves são `test`/`test` (o floci aceita qualquer uma), mas o hábito fica. Crias a Secret à mão no bootstrap.

## O repo tem de ser público?
Não. Um repo **público** lê-se sem credenciais, com o URL HTTPS. Um repo **privado** precisa de uma credencial dada ao Argo. Este lab usa uma **deploy key SSH** (só de leitura, só para este repo), guardada numa Secret criada à mão. Funciona com repos privados e públicos. O passo a passo está na secção "Repo privado" do guia.

## Posso usar a minha chave SSH pessoal?
Não deves. A tua chave dá acesso, com escrita, a todos os teus repos. Uma deploy key só lê o repo para que a registaste.

## Porque é que o `repoURL` não é o meu `git remote`?
O `git remote` pode usar um alias do teu `~/.ssh/config` (por exemplo `git@github-vln:...`). Esse alias só existe no teu PC. O Argo corre no cluster e precisa do host verdadeiro: `git@github.com:<user>/gitops-crossplane-lab.git`.

---

# Perguntas de quem vê o lab pela primeira vez

## Porque é que parece complicado?
Há muitas peças ao mesmo tempo: Argo, Crossplane, provider, functions, XRD, Composition e XR. E duas camadas de indireção: o pedido passa pelo XR, pela Composition e pelo provider antes de ser um bucket. O valor do lab está no **padrão** (GitOps + uma API de plataforma), não no bucket.

## Que partes costumam suscitar perguntas?
- **O hack do CoreDNS**: é um workaround, não faz parte do GitOps.
- **O health check em Lua** no `argocd-cm`: copiado da doc, mas é uma peça frágil.
- **O Crossplane a provisionar recursos locais** (floci): é para o lab não precisar de uma cloud.
- **O Argo a gerir-se a si próprio**.

São perguntas razoáveis. Não são necessariamente erros, mas convém saber justificar cada escolha.

## Isto funciona de certeza?
Os manifestos estão validados contra os schemas reais e o `render` da Composition está confirmado. **Falta correr o fluxo de ponta a ponta** (bootstrap, a ordem das waves, o floci acessível a partir do k3d, o bucket criado, e o `prune`/`selfHeal`). O guia diz exatamente o que foi e o que não foi testado.
