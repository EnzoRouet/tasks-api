# TP3 : Docker et livraison continue

**Durée : environ 3 h 30** · Jour 3, après-midi

## Objectif

Mettre l'application dans une image Docker, publier automatiquement cette image à chaque changement,
créer une version (release) par un tag Git, et déployer avec une approbation avant la production.

> 📸 Ce TP contient 8 captures pour le dossier (TP3-1 à TP3-8).

## Avant de commencer

- [ ] **Docker Desktop est lancé** (icône verte). Test : `docker run --rm hello-world`
- [ ] `main` à jour : `git switch main && git pull`, puis `git switch -c tp3`
- [ ] Le TP2 est terminé (CI avec `lint` et `test`)

### Les commandes qui changent selon votre système

| Action                                 | macOS / Linux                                       | Windows (PowerShell)                                                |
| -------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------- |
| Appeler l'API                          | `curl http://localhost:8000/health`                 | `curl.exe http://localhost:8000/health`                             |
| Définir une variable pour une commande | `GITHUB_OWNER=moi IMAGE_TAG=0.1.0 docker compose …` | `$env:GITHUB_OWNER="moi"; $env:IMAGE_TAG="0.1.0"; docker compose …` |

---

## Partie 1 : Votre Dockerfile (40 min)

Ouvrez le `Dockerfile` : il contient 6 `TODO`. Complétez-le (aidez-vous du cours), puis :

```bash
docker build -t tasks-api:local .
docker run --rm -p 8000:8000 tasks-api:local
```

Dans un autre terminal : appelez `/health` et `/version`. Arrêtez avec `Ctrl + C`.

Contrôles :

- `docker images tasks-api` : notez la taille de l'image ;
- modifiez un fichier de `app/`, reconstruisez : quelles étapes sont « CACHED » ? Pourquoi `requirements.txt` est-il copié **avant** le code ?

> 📸 **TP3-1 :** le `docker build` réussi, puis `curl` sur `/health` qui répond `{"status":"ok"}`.

[docker build](../asset/images/TP3/TP3-1-1.png)

[curl health](../asset/images/TP3/TP3-1-2.png)

**Questions :** Différence entre une image et un conteneur ? À quoi sert le `.dockerignore` ?

Un image c'est un modèle qu'on utilise pour faire un conteneur, Donc le conteneur lui se sert du modèle pour tourner, il permet de faire comme un gitignore mais pour docker donc cacher des fichier a docker

---

## Partie 2 : Docker Compose (20 min)

Créez `docker-compose.yml` avec un service `api` qui :

- construit l'image du dossier courant ;
- publie le port 8000 ;
- définit la variable d'environnement `APP_VERSION` à `local`.

```bash
docker compose up --build
```

Vérifiez que `/version` répond `local`, puis `docker compose down`.

> 📸 **TP3-2 :** `docker compose up` en cours et le `curl` sur `/version`.

[docker compose up](../asset/images/TP3/TP3-2-1.png)

[curl version](../asset/images/TP3/TP3-2-2.png)

**Question :** Que gagne-t-on à décrire le lancement dans un fichier plutôt que dans une longue commande ?

Cela permet de le lancer plus facilement et que tout le monde puisse le lancer meme le mec qui connait pas les flags de base a mettre fin les options

---

## Partie 3 : Publier l'image depuis le pipeline (45 min)

Créez `.github/workflows/docker.yml` avec **deux jobs** :

1. `test` : checkout, Python, `pip install -r requirements-dev.txt`, `pytest` ;
2. `build` (après `test`, avec `needs: test`) : se connecte à GHCR, construit l'image et la **pousse**.

Aide, étape par étape :

- déclencheurs : `push` sur `main`, `push` de tags `v*`, et `workflow_dispatch` ;
- le job `build` demande la permission `packages: write` ;
- actions à utiliser : `docker/setup-buildx-action@v4`, `docker/login-action@v4` (registre `ghcr.io`,
  utilisateur `${{ github.actor }}`, mot de passe `${{ secrets.GITHUB_TOKEN }}`), `docker/metadata-action@v6`, `docker/build-push-action@v7` ;
- tags voulus : `type=sha,format=short` (ex. `sha-1a2b3c4`) et `type=semver,pattern={{version}}` pour les tags Git ;
- **version de l'application :** pour que `/version` réponde `0.1.0` dans l'image publiée, passez la version à la construction avec `build-args: APP_VERSION=…` dans `docker/build-push-action`.
  Calculez-la dans une étape : pour un tag `v0.1.0`, `${GITHUB_REF_NAME#v}` donne `0.1.0` ; pour un push sur `main`, utilisez par exemple `dev`.
  (Cela suppose que votre Dockerfile déclare `ARG APP_VERSION` et `ENV APP_VERSION`, voir le TODO 6.)
- **piège :** le nom d'une image GHCR doit être en **minuscules**. Calculez-le dans une étape : `${GITHUB_REPOSITORY_OWNER,,}`.

Poussez sur `main`, puis allez voir votre profil GitHub → **Packages**.

