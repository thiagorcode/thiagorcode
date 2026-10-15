# Roteiro: Docker + Kubernetes — do zero ao avançado

> Recorte Docker + Kubernetes do roteiro DevOps geral. Linux de base, cloud e Terraform continuam no roteiro geral — aqui é só o que importa para containers.

## Como usar este roteiro

- **A ordem importa.** Kubernetes antes de Docker sólido vira decoreba que desmorona na primeira crise. Não pule etapas.
- **Estimativas consideram ~15 h/semana.** Tempo integral encurta tudo; 3 h/semana alonga. Consistência > intensidade.
- **Marque os itens "Dominar" só quando conseguir fazer sem ajuda.** É o seu painel de progresso — commite este arquivo no repositório a cada etapa concluída.
- **80% praticando, 20% lendo.** Todos os seus labs vão num repositório git próprio (ex.: `lab-docker-k8s`).
- **Ambiente:** o seu WSL2 serve para tudo até a etapa 8 (Docker + `kind` para cluster local). Na etapa 9, um VPS barato com `k3s` faz diferença.

| Etapa | Foco | Estimativa |
|---|---|---|
| 0 | Base mínima: Linux, redes e CLI do jeito que containers exigem | 2 semanas |
| 1 | O que é um container + imagens e Dockerfile | 2–3 semanas |
| 2 | Rodar, conectar e depurar | 1–2 semanas |
| 3 | Dockerfile de produção + Docker Compose | 2–3 semanas |
| 4 | Entrega: registry, tags, scan, CI | 1–2 semanas |
| 5 | Arquitetura do Kubernetes + kubectl + primeiros manifests | 3–4 semanas |
| 6 | Núcleo do dia a dia: deploys, probes, recursos, troubleshooting | 4–6 semanas |
| 7 | Expor e persistir: Service, Ingress, NetworkPolicy, storage | 3–4 semanas |
| 8 | Operação: Helm, escalabilidade, segurança, observabilidade | 5–6 semanas |
| 9 | Avançado: internals, etcd, GitOps, operators | contínua |

Resumo honesto: Docker sólido em ~2 meses e meio; núcleo de Kubernetes em ~3 meses depois disso; etapa 8 dominada em ~10–12 meses no total, a 15 h/semana.

---

## Etapa 0 — Base mínima: o que containers exigem de você (2 semanas)

Containers são Linux isolado. Sem esta base, tudo depois é decoreba.

**Aprender**
- Linux CLI: hierarquia do filesystem, permissões (`chmod`/`chown` e o que UID/GID significam — containers rodam como UIDs, nomes não passam do `/etc/passwd`), processos com `ps`
- Sinais: `SIGTERM` vs `SIGKILL` — é assim que containers param "de forma limpa" ou "à força"
- stdout/stderr: containers não gravam log em arquivo — escrevem no stdout; arquivar é problema de outro componente
- Bash: pipes, `grep`, `curl`, variáveis, exit codes (`$?`, `set -e`)
- Redes: portas e TCP/UDP, DNS (metade do Kubernetes é descoberta via DNS), HTTP/HTTPS e status codes, o conceito de proxy reverso e load balancer (Service e Ingress do K8s são exatamente isso)
- Git básico, para versionar todos os seus labs

**Praticar**
- OverTheWire **Bandit**, níveis 0–20 (overthewire.org) — Linux na prática, com propósito
- Suba um servidor HTTP qualquer (ex.: `python3 -m http.server`), consuma com `curl`, observe os headers; mate com `TERM` e com `-9` e compare o que aconteceu
- Quebre o acesso a um arquivo mexendo nas permissões, de propósito, e conserte

**Dominar**
- [ ] Explico o que um `SIGTERM` faz e por que `kill -9` é a última opção
- [ ] Faço requests com `curl` e interpreto status code e headers
- [ ] Leio `rwxr-xr-x` e sei o que UID e GID representam — e por que isso importa num container "non-root"
- [ ] Narro em 2 minutos o caminho de um `curl https://site.com`: DNS → TCP → TLS → HTTP → resposta

---

## Etapa 1 — O que é um container + imagens e Dockerfile (2–3 semanas)

