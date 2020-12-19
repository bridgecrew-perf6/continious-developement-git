# CONTINIOUS INTEGRATION AVEC GIT

Relou de ssh sur votre server distant pour git pull et restart apache2/nginx à chaque modif ? 
Deployer en un push un projet en configurant un git remote sur votre server distant !

## Prérequis


 - Avoir un projet pret à être deployer en local
 - Avoir un accès SSH sur un server distant

Mon projet d'exemple sera un projet Vue à deployer sur un server AWS ec2 avec Nginx
Il est possible d'avoir déjà un repository git existant sur votre projet

## 1. Création du dossier de production sur le server distant


```
ssh USER@SERVER_HOST
mkdir ~/DEPLOY-FOLDER

USER: Utilisateur de votre server distant
DEPLOY-FOLDER: Dossier que vous allez faire pointer sur votre nom de domaine
```

## 2. Création du repository dans le server distant


```
mkdir ~/BARE-DEPLOY-FOLDER
cd ~/BARE-DEPLOY-FOLDER
git -init --bare PROJECT-NAME.git

BARE-DEPLOY-FOLDER: Dossier dans lequel il contiendra uniquement votre dossier .git de votre projet
PROJECT-NAME: nom de votre projet
```

## 3. Configurer Nginx


Dans le cas de mon projet Vue on doit mettre en place un reverse proxy dans la configuration nginx sur le port 8080

```
server {
        server_name DOMAINE-NAME;
        location / {
                proxy_pass http://localhost:8080;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
}
```

## 4. Création du script de deploiment


Création du fichier "post-receive" dans le dossier hooks du PROJECT.git qui s'éxecutera à chaque push

```
sudo vi BARE-DEPLOY-FOLDER/PROJECT-NAME.git/hooks/post-receive
```

Le script à mettre pour la vue app  : 

```
#!/bin/bash
TARGET="DEPLOY-FOLDER"
GIT_DIR="BARE-REPOSITORY-FOLDER/PROJECT-NAME.git"
BRANCHE="BRANCH-TO-DEPLOY"
while read oldrev newrev ref
do
        if [[ $ref = refs/heads/$BRANCHE ]];
        then
                echo "Ref $ref received. Deploying ${BRANCHE} branch to preprod..."
                git --work-tree=$TARGET --git-dir=$GIT_DIR checkout $BRANCHE -f
                pm2 stop front
                cd $TARGET
                npm install
                npm run build
                pm2 start front

        else
                echo "Ref $ref received. Doing nothing: only the ${BRANCHE} branch may be deployed on this server."
        fi
done

DEPLOY-FOLDER: dossier de déploiement crée à l'étape 1
BARE-REPOSITORY-FOLDER: dossier du repository crée à l'étape 2
PROJECT-NAME: nom du projet crée à l'étape 2
BRANCH-TO-DEPLOY: la branch de déploiement (en prod par exemple: master)
```

Le script à mettre pour servir juste une app servie des fichiers  : 

```
#!/bin/bash
TARGET="DEPLOY-FOLDER"
GIT_DIR="BARE-REPOSITORY-FOLDER/PROJECT-NAME.git"
BRANCHE="BRANCH-TO-DEPLOY"
while read oldrev newrev ref
do
        if [[ $ref = refs/heads/$BRANCHE ]];
        then
                echo "Ref $ref received. Deploying ${BRANCHE} branch to preprod..."
                git --work-tree=$TARGET --git-dir=$GIT_DIR checkout $BRANCHE -f
                sudo chown www-data:www-data -R $TARGET

        else
                echo "Ref $ref received. Doing nothing: only the ${BRANCHE} branch may be deployed on this server."
        fi
done

DEPLOY-FOLDER: dossier de déploiement crée à l'étape 1
BARE-REPOSITORY-FOLDER: dossier du repository crée à l'étape 2
PROJECT-NAME: nom du projet crée à l'étape 2
BRANCH-TO-DEPLOY: la branch de déploiement (en prod par exemple: master)
```

## 5. Mise en place de l'app avec pm2


Etant donnée qu'une Vue app est lancée sur un port il est primordial d'utiliser pm2 pour faire tourner la Vue app en tache de fond afin de la servir avec Nginx
- Transférer ou cloner votre application dans votre server distant avec rsync ou un repository git déjà existant
- Lancer l'app en tache de fond 

```
cd DEPLOY-FOLDER
npm i 
npm run build
pm2 start "npm run serve" --name app
```

La Vue app tourne maintenant en production sur le domaine spécifier dans ma conf nginx
Il nous reste juste à ajouter notre git remote pour mettre en place le continious deploiment

## 6. Ajouter le remote du repository du server distant dans le projet local


Une fois le script mise en place dans le server distant, nous allons ajouter le git remote à notre projet local
Si votre server nécessite pas de clé ssh spécial et votre clé public est déjà enregistré dans votre server

```
 git remote add production USER@SERVER-HOST:/PATH/TO/BARE-DEPLOY-SERVER/PROJECT.git
```

Si votre server nécessite une clée ssh spécial comme une clé .pem avec AWS ec2, vous devez d'abord ajouter votre clée public dans vos authorized_keys avec cette commande :

```
cat ~/.ssh/id_?sa.pub | ssh -i awsKey.pem ubuntu@EC2-PUBLIC-IP "cat >> .ssh/authorized_keys"
```

Une fois l'accès ssh possible avec votre clé public enregisté, vous pouvez ajouter le git remote comme un server lambda :

```
git remote add prod USER@SERVER-HOST:/PATH/TO/BARE-DEPLOY-SERVER/PROJECT.git
```

## 6. Tenter de deployer

```
git push prod master
```


[source](https://gist.github.com/noelboss/3fe13927025b89757f8fb12e9066f2fa#file-git-deployment-md)