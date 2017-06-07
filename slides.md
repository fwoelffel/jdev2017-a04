## Utilisation de conteneurs dans votre outil de CI
**Gitlab**

*JDEV2017 - Frédéric Woelffel*

---

## Moi ?

- Frédéric Woelffel
- Ingénieur R&D à l'IHU de Strasbourg
- Développeur Node / DevOps

![](./img/miaou.jpg) <!-- .element: class="fragment" -->

---

https://goo.gl/FxmiBe

(http://static.fwoelffel.me/JDEV2017/a04)

---

## Pré-requis

----

Binaires :
- docker 17.03
- git

----

Images Docker:
```
docker pull gitlab/gitlab-ce:9.3.0-ce.0
docker pull parabuzzle/craneoperator:2.1.3
docker pull registry:2
docker pull rancher/server:v1.6.2
docker pull rancher/agent:v1.2.2
```

----

Créons un réseau Docker pour cet atelier

```
docker network create --subnet=172.28.0.0/16 jdev
```

----

Mettons à jour la configuration du daemon Docker

```
sudo nano /etc/docker/daemon.json
```

```json
{
  ...
  "insecure-registries" : [ "172.28.0.0/16" ]
  ...
}
```

----

Redémarrons le daemon Docker

```
sudo service docker restart
```

---

## Installons Gitlab CE

----

Démarrons une instance de Gitlab CE

```
docker run \
  --detach \
  --publish 80:80 \
  --net jdev \
  --ip 172.28.0.10 \
  --name gitlab \
  --env GITLAB_OMNIBUS_CONFIG="external_url 'http://172.28.0.1/'" \
  gitlab/gitlab-ce:9.3.0-ce.0
```

----

- Rendons nous sur la page d'accueil de cette instance Gitlab http://127.0.0.1
- Définissons un mot de passe administrateur (password, c'est facile à retenir)
- Connectons nous
- Créons le groupe *jdev*

---

## Mettons en place un registre Docker privé

----

Démarrons une instance de registre

```
docker run \
  --detach \
  --publish 5000:5000 \
  --net jdev \
  --ip 172.28.0.11 \
  --name registry \
  registry:2
```

----

Vérifions qu'il est correctement exécuté

http://127.0.0.1:5000/v2/_catalog

La réponse attendue est la suivante

```json
{
  "repositories": []
}
```

---

## Avec une interface graphique c'est toujours plus sympa

----

Démarrons une instance d'interface graphique pour le registre Docker

```
docker run \
  --detach \
  --publish 5001:80 \
  --net jdev \
  --ip 172.28.0.12 \
  --env REGISTRY_HOST=172.28.0.11 \
  --env REGISTRY_PORT=5000 \
  --env REGISTRY_PROTO=http \
  --env REGISTRY_PUBLIC_URL=http://127.0.0.1:5000 \
  --name registry-ui \
  parabuzzle/craneoperator:2.1.3
```

----

Admirons

http://127.0.0.1:5001

---

## Préparons un Runner Gitlab-CI

----

Récupérons le code d'un Runner Gitlab

```
git clone https://github.com/FWoelffel/self-registering-gitlab-runner-docker.git
```

----

Construisons l'image Docker du Runner...

```
docker build -t my-gitlab-runner .
```

---

## Démarrons un Runner Gitlab-CI

----

Commençons par récupérer un token de Runner

http://127.0.0.1/admin/runners

----

Démarrons ensuite au moins une instance de Runner

```
docker run \
  --detach \
  --name gitlab-runner \
  --net jdev \
  --ip 172.28.0.13 \
  --env CI_SERVER_URL=http://172.28.0.10/ci \
  --env REGISTRATION_TOKEN=< votre token > \
  --env RUNNER_ENV="GIT_SSL_NO_VERIFY=1" \
  --env DOCKER_DISABLE_CACHE=true \
  --env DOCKER_IMAGE=true \
  --env CONCURRENCY=1 \
  --env REGISTER_RUN_UNTAGGED=true \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  my-gitlab-runner
```

----

Si tout s'est bien passé, nous devrions voir notre Runner enregistré auprès de Gitlab

http://127.0.0.1/admin/runners

---

## Créons une application

----

Commençons par instancier le projet *hello-world* sur Gitlab

----

Récupérons les sources d'une application sur Github

```
git clone https://github.com/FWoelffel/express-hello-world.git
```

----

Publions cette application sur notre instance de Gitlab

```
cd express-hello-world
git remote add jdev http://127.0.0.1/jdev/hello-world
git push --set-upstream jdev master
```

---

<!-- .slide: data-background="./img/minions_joy.gif" -->

## Mettons en place les scenarii de CI

----

Commençons par créer un fichier *.gitlab-ci.yml* dans notre projet *hello-world*

----

### Builder une image

```yaml
image: docker:17
services:
  - docker:17.03.1-ce-dind
variables:
  DOCKER_HOST: tcp://docker:2375
  CI_IMAGE_NAME: ${DOCKER_REGISTRY}/${CI_PROJECT_PATH_SLUG}:ci_${CI_COMMIT_SHA}
build:
  script:
    - docker build -t $CI_IMAGE_NAME .
```

----

C'est quoi ces variables ??

| Variable | Signification |
| --- | --- |
| CI_PROJECT_PATH_SLUG | Définie par Gitlab; Groupe et nom du projet |
| CI_COMMIT_SHA | Définie par Gitlab; Identifiant du commit |
| DOCKER_REGISTRY | A nous de la définir |

https://docs.gitlab.com/ee/ci/variables/

----

Définissons cette fameuse variable

http://127.0.0.1/jdev/hello-world/settings/ci_cd

`DOCKER_REGISTRY = 172.28.0.1:5000`

----

```
git add .gitlab-ci.yml
git commit -m "ci(gitlab): Add a build job"
git push
```

----

### Tester une image

```yaml
image: docker:17
services:
  - lordgaav/dind-options:latest
variables:
  DOCKER_HOST: tcp://lordgaav__dind-options:2375
  DOCKER_OPTS: --insecure-registry=${DOCKER_REGISTRY}
  CI_IMAGE_NAME: ${DOCKER_REGISTRY}/ci_${CI_PROJECT_PATH_SLUG}:${CI_COMMIT_SHA}
stages:
  - builds
  - tests
build:
  stage: builds
  script:
    - docker build -t $CI_IMAGE_NAME .
    - docker push $CI_IMAGE_NAME
test:
  stage: tests
  script:
    - docker run $CI_IMAGE_NAME test
```

----

```
git add .gitlab-ci.yml
git commit -m "ci(gitlab): Add stages and a test job"
git push
```

----

### Releasons une image

```yaml
image: docker:17
services:
  - lordgaav/dind-options:latest
variables:
  DOCKER_HOST: tcp://lordgaav__dind-options:2375
  DOCKER_OPTS: --insecure-registry=${DOCKER_REGISTRY}
  CI_IMAGE_NAME: ${DOCKER_REGISTRY}/${CI_PROJECT_PATH_SLUG}:ci_${CI_COMMIT_SHA}
  LATEST_IMAGE_NAME: ${DOCKER_REGISTRY}/${CI_PROJECT_PATH_SLUG}:latest
  TAGGED_IMAGE_NAME: ${DOCKER_REGISTRY}/${CI_PROJECT_PATH_SLUG}:${CI_COMMIT_TAG}
stages:
  - builds
  - tests
  - releases
build:
  stage: builds
  script:
    - docker build -t $CI_IMAGE_NAME .
    - docker push $CI_IMAGE_NAME
test:
  stage: tests
  script:
    - docker run $CI_IMAGE_NAME test
release:latest:
  stage: releases
  only:
    - master
  script:
    - docker pull $CI_IMAGE_NAME
    - docker tag $CI_IMAGE_NAME $LATEST_IMAGE_NAME
    - docker push $LATEST_IMAGE_NAME
release:tagged:
  stage: releases
  only:
    - tags
  script:
    - docker pull $CI_IMAGE_NAME
    - docker tag $CI_IMAGE_NAME $TAGGED_IMAGE_NAME
    - docker push $TAGGED_IMAGE_NAME
```

----

```
git add .gitlab-ci.yml
git commit -m "ci(gitlab): Add release jobs"
git push
```

----

Et pourquoi ne pas poser un tag ?
```
git tag 1.0.0
git push --tags
```

---

## Et si j'ai envie de déployer mon application ?

----

Ajoutons un fichier *docker-compose.yml* à notre projet

```yaml
version: '2'
services:
  hello-world:
    image: 172.28.0.1:5000/jdev-hello-world:latest
    ports:
      - 3000:3000
```

----

```
git add docker-compose.yml
git commit -m "chore(docker): Add a Docker orchestration file [skip ci]"
git push
```

---

## Installons un outil d'orchestration de conteneurs

----

Démarrons une instance de Rancher Server

```
docker run \
  --detach \
  --net jdev \
  --ip 172.28.0.14 \
  --publish 8080:8080 \
  --name rancher-server \
  rancher/server:v1.6.2
```

----

Vérifions que ce conteneur est bien démarré

http://127.0.0.1:8080

---

## Mettons en place une instance de Rancher Agent

----

Commençons par mettre à jour l'URL d'inscription des agents avec la valeur http://172.28.0.1:8080

----

Démarrons une instance de Rancher Agent sur le réseau par défaut de Docker

```
docker run \
  --detach\
  --rm \
  --env CATTLE_AGENT_IP="172.17.0.1"
  --volume /var/run/docker.sock:/var/run/docker.sock \
  --volume /var/lib/rancher:/var/lib/rancher \
  rancher/agent:v1.2.2 http://172.28.0.1:8080/v1/scripts/FEADA433891177754729:1483142400000:F3TVBX3X1LGIL9wHVoUnMMiQkYk
```

----

Nous disposons maintenant d'un esclave Rancher.

---

## Initialisons notre orchestration

----


Rendons-nous à l'adresse suivante et ajoutons une nouvelle stack.

http://127.0.0.1:8080/env/1a7/apps/stacks?which=all

----

Appelons cette stack *staging* et collons le contenu de notre fichier *docker-compose.yml* dans le bloc approprié.

---

## Et si j'ai envie que mon application se déploie automatiquement ?

----

Commençons par mettre à jour le fichier *.gitlab-ci.yml* du projet *hello-world*

```yaml
image: docker:17
services:
  - lordgaav/dind-options:latest
variables:
  DOCKER_HOST: tcp://lordgaav__dind-options:2375
  DOCKER_OPTS: --insecure-registry=${DOCKER_REGISTRY}
  CI_IMAGE_NAME: ${DOCKER_REGISTRY}/${CI_PROJECT_PATH_SLUG}:ci_${CI_COMMIT_SHA}
  LATEST_IMAGE_NAME: ${DOCKER_REGISTRY}/${CI_PROJECT_PATH_SLUG}:latest
  TAGGED_IMAGE_NAME: ${DOCKER_REGISTRY}/${CI_PROJECT_PATH_SLUG}:${CI_COMMIT_TAG}
stages:
  - builds
  - tests
  - releases
  - deployments
build:
  stage: builds
  script:
    - docker build -t $CI_IMAGE_NAME .
    - docker push $CI_IMAGE_NAME
test:
  stage: tests
  script:
    - docker run $CI_IMAGE_NAME test
release:latest:
  stage: releases
  only:
    - master
  script:
    - docker pull $CI_IMAGE_NAME
    - docker tag $CI_IMAGE_NAME $LATEST_IMAGE_NAME
    - docker push $LATEST_IMAGE_NAME
release:tagged:
  stage: releases
  only:
    - tags
  script:
    - docker pull $CI_IMAGE_NAME
    - docker tag $CI_IMAGE_NAME $TAGGED_IMAGE_NAME
    - docker push $TAGGED_IMAGE_NAME
deploy:staging:
  stage: deployments
  only:
    - master
  script:
    - docker run --rm --volume $(pwd):/mnt rancher/cli --env Default --url http://172.28.0.1:8080/v1 --access-key ${RANCHER_ACCESS_KEY} --secret-key ${RANCHER_SECRET_KEY} up -d --upgrade --force-upgrade --stack staging --pull hello-world
    - docker run --rm --volume $(pwd):/mnt rancher/cli --env Default --url http://172.28.0.1:8080/v1 --access-key ${RANCHER_ACCESS_KEY} --secret-key ${RANCHER_SECRET_KEY} up -d --confirm-upgrade --stack staging
```

----

Récupérons un accès à l'API de Rancher

http://127.0.0.1:8080/env/1a5/api/keys

----

Configurons ensuite les variables de la CI de ce projet

| Variable | Valeur |
| --- | --- |
| RANCHER_ACCESS_KEY | 6C42B5... |
| RANCHER_SECRET_KEY | 6sZt1UEM... |

----

Eeeeeeet soumettons nos changements

```
git add .gitlab-ci.yml
git commit -m "ci(gitlab): Add a staging deployment job"
git push
```

----

Des sceptiques ?

![](./img/suspicious.gif)

Modifions notre *Hello world!* (api.js)

----

```
git add api.js
git commit -m "feat: Change the hello world message"
git push
```

----

http://127.0.0.1:3000

---

<!-- .slide: data-background="./img/standing_ovation.gif" -->

---

## En résumé

----

- Installation de Gitlab CE
- Installation d'un registre Docker et de son interface graphique
- Installation d'un Runner Gitlab-CI
- Définition de scenarii de CI
- Mise en place d'un environnement d'intégration
- Configuration du déploiement automatique des services

---

### Utilisation de conteneurs dans votre outil de CI
**Gitlab**

*JDEV2017 - Frédéric Woelffel*

-----

frederic.woelffel@ihu-strasbourg.eu