> 📸 **TP3-3 :** le run avec `test` puis `build` au vert.

[les run au vert](../asset/images/TP3/TP3-3.png)

> 📸 **TP3-4 :** la page du package `tasks-api` avec son tag `sha-…`.

[page du package](../asset/images/TP3/TP3-4.png)

**Questions :** Pourquoi `build` attend-il `test` ? Pourquoi éviter le tag `latest` en production ?

Car pour build il faut que les test passe sinon ca veut dire qu'il y a une erreur donc on build pas ca sert a rien, car on pourra pas revenir en arrière si on a pas le tag exact de la version d'avant et car on peut avoir deux serveur qui utilisent pas le même package car entre temps de déploiement l'un a pris le dernier push et l'autre non

---

## Partie 4 : Une release par un tag Git (30 min)

1. Sur GitHub, ouvrez votre package → **Package settings** → **Change visibility** → **Public** (nécessaire pour le jour 4).
2. Créez la version 0.1.0 :

```bash
git switch main && git pull
git tag v0.1.0
git push origin v0.1.0
```

3. Le pipeline se lance : l'image `0.1.0` doit apparaître dans vos packages.
4. Testez comme un utilisateur : **supprimez votre image locale**, puis lancez l'image publiée :

```bash
docker run --rm -p 8000:8000 ghcr.io/<votre-compte-minuscules>/tasks-api:0.1.0
```

> 📸 **TP3-5 :** la liste des tags du package (`0.1.0` et `sha-…`).

[liste des tags](../asset/images/TP3/TP3-5.png)

> 📸 **TP3-6 :** `docker run` de l'image **téléchargée depuis GHCR** et `curl /version` qui répond `0.1.0`.

[docker run](../asset/images/TP3/TP3-6-1.png)

[curl](../asset/images/TP3/TP3-6-2.png)

**Question :** Quel est l'avantage d'une image versionnée par rapport à « le code de ma machine » ?

Car ca permet d'avoir une version unifié qui fonctionne pour tout le monde et pas de ca marche sur ma machine

---

## Partie 5 : Environnements et approbation (25 min)

Ajoutez à `docker.yml` deux jobs de **déploiement simulé** :

- `deploy-staging` (après `build`, `environment: staging`) : se connecte à GHCR, lance l'image du commit
  (`docker run -d -p 8000:8000 …`), puis fait un **test de fumée** : `curl -f` sur `/health` et `/version` (avec quelques essais, le démarrage prend du temps) ;
- `deploy-production` (après `deploy-staging`, `environment: production`, **uniquement sur un tag** : `if: startsWith(github.ref, 'refs/tags/v')`) : même test de fumée avec l'image de la release.

Sur GitHub : **Settings → Environments → production** → cochez **Required reviewers** et ajoutez-vous.

Créez `v0.1.1` (nouveau tag) : le pipeline passe sur staging, puis **attend votre approbation** avant la production. Approuvez-le.

> 📸 **TP3-7 :** le run en attente d'approbation (« Waiting » / « Review deployments »), puis le run terminé.

[En attente](../asset/images/TP3/TP3-7-1.png)

[deploy](../asset/images/TP3/TP3-7-2.png)

**Questions :** À quoi sert un test de fumée ? Pourquoi demander une approbation humaine avant la production ?

Ca permet de vérifier direct si le serveur tourne et qu'il peut recevoir des requêtes, cela permet de vérifier que tout est bon et qu'une personne qui a les autorisation pour pousser sur la main valide pour la prod

---

## Partie 6 : Sécurité de l'image (10 min)

1. Vérifiez que votre conteneur n'est **pas root** : `docker run --rm tasks-api:local id` (vous devez voir un uid différent de 0).
2. Lisez ensemble le **cas d'étude** présenté par le formateur : en mars 2026, l'action de scan Trivy a été compromise.
   Quelles bonnes pratiques en tirer ? (épingler les versions, limiter les permissions, ne jamais exposer de secrets inutiles)

> 📸 **TP3-8 :** la sortie de la commande `id` dans le conteneur.

[id](../asset/images/TP3/TP3-8.png)

**Question :** Pourquoi ne pas exécuter une application en root dans un conteneur ?

Pour des raisons de sécurité, car un user a moins de privilèges qu'un admin donc si on pirate le compte bah on aura moins de problèmes

---

## Préparer demain (10 min)

Créez dès maintenant le mini-cluster du jour 4 pour gagner du temps (téléchargement long) : `kind create cluster --name gitops`, puis `kubectl get nodes`.
Si la commande échoue, notez l'erreur : on la réglera demain.

## Checklist de fin de TP

- [ ] Dockerfile complet, image construite et testée
- [ ] `docker-compose.yml` fonctionnel
- [ ] Image publiée sur GHCR par le pipeline, package public
- [ ] Release `v0.1.0` publiée et testée depuis GHCR
- [ ] Environnement `production` avec approbation
- [ ] Mes 8 captures et mes réponses sont dans le dossier
