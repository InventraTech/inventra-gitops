# inventra-gitops

Estado desejado do cluster de **produção** da Inventra. O Argo CD observa este repo e
aplica o que está na `main`. Ninguém roda `kubectl apply` à mão (exceto o bootstrap, uma vez).

## Estrutura

```
bootstrap/argocd/      instala o Argo CD (aplicado manualmente uma vez)
bootstrap/root-app.yaml  app-of-apps que aponta para clusters/prod
clusters/prod/         Applications do Argo (cert-manager, sealed-secrets, issuers, inventra-api)
platform/              cert-manager, sealed-secrets, ClusterIssuers (Let's Encrypt)
apps/inventra-api/     base + overlays/prod (tag da imagem, ingress, segredos selados)
```

Ordem de sync (sync-waves): cert-manager e sealed-secrets (0) → cluster-issuers (1) → inventra-api (2).

## Fluxo de release

1. Tag `vX.Y.Z` no repo do serviço → o CD builda e publica `ghcr.io/inventratech/inventra-api:vX.Y.Z`.
2. O CD abre uma PR aqui alterando `images[].newTag` em `apps/inventra-api/overlays/prod/kustomization.yaml`.
3. A PR é revisada e mergeada (este é o gate de produção) e o Argo CD sincroniza.
4. **Rollback:** `git revert` do commit do bump e merge.

> O passo 2 ainda não existe: depende do marco M5 (ajuste do `cd-java.yaml`).

## Bootstrap (uma vez, no servidor novo)

Pré-requisito: cluster k3s no ar e `kubectl` funcionando.

```bash
# --server-side é obrigatório: os CRDs do Argo CD estouram o limite de annotation do apply comum.
kubectl apply -k bootstrap/argocd --server-side --force-conflicts
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -f bootstrap/root-app.yaml
```

O repo é privado: antes do `root-app`, cadastre uma **deploy key somente leitura** (as `repoURL` usam SSH):

```bash
ssh-keygen -t ed25519 -f argocd-deploy -N "" -C "argocd-inventra-gitops"
gh repo deploy-key add argocd-deploy.pub --repo InventraTech/inventra-gitops --title "argocd (somente leitura)"
kubectl -n argocd create secret generic repo-inventra-gitops   --from-literal=type=git   --from-literal=url=git@github.com:InventraTech/inventra-gitops.git   --from-file=sshPrivateKey=argocd-deploy
kubectl -n argocd label secret repo-inventra-gitops argocd.argoproj.io/secret-type=repository
# guarde a chave privada no cofre da equipe e APAGUE os dois arquivos argocd-deploy*
```

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
REDIS_USERNAME REDIS_PASSWORD`.

```bash
kubectl create secret generic inventra-api-secrets -n inventra \
  --from-literal=DB_HOST=... --from-literal=JWT_SECRET=... \
  --dry-run=client -o yaml \
  | kubeseal --controller-namespace kube-system -o yaml \
  > apps/inventra-api/overlays/prod/sealed-secret-api.yaml
```

Pull secret do GHCR (PAT com `read:packages`):

```bash
kubectl create secret docker-registry ghcr-pull -n inventra \
  --docker-server=ghcr.io --docker-username=<usuario> --docker-password=<PAT> \
  --dry-run=client -o yaml \
  | kubeseal --controller-namespace kube-system -o yaml \
  > apps/inventra-api/overlays/prod/sealed-secret-ghcr.yaml
```

Depois descomente as linhas correspondentes em `overlays/prod/kustomization.yaml`.

## Pendências (TODO)

- [ ] **API:** liberar `/actuator/health/**` no `SecurityConfig` (hoje só `/actuator/health` é `permitAll`; as probes de liveness e readiness retornariam 401).
- [ ] **API:** trocar o CI para não usar banco e Redis reais.
- [ ] Gerar os SealedSecrets (acima) e descomentá-los no overlay.
- [ ] `CODEOWNERS`: trocar o dono individual por um time.
- [ ] Login do Argo via GitHub OAuth (Dex) e acesso de leitura para a equipe.
- [ ] Proteger a `main` (PR obrigatória + check `Validate`).
- [ ] M5: CD por tag abrindo a PR de bump aqui.
