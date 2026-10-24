# inventra-gitops

Estado desejado do cluster de **produção** da Inventra. O Argo CD observa este repo e
aplica o que está na `main`. Ninguém roda `kubectl apply` à mão (exceto o bootstrap, uma vez).

## Estrutura

```
bootstrap/argocd/      instala o Argo CD (aplicado manualmente uma vez)
bootstrap/root-app.yaml  app-of-apps que aponta para clusters/prod
clusters/prod/         Applications do Argo (cert-manager, sealed-secrets, argo-rollouts, issuers, inventra-api)
platform/              cert-manager, sealed-secrets, argo-rollouts, ClusterIssuers (Let's Encrypt)
apps/inventra-api/     base (Rollout canary) + overlays/prod (tag da imagem, IngressRoute, segredos selados)
```

Ordem de sync (sync-waves): cert-manager, sealed-secrets e argo-rollouts (0) → cluster-issuers (1) → inventra-api (2).

## Fluxo de release

1. Tag `vX.Y.Z` no repo do serviço → o CD builda e publica `ghcr.io/inventratech/inventra-api:vX.Y.Z`.
2. O CD abre uma PR aqui alterando `images[].newTag` em `apps/inventra-api/overlays/prod/kustomization.yaml`.
3. A PR é revisada e mergeada (este é o gate de produção) e o Argo CD sincroniza.
4. O Argo Rollouts faz o deploy **canary**: a versão nova recebe 25% → 50% → 75% → 100% do
   tráfego, com 5 minutos em cada passo (`apps/inventra-api/base/rollout.yaml`). Se o pod novo não
   ficar Ready, o rollout para e a versão estável continua atendendo.
5. **Rollback:** `git revert` do commit do bump e merge. Durante o canary, também dá para abortar (abaixo).

### Acompanhar o canary

No Argo CD, o app `inventra-api` mostra o `Rollout` como `Suspended` durante as pausas e
`Healthy` no fim. Para ver os passos e os pesos, use o plugin do Argo Rollouts no servidor:

```bash
curl -fsSLo kubectl-argo-rollouts https://github.com/argoproj/argo-rollouts/releases/download/v1.10.1/kubectl-argo-rollouts-linux-amd64
sudo install -m 755 kubectl-argo-rollouts /usr/local/bin/kubectl-argo-rollouts

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml   # no k3s (como root/sudo)
kubectl argo rollouts get rollout inventra-api -n inventra --watch
kubectl argo rollouts promote inventra-api -n inventra   # pula a pausa atual
kubectl argo rollouts abort inventra-api -n inventra     # volta 100% do tráfego para a versão estável
```

> O passo 2 ainda não existe: depende do marco M5 (ajuste do `cd-java.yaml`).

## Bootstrap (uma vez, no servidor novo)

Pré-requisito: cluster k3s no ar e `kubectl` funcionando.

```bash
# --server-side é obrigatório: os CRDs do Argo CD estouram o limite de annotation do apply comum.
kubectl apply -k bootstrap/argocd --server-side --force-conflicts
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -f bootstrap/root-app.yaml
```

O bootstrap **não** é sincronizado pelo Argo CD: depois de mudar `bootstrap/argocd` (por exemplo, a
versão da extensão), reaplique no cluster com o mesmo `kubectl apply -k bootstrap/argocd --server-side --force-conflicts`.

A UI do Argo CD tem a **extensão de Rollout** (`argoproj-labs/rollout-extension`, v0.3.7): ao clicar no
`Rollout` na árvore do app aparece a aba **Rollout**, com os passos do canary, os pesos e as revisões.
A v0.4.x exige Argo CD >= 3.5; ao atualizar o Argo CD, atualize a extensão junto.

O repo é **público**: o Argo CD lê sem nenhuma credencial. Por isso **nunca** coloque segredo em texto puro aqui (a CI bloqueia `kind: Secret`); só `SealedSecret`, que apenas o cluster decifra.

Senha inicial do `admin`:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

A UI do Argo CD **não é pública**. Acesse pelo túnel (com o `kubectl` apontando para o cluster via Tailscale):

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:80
# abrir http://localhost:8080
```

Depois de instalar o sealed-secrets, **faça backup da chave privada num cofre da equipe**
(sem ela não dá para decifrar os segredos num servidor novo):

```bash
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-master.key
# guardar no gerenciador de senhas e APAGAR o arquivo local (está no .gitignore)
```

## Segredos

Só `SealedSecret` entra no git (a CI bloqueia `kind: Secret`).

Chaves de `inventra-api-secrets`: `DB_HOST DB_PORT DB_NAME DB_USER DB_PASSWORD JWT_SECRET
CLOUDINARY_CLOUD_NAME CLOUDINARY_API_KEY CLOUDINARY_API_SECRET REDIS_HOST REDIS_PORT
REDIS_USERNAME REDIS_PASSWORD SENDGRID_API_KEY MAIL_FROM PASSWORD_RESET_WEB_URL`.

```bash
kubectl create secret generic inventra-api-secrets -n inventra \
  --from-literal=DB_HOST=... --from-literal=JWT_SECRET=... \
  --dry-run=client -o yaml \
  | kubeseal --controller-namespace kube-system -o yaml \
  > apps/inventra-api/overlays/prod/sealed-secret-api.yaml
```

Para **adicionar chaves** sem refazer as existentes, use `--merge-into` (não precisa dos valores antigos):

```bash
kubectl create secret generic inventra-api-secrets -n inventra   --from-literal=REDIS_HOST=... --from-literal=REDIS_PORT=...   --from-literal=REDIS_USERNAME=... --from-literal=REDIS_PASSWORD=...   --from-literal=SENDGRID_API_KEY=... --from-literal=MAIL_FROM=...   --from-literal=PASSWORD_RESET_WEB_URL=https://SEU-FRONT/redefinir-senha   --dry-run=client -o yaml   | kubeseal --controller-namespace kube-system -o yaml       --merge-into apps/inventra-api/overlays/prod/sealed-secret-api.yaml
```

Imagem no GHCR: o pacote é **público** (o repo da API é público), então não há pull secret. Depois do primeiro push, em *Packages → inventra-api → Package settings → Change visibility → Public*.

Depois descomente as linhas correspondentes em `overlays/prod/kustomization.yaml`.

## Pendências (TODO)

- [ ] **API:** liberar `/actuator/health/**` no `SecurityConfig` (hoje só `/actuator/health` é `permitAll`; as probes de liveness e readiness retornariam 401).
- [ ] **API:** trocar o CI para não usar banco e Redis reais.
- [ ] Gerar os SealedSecrets (acima) e descomentá-los no overlay.
- [ ] `CODEOWNERS`: trocar o dono individual por um time.
- [ ] Login do Argo via GitHub OAuth (Dex) e acesso de leitura para a equipe.
- [ ] Proteger a `main` (PR obrigatória + check `Validate`).
- [ ] M5: CD por tag abrindo a PR de bump aqui.
