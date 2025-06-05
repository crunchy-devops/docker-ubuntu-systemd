Oui, c'est techniquement possible, mais c'est généralement considéré comme une **mauvaise pratique** pour tout ce qui n'est pas du débogage temporaire.

Laissez-moi vous expliquer le "comment", puis le "pourquoi il ne faut pas le faire en production", et enfin la "bonne méthode".

Pour l'exemple, je vais supposer que "quandary" est un service hypothétique. Comme il n'existe pas dans les dépôts Alpine, je vais utiliser un vrai service simple à la place : `cronie` (le démon `cron` qui permet de planifier des tâches).

### 1. La méthode "en direct" (pour le débogage)

Vous pouvez utiliser `docker exec` pour obtenir un shell à l'intérieur de votre conteneur déjà actif et y effectuer des opérations.

```bash
# Nom du conteneur que nous avons lancé précédemment
CONTAINER_NAME="mon-alpine-nginx"

# 1. Se connecter au conteneur en mode interactif
docker exec -it $CONTAINER_NAME /bin/sh

# --- Vous êtes maintenant à l'intérieur du conteneur ---

# 2. Mettre à jour la liste des paquets et installer le nouveau service
#    (remplacez 'cronie' par le nom de votre paquet)
apk update
apk add cronie

# 3. Ajouter le nouveau service à OpenRC pour qu'il le connaisse
rc-update add cronie default

# 4. Démarrer le service manuellement
rc-service cronie start

# 5. Vérifier que les services (nginx et cronie) sont bien démarrés
rc-status

# Pour quitter le shell du conteneur, tapez "exit"
exit
```

À ce stade, le service `cronie` est installé et actif **uniquement dans cette instance spécifique du conteneur**.

### 2. Le problème : Les changements sont éphémères

C'est le point le plus important. Les conteneurs Docker sont conçus pour être **immuables et éphémères**.

* **Perte des données :** Si vous arrêtez et supprimez ce conteneur (`docker stop mon-alpine-nginx` puis `docker rm mon-alpine-nginx`), toutes les modifications que vous venez de faire (l'installation de `cronie`) seront **définitivement perdues**.
* **Non reproductible :** Si vous lancez un nouveau conteneur à partir de votre image `alpine-openrc`, il n'aura que `nginx`, car `cronie` n'a jamais été ajouté à l'image elle-même (le "plan de construction").

Utiliser `docker exec` pour modifier un conteneur en production, c'est comme prendre des notes sur un post-it qui sera jeté à la fin de la journée.

### 3. La bonne méthode : Mettre à jour le `Dockerfile`

La seule façon correcte et pérenne d'ajouter un service est de modifier la "source" de votre conteneur : le `Dockerfile` et les scripts associés.

Voici comment vous devriez procéder pour ajouter `cronie` de manière permanente :

**1. Mettez à jour votre `Dockerfile` :**

```dockerfile
# ... (début du Dockerfile) ...

# 2. Installer OpenRC, Nginx ET notre nouveau service cronie
RUN apk add --no-cache openrc nginx cronie

# 3. Créer les fichiers nécessaires pour OpenRC
RUN mkdir -p /run/openrc && \
    touch /run/openrc/softlevel

# 4. Ajouter Nginx ET cronie au runlevel "default"
RUN rc-update add nginx default
RUN rc-update add cronie default

# ... (reste du Dockerfile) ...
```

**2. Mettez à jour votre `entrypoint.sh` :**

```sh
#!/bin/sh
set -e
openrc sysinit

echo "Démarrage du service Nginx via OpenRC..."
rc-service nginx start

echo "Démarrage du service Cronie via OpenRC..."
rc-service cronie start

echo "Conteneur prêt. Affichage des logs de Nginx..."
tail -F /var/log/nginx/access.log /var/log/nginx/error.log
```

**3. Reconstruisez votre image et relancez le conteneur :**

```bash
# 1. Reconstruire l'image (vous pouvez lui donner un nouveau tag, ex: v2)
docker build -t alpine-openrc:v2 .

# 2. Arrêter et supprimer l'ancien conteneur
docker stop mon-alpine-nginx
docker rm mon-alpine-nginx

# 3. Lancer un nouveau conteneur à partir de la nouvelle image
docker run -d --name mon-alpine-nginx-v2 -p 8080:80 alpine-openrc:v2
```

Désormais, **tous les conteneurs** que vous créerez à partir de l'image `alpine-openrc:v2` contiendront et lanceront `nginx` et `cronie` de manière fiable et reproductible.

### Conclusion

| Méthode | Quand l'utiliser ? | Avantages | Inconvénients |
| :--- | :--- | :--- | :--- |
| **`docker exec`** | Pour du **débogage rapide et temporaire**. | Rapide, pas besoin de reconstruire. | **NON PERMANENT**, non reproductible, mauvaise pratique en production. |
| **Modifier le `Dockerfile`** | **TOUJOURS** pour ajouter/modifier des services de manière permanente. | **Permanent**, fiable, reproductible, suit les bonnes pratiques "Infrastructure as Code". | Nécessite de reconstruire l'image et de redéployer le conteneur. |