**Aprender**
- Arquitetura do Docker: client → daemon (`dockerd`) → `containerd` → `runc`; e o que é OCI (spec de imagem e de runtime — por isso `docker` e `podman` usam as mesmas imagens)
- O kernel por trás: **namespaces** (pid, net, mnt, uts, ipc, user) isolam a visão; **cgroups** limitam CPU/memória. Container = processo Linux isolado e limitado, não uma mini-VM
- Imagens e layers: cada instrução do Dockerfile gera uma layer; o cache de build reusa até a primeira instrução que mudou; filesystem por cima de layers (`overlayfs`)
- Dockerfile completo: `FROM`, `RUN`, `COPY` (e por que `ADD` quase nunca), `WORKDIR`, `ENV` vs `ARG`, `USER`, `EXPOSE`, `VOLUME`, `HEALTHCHECK`, `.dockerignore`
- `CMD` vs `ENTRYPOINT`; forma shell vs forma exec — volta com força na etapa 3
- Registries: Docker Hub e GHCR; `pull`, `push`, `login`; tags e digests

**Praticar**
- Instale o Docker no WSL2 (Docker Desktop ou engine nativa com systemd habilitado — com a nativa você vê o `dockerd` de verdade no `journalctl`)
- Escreva o Dockerfile de uma API pequena (FastAPI ou Express) partindo de `python:3.x-slim` / `node:xx-slim`, sem copiar template
- `docker exec -it <container> sh` e olhe lá dentro: `ps` só enxerga os processos do container, o hostname é outro, `/proc/1` é o seu app — você está dentro de um namespace
- Mude a ordem das instruções do Dockerfile e observe o cache: o que invalida, o que reusa (dica: copie o código por último)

**Dominar**
- [ ] Explico container vs VM sem usar a palavra "leve" — namespaces, cgroups e uma imagem que é só camadas
- [ ] Escrevo Dockerfile de uma aplicação do zero, escolhendo a base conscientemente
- [ ] Sei por que "cada instrução é uma layer" dita a ordem em que escrevo o Dockerfile
- [ ] Desenho o caminho de um `docker run` do ponto de vista dos componentes (dockerd, containerd, runc, namespaces)

---

## Etapa 2 — Rodar, conectar e depurar (1–2 semanas)

**Aprender**
- `docker run` de verdade: `-d`, `-p` (publicar porta — e a diferença entre `-p` e `EXPOSE`), `-e`, `--name`, `--rm`, `--restart` (no, on-failure, always) — e o que a política de restart **não** faz: ela age em container que morreu, não em healthcheck vermelho
- Volumes vs bind mounts vs tmpfs: onde os dados vivem, o que morre com o container, quando usar cada um
- Redes: bridge padrão, redes definidas pelo usuário (com DNS entre containers resolvendo por nome), `host`, `none`
- CLI de diagnóstico: `logs -f`, `exec`, `ps -a`, `inspect` (com `--format`), `stats`, `events`, `top`, `cp`, `prune`
- Ciclo de vida: created → running → paused → exited → removed; `stop` manda SIGTERM e espera o timeout; `kill` não espera

**Praticar**
- Postgres num volume nomeado: grave dados, destrua o container, recrie em outro e prove que os dados ficaram
- Dois containers na mesma rede definida por você: a app resolvendo `db` por nome, sem IP escrito à mão
- Charadas de diagnóstico (crie de propósito): um container que sai na hora (leia o `logs`), um que roda mas não responde como esperado (`inspect` na porta), um inalcançável (rede/porta errada)

**Dominar**
- [ ] Escolho volume, bind mount ou tmpfs sem hesitar nos casos típicos (banco de dados, código no dev, arquivo temporário)
- [ ] Explico `-p 8080:80` nos dois sentidos (porta do host : porta do container)
- [ ] Diagnostico um container que "não sobe" só com `logs`, `inspect` e `events` em menos de 5 minutos
- [ ] Explico exatamente o que sobrevive e o que morre quando um container é removido

---

## Etapa 3 — Dockerfile de produção + Docker Compose (2–3 semanas)

