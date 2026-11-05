# TP4 : GitOps avec Argo CD

**Durée : environ 3 h** (+ QCM) · Jour 4, après-midi

## Objectif

Déployer l'application avec le GitOps : on ne touche plus jamais au cluster à la main, on modifie un dépôt Git
et Argo CD applique le changement. À la fin, un simple `git tag` déploie une nouvelle version dans le cluster.

> 📸 Ce TP contient 10 captures pour le dossier (TP4-1 à TP4-10).

## Avant de commencer

- [ ] Docker Desktop lancé, avec **au moins 4 Go de mémoire** (Settings → Resources)
- [ ] Votre image `ghcr.io/<compte>/tasks-api:0.1.0` existe et son package est **public**
- [ ] `kubectl version --client` et `kind version` répondent

### Les commandes qui changent selon votre système

| Action               | macOS / Linux                                                                                          | Windows (PowerShell)                                                                                                                                      |
| -------------------- | ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mot de passe Argo CD | `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" \| base64 -d` | `[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))` |
| Appeler l'API        | `curl http://localhost:8081/version`                                                                   | `curl.exe http://localhost:8081/version`                                                                                                                  |

---

## Partie 1 : Cluster et Argo CD (35 min)

```bash
kind create cluster --name gitops        # inutile si déjà fait hier
kubectl get nodes
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

Ouvrez l'interface : dans un terminal **qui reste ouvert**, lancez `kubectl port-forward svc/argocd-server -n argocd 8080:443`,
puis allez sur https://localhost:8080 (acceptez l'avertissement du certificat). Utilisateur : `admin`, mot de passe : voir le tableau ci-dessus.

> 📸 **TP4-1 :** `kubectl get nodes` (un nœud `Ready`).

[get nodes reponse](../asset/images/TP4/TP4-1.png)

> 📸 **TP4-2 :** `kubectl get pods -n argocd` (tous `Running`) et la page de connexion d'Argo CD.

[get pods](../asset/images/TP4/TP4-2-1.png)

[page de connexion Argo CD](../asset/images/TP4/TP4-2-2.png)

**Questions :** C'est quoi un pod ? Un deployment ? Pourquoi un `namespace` ?

Un pod c'est un conteneur éphémère qui tourne, un deployment c'est le fait de demandé combien de pods on veut, pour séparer les outils et les environnements

---

## Partie 2 : Le dépôt de configuration (30 min)

1. Créez un **nouveau dépôt GitHub public** nommé `tasks-config`.
2. Copiez-y le contenu du dossier `config-repo/` de votre projet (le dossier `k8s/` et le `README.md`).
3. Dans `k8s/deployment.yaml`, remplacez `<votre-compte-github>` par votre identifiant (minuscules).
4. Commitez et poussez sur `main`.
5. Dans votre projet applicatif, ouvrez `argocd/application.yaml`, remplacez aussi `<votre-compte-github>`, puis :

```bash
kubectl apply -f argocd/application.yaml
kubectl get pods -n tasks -w     # Ctrl + C pour quitter
```

6. Observez l'application dans l'interface Argo CD : elle doit devenir **Synced** et **Healthy**.
7. Accédez à l'API : `kubectl port-forward svc/tasks-api -n tasks 8081:80` (terminal ouvert), puis appelez `/version`.

> 📸 **TP4-3 :** l'application dans Argo CD, état Synced et Healthy, avec ses 2 pods.

[Argo CD avec les 2 pods](../asset/images/TP4/TP4-3.png)

> 📸 **TP4-4 :** la réponse de `/version` obtenue dans le cluster.

[reponse de l'api sur l'uri version](../asset/images/TP4/TP4-4.png)

**Question :** Pourquoi sépare-t-on le dépôt de l'application et le dépôt de configuration ?

Pour éviter de pouvoir modifier directement le nombre de replicas directement en commande et devoir faire dans git donc séparer les responsabilité

---

## Partie 3 : Le GitOps en action (25 min)

**A. Changer par Git.** Dans `tasks-config`, passez `replicas` de 2 à 3, commitez, poussez. Cliquez sur **Refresh** dans Argo CD
(ou attendez environ 3 minutes) : un 3ᵉ pod apparaît.

**B. Dérive.** Faites une modification **à la main** : `kubectl scale deployment tasks-api -n tasks --replicas=5`.
Observez : Argo CD détecte l'écart et **remet 3** (c'est `selfHeal`).

**C. Retour arrière.** Dans `tasks-config`, faites `git revert HEAD` (annule le dernier commit), poussez : le nombre de pods revient à 2.

> 📸 **TP4-5 :** le commit sur `replicas: 3` et les 3 pods dans Argo CD.

[Les 3 pods](../asset/images/TP4/TP4-5.png)

> 📸 **TP4-6 :** les pods juste après le `kubectl scale` à 5, puis après la correction automatique (deux captures).

[Les 5 pods](../asset/images/TP4/TP4-6-1.png)

[La correction automatique](../asset/images/TP4/TP4-6-2.png)

> 📸 **TP4-7 :** l'historique des commits de `tasks-config` avec le `revert`, et le retour à 2 pods.

[Historique commits](../asset/images/TP4/TP4-7.png)

**Questions :** Que signifie `selfHeal` ? Pourquoi le retour arrière est-il simple en GitOps ?

Ca signifie que tout run correctement, tout fonctionne ca répond et en accord avec ce qui est demandé, car il faut juste faire un revert pour revenir a la version antécédente

---

## Partie 4 : La chaîne complète (60 min)

Objectif : un tag `v0.2.0` sur le dépôt applicatif doit aboutir à la nouvelle version dans le cluster, sans aucune commande manuelle.

1. **Jeton d'accès.** GitHub → Settings → Developer settings → **Fine-grained personal access tokens** → _Generate new token_ :
   accès limité au **seul dépôt `tasks-config`**, permission **Contents : Read and write**. Copiez le jeton.
2. Dans le dépôt **applicatif** : Settings → Secrets and variables → Actions → nouveau secret `CONFIG_REPO_TOKEN` (collez le jeton).
3. Le dossier `scripts/` de votre projet contient déjà `set-image-tag.sh` : il remplace la version dans le manifeste, vous n'avez rien à écrire.
4. Ajoutez à `docker.yml` un job `update-config` (après `deploy-production`, uniquement sur tag) qui :
   - récupère le dépôt applicatif **et** le dépôt `tasks-config` (`actions/checkout` avec `repository`, `token: ${{ secrets.CONFIG_REPO_TOKEN }}` et `path: config`) ;
   - lance `bash scripts/set-image-tag.sh config/k8s/deployment.yaml "$VERSION"` ;
   - commite et pousse dans `tasks-config`.
     (Passez la version par une variable d'environnement `VERSION`, **jamais** en écrivant `${{ … }}` directement dans la commande.)
5. Créez le tag `v0.2.0`, approuvez la production, et suivez : pipeline → commit automatique dans `tasks-config` → Argo CD → nouveaux pods.
6. Vérifiez que `/version` répond `0.2.0`.

> 📸 **TP4-8 :** le run complet avec tous les jobs au vert, `update-config` compris.

[run de tous les jobs au vert](../asset/images/TP4/TP4-8.png)

> 📸 **TP4-9 :** le commit automatique dans `tasks-config` et Argo CD synchronisé sur `0.2.0`.

[commit automatique](../asset/images/TP4/TP4-9-1.png)

[Argo CD synchro](../asset/images/TP4/TP4-9-2.png)

> 📸 **TP4-10 :** `/version` qui répond `0.2.0`.

[reponse de l'api sur l'uri version](../asset/images/TP4/TP4-10.png)

**Questions :** Résumez en 5 lignes le chemin d'un `git tag` jusqu'au pod. Qui pousse (push) et qui tire (pull) ? Quelle différence avec un déploiement CI/CD « classique » ?

Le git tag est push par le dev sur le dépot applicatif

Actions lance les tests, publie le package avec le tag

le job update-config execute le .sh dans /scripts et commit et push sur le dépot config

Argo CD détecte le push et se réadapte en fonction de ce qui change

Argo CD se sync et remplace tout bien correctement

Github Actions push dans le dépot config et Argo CD pull

Ici le CI n'a pas de perm sur le cluster directement sinon dans le classique on kubectl apply de la CI

---

## QCM final (30 min)

Le formateur vous donne un lien : QCM individuel, sans documents. C'est la moitié de la note.

## Dossier à rendre

Complétez votre dossier (TP1 à TP4 + bilan) : à rendre le vendredi 6 novembre à 17 h. Voir `docs/DOSSIER.md`.

## Nettoyage (facultatif)

`kind delete cluster --name gitops`