**Aprender**
- Multi-stage builds: estágio de build (compila, resolve dependências) e estágio final mínimo (só o que roda)
- Sinais, agora a sério: forma exec (`CMD ["node", "server.js"]`) faz seu app ser o PID 1 e receber o SIGTERM; forma shell (`CMD node server.js`) coloca `/bin/sh` como PID 1, que ignora o sinal — e o `docker stop` vira `docker kill` após o timeout. Zombie reaping e por que existe `--init`
- `HEALTHCHECK`: informa estado; limitação real — no Docker puro, health vermelho **não** reinicia nada (o Kubernetes fará isso; guarde o contraste)
- Limits: `--memory` e `--cpus`; **OOMKilled = exit code 137**
- Docker Compose (v2, arquivo `compose.yaml`): services, `depends_on` com `condition: service_healthy`, redes internas, volumes, `env_file`, profiles, arquivos de override por ambiente
- Práticas que importam de verdade: usuário não-root, versões pinadas, `.dockerignore` enxuto

**Praticar**
- Ambiente completo no Compose: sua API + Postgres + Redis, com healthchecks em tudo e `depends_on` condicional. Tem que subir verde com um único `docker compose up -d`
- Encolha a sua imagem com multi-stage até virar uma fração da original — meça com `docker images`
- Prove o problema da forma shell: uma app que ignora SIGTERM por 10+ segundos no `docker stop`; corrija com a forma exec e meça de novo
- Aperte `--memory` no Postgres até morrer `137` e encontre nos logs o motivo

**Dominar**
- [ ] `docker compose up -d` sobe meu ambiente completo com healthchecks verdes
- [ ] Explico por que a forma exec do entrypoint faz o `docker stop` ser rápido
- [ ] Fiz um multi-stage de cabeça própria e sei dizer exatamente o que entra no estágio final
- [ ] Abro o `docker inspect` de qualquer container e acho limites, healthcheck, restart policy e montagens

---

## Etapa 4 — Docker como entrega: registry, tags, scan, CI (1–2 semanas)

**Aprender**
- Tags imutáveis (SHA do commit ou semver). `latest` em produção é erro, não atalho
- Scan de vulnerabilidade com **Trivy** (`trivy image`) e o fix típico do achado: trocar a base, atualizar pacotes
- BuildKit na prática: cache mounts (`RUN --mount=type=cache`) para dependências e **secrets no build** (`--mount=type=secret`) — senha nunca em `ARG`
- Build + push no GitHub Actions: login no GHCR, buildx, push com a tag do SHA

**Praticar**
- Pipeline que builda a cada push, escaneia com Trivy (falha ao achar CRITICAL) e publica com tag = SHA do commit
- Coloque cache mount no Dockerfile de uma app Python/Node e meça o tempo de build antes/depois

**Dominar**
- [ ] Nenhuma imagem minha em produção aponta para `latest`
- [ ] A pipeline falha quando a imagem tem vulnerabilidade crítica — e eu sei resolver
- [ ] Consigo recriar hoje exatamente a imagem que rodei semana passada (mesma tag, mesmo código)

---

# Kubernetes

A partir daqui, tudo se apoia nas etapas 1–4: um Pod é container(es) com IP próprio; um Deployment é "estado desejado vs observado"; a rede é DNS e portas. Se a base tremer, volte — é mais rápido que avançar trêmulo.

## Etapa 5 — Arquitetura, kubectl e primeiros manifests (3–4 semanas)

**Aprender**
- O problema que orquestração resolve: self-healing, escala, rolling updates, descoberta — e por que Docker puro não resolve isso em mais de um host
- Arquitetura de verdade: control plane (API server, `etcd`, scheduler, controller manager) e nodes (kubelet, kube-proxy, container runtime). **Tudo é conversa com a API** — kubectl, kubelet, tudo
- Cluster local: **kind** (recomendado — clusters de vários "nodes" dentro do seu Docker) ou minikube; `k3s` fica para a etapa 9
- kubectl de sobrevivência: `contexts`, `get`, `describe`, `apply`, `delete`, `logs`, `exec`, `port-forward`
- Manifests YAML: apiVersion, kind, metadata, spec, status; objetos centrais: Pod, Deployment, Service, ConfigMap, Secret, Namespace
- Labels e selectors — toda a API do K8s funciona por seleção
- Imperativo (`run`, `create`, `expose`) vs declarativo (`apply`); e estudar pela própria API: `kubectl explain deployment.spec`, `kubectl api-resources`

**Praticar**
- Cluster kind de 2 "nodes" no WSL2
- Deploy da app da etapa 3 (imagem do GHCR): Deployment + Service ClusterIP + ConfigMap; acesso via `port-forward`
- Delete pods no meio da operação e veja o control plane recriar; `kubectl scale` e veja os pods novos
- Crie tudo imperativo (`run` + `expose`), traduza para YAML, `apply`; apague o que criou imperativo — daqui para frente é declarativo
- `kubectl explain` em dez campos diferentes; descubra o que `minReadySeconds` faz sem sair do terminal

**Dominar**
- [ ] Desenho o control plane e os componentes de um node de memória
- [ ] Explico como o Deployment converge o estado desejado (reconciliation loop: deployment → replicaset → pods)
- [ ] Descubro qualquer campo de qualquer objeto com `kubectl explain` sem buscar na internet
- [ ] Explico o que `port-forward` faz e por que ele não é a forma de expor um serviço

---

## Etapa 6 — O núcleo do dia a dia: deploys, probes, recursos, troubleshooting (4–6 semanas)

A etapa que separa "apliquei um YAML" de "opero o cluster".

**Aprender**
- Deployments a fundo: `RollingUpdate` (`maxSurge`/`maxUnavailable`), revisions, `rollout status/history/undo`
- Desligamento de pod: `terminationGracePeriodSeconds`, hook `preStop`, saída da lista de endpoints — a receita de **deploy sem downtime**, uma sequência de eventos que você precisa saber narrar
- Probes: `liveness`, `readiness`, `startup`; o erro clássico do liveness agressivo (restart storm); pod "running" que não recebe tráfego = readiness
- Recursos: `requests` (direcionam o scheduling, bin-packing) vs `limits` (isolamento, throttling, OOM); classes de QoS (Guaranteed/Burstable/BestEffort); `OOMKilled` no describe
- DNS interno: `service`, `service.namespace`, FQDN (`service.namespace.svc.cluster.local`); Endpoints/EndpointSlices
- Método de troubleshooting, nessa ordem: `describe` → events → `logs` → `exec`; erros-padrão: `Pending`, `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`
- Inspeção e edição: `get -o yaml`, `-o jsonpath`, `edit`, `patch`

**Praticar**
- Mude o Compose da etapa 3 para manifests: API + Redis
- Faça o rollout de uma versão ruim e o `rollout undo` — cronometrado
- Fábrica de incidentes, um por vez, todos de propósito: `CrashLoopBackOff` (processo que falha), `Pending` (requests maiores que o node), `ImagePullBackOff` (tag errada), `OOMKilled` (memória apertada) — cada um diagnosticado com `describe` + logs, sem adivinhar
- ConfigMap via env e via volume: veja o volume recarregar sozinho (e o `subPath` não atualizar)
- Uma app acessando a outra por DNS dentro do cluster, com `exec` + `curl` entre pods

**Dominar**
- [ ] Narro a sequência de shutdown de um pod (SIGTERM, preStop, grace period, sai dos endpoints) e por que ela evita 502 no rollout
- [ ] Diagnostico CrashLoopBackOff / Pending / ImagePullBackOff / OOMKilled com método, não com chute
- [ ] Explico requests vs limits e prevejo a QoS de um manifest só de olhar
- [ ] Sei dizer num minuto por que um pod running não recebe tráfego
- [ ] Escrevi um Deployment zero-downtime (probes calibradas + preStop + grace period) e provei com um `while curl` rodando durante o rollout

---

## Etapa 7 — Expor e persistir: Service, Ingress, NetworkPolicy, storage (3–4 semanas)

**Aprender**
- Service por baixo: kube-proxy (iptables/IPVS) redireciona para endpoints; o VIP do ClusterIP não é um servidor — é regra. Tipos: ClusterIP, NodePort, LoadBalancer, headless (`clusterIP: None`)
- Ingress com ingress-nginx: roteamento por host/path, TLS com **cert-manager** (Let's Encrypt); a diferença entre "expor via Service" e "expor via Ingress"
- **NetworkPolicies**: namespace NÃO é fronteira de rede; default-deny + allows por labels é o firewall nativo
- Storage: PV (o disco), PVC (o pedido de disco), StorageClass (quem provisiona); `Retain` vs `Delete`
- StatefulSet: identidade estável (ordinal), headless service, um PVC por réplica; StatefulSet vs Deployment
- Realidade de mercado: banco em produção costuma ser managed fora do cluster (RDS etc.) — mas você precisa saber operar dentro para labs e casos reais

**Praticar**
- Desenhe o caminho request externo → Ingress → Service → Endpoints → Pod; depois implemente com ingress-nginx e duas rotas por path
- TLS real com cert-manager num k3s em VPS (no kind, use certificado self-signed para o lab)
- NetworkPolicy de verdade: default-deny no namespace, depois libere exatamente app→Postgres — prove com `exec` + `curl` que o resto morreu
- Postgres em StatefulSet com PVC: mate o pod e veja o volume reancorar na mesma identidade; delete o StateSet e veja o PVC sobreviver
- Teste `Retain` vs `Delete`: delete uma PVC em cada modo e observe o destino do PV

**Dominar**
- [ ] Descrevo o caminho completo do request externo até o pod sem hesitar
- [ ] Sei montar um namespace "trancado" com NetworkPolicy default-deny e exceções mínimas testadas
- [ ] Sei quando StatefulSet e quando Deployment — e quando o banco merece ficar fora do cluster
- [ ] Explico o trio PV/PVC/StorageClass e digo o que acontece com o disco quando aperto delete

---

## Etapa 8 — Operação de verdade: Helm, escalabilidade, segurança, observabilidade (5–6 semanas)

**Aprender**
- Helm: estrutura de chart (Chart.yaml, templates, values), `install/upgrade/rollback`; leia os templates do chart antes de instalar — um chart é opinião de terceiros rodando no seu cluster
- Escala: **HPA** com metrics-server (target CPU/mem, min/max); VPA e cluster-autoscaler só como conceito; quando escala horizontal não resolve (banco, filas)
- Jobs e CronJobs; initContainers; sidecars nativos (init container com `restartPolicy: Always`, K8s ≥ 1.29)
- Segurança aplicada, por ordem de impacto: ServiceAccounts (desligue `automountServiceAccountToken` se o pod não fala com a API), RBAC (Role/RoleBinding/ClusterRole, verbs), pods non-root com `readOnlyRootFilesystem` e caps dropadas (Pod Security Standards: baseline/restricted); Secret no K8s é base64, não criptografia — sealed-secrets ou external-secrets quando precisar guardar de verdade
- Confiabilidade: PodDisruptionBudget; drenagem de node (cordon/drain)
- Observabilidade do cluster: kube-prometheus-stack (Prometheus + Grafana + Alertmanager), ServiceMonitors, PromQL básico (rate, latência, erros), alertas que significam algo; Loki para logs centralizados

**Praticar**
- `helm install` do kube-prometheus-stack; dashboard da sua app (latência, taxa de erro, CPU/mem) e um alerta quando um pod reinicia
- Escreva um chart pequeno para a sua app: Deployment/Service/Ingress + values por ambiente (dev/prod)
- HPA de 1→5 réplicas sob carga (k6 ou hey) e veja o scale-out nos gráficos
- Lab de RBAC: ServiceAccount que só lista pods no namespace X — prove com `kubectl --as`
- `drain` num node com PDB protegendo — o drain respeita sua disponibilidade mínima

**Dominar**
- [ ] Instalo, atualizo e reviro charts Helm — sempre lendo values e templates antes
- [ ] Minha app escala sozinha sob carga, e eu vejo isso nos gráficos
- [ ] Meus pods passam o padrão restricted (non-root, read-only fs, sem caps) sem quebrar
- [ ] Meus alertas são todos acionáveis — quando tocam, é hora de agir

---

## Etapa 9 — O avançado de verdade (contínua)

Comece aqui só com a etapa 8 dominada — é onde conhecimento vira senioridade.

**Aprender**
- **Kubernetes the Hard Way** (Kelsey Hightower): monte um cluster peça por peça — certificados, binários, etcd — para ver o que o kind esconde
- **etcd a sério**: snapshot com `etcdctl`, o que acontece quando ele morre, e restore do cluster inteiro a partir do snapshot
- Scheduling avançado: taints/tolerations, affinity/anti-affinity, topology spread, PriorityClass
- CRDs e Operators: entenda o modelo (CRD = tipo novo; operator = controlador de desejado vs observado); use operators de mercado com competência; construa um só se o trabalho pedir
- **GitOps com ArgoCD** (ou Flux): o repositório é a única fonte da verdade; mudança é PR, rollback é git revert; sync waves e hooks
- Troubleshooting do cluster (não do pod): node NotReady, DiskPressure, certificado expirado, CNI com problema — e um runbook para cada
- Kyverno/OPA Gatekeeper (política como código), quando o contexto exigir compliance

**Praticar**
- Cluster **real de 1–3 nós**: k3s num VPS barato; sua app completa lá fora com Ingress e TLS
- Migre a operação da app para ArgoCD: nada de `apply` manual — toda mudança por PR
- Backup do etcd + game day de desastre: destrua e reconstitua o cluster pelo snapshot
- Kyverno negando pods root num namespace
- Game days regulares: kubelet morto, disco cheio, etcd fora — recupere com runbook, sem improviso

**Dominar**
- [ ] Construí (KTHW) ou explico componente a componente o funcionamento do cluster
- [ ] Recuperei um cluster inteiro a partir de um snapshot do etcd
- [ ] Minha única via de mudança no cluster é o Git (ArgoCD) — manual é exceção documentada
- [ ] Tenho runbooks testados: os desastres do meu cluster têm receita escrita

---

## Projeto capstone — o portfólio

Um único repositório público que conta a história inteira: **API com testes + Postgres + Redis**, onde:

1. Dockerfiles multi-stage, imagens no GHCR com tag por SHA (etapas 3–4)
2. Compose para dev local; chart Helm para o cluster (etapa 8)
3. k3s em VPS: Ingress com TLS via cert-manager, HPA, PDB, NetworkPolicies (etapas 7–8)
4. kube-prometheus-stack: dashboard da app + alertas; Loki para logs (etapa 8)
5. ArgoCD no comando — toda mudança por PR (etapa 9)
6. Um game day executado com postmortem escrito no README, junto do diagrama de arquitetura

Esse repositório vale mais que qualquer certificação: é prova de que você **construiu** infraestrutura, não só leu sobre ela.

## O que NÃO aprender agora (cortar sem culpa)

- **Docker Swarm** — o mercado encerrou a disputa; orquestração se aprende no Kubernetes. Se uma vaga pedir Swarm depois, uma semana resolve — os conceitos transferem
- **Service mesh (Istio, Linkerd)** — até aparecer um problema real que exija (domine Ingress + NetworkPolicy primeiro)
- **ECS, Nomad, Fargate** — depois do Kubernetes sólido, se o trabalho pedir
- **Cilium/eBPF e CNIs exóticos** — domine o modelo de rede padrão primeiro
- **Rancher/OpenShift e UIs gerenciadas** — kubectl puro primeiro; depois, qualquer UI fica trivial
- **k9s** — excelente ferramenta, mas só depois do kubectl fluente, senão vira muleta
- **Escrever operators em Go** antes de saber usar operators
- **Certificações no início** — depois da etapa 8, se quiser validação: **CKA** (prática de verdade, a que o mercado respeita neste tema; CKAD se você vier do lado de desenvolvimento)

## Recursos (todos gratuitos)

- **docs.docker.com** e **kubernetes.io/docs** — a bíblia; habitue-se a ler documentação oficial em inglês
- **overthewire.org** (Bandit) — etapa 0
- **killercoda.com** — cenários interativos de Kubernetes, ótimos para os drills das etapas 5–7
- **github.com/kelseyhightower/kubernetes-the-hard-way** — etapa 9

---

*Checklist é contrato: marque "Dominar" só quando fizer sem ajuda.*
