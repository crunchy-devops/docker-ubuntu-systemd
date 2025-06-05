Parfait \! Utiliser Alpine avec son gestionnaire de services natif, **OpenRC**, est la bonne approche. C'est une méthode légère, sécurisée et parfaitement adaptée à la philosophie d'Alpine.

Voici comment créer une image Alpine qui utilise OpenRC pour gérer un service (nous prendrons `nginx` comme exemple). Le principe est d'utiliser un script d'entrée (`entrypoint.sh`) qui initialise OpenRC puis lance les services nécessaires.

-----

### Solution : Dockerfile pour Alpine avec OpenRC et Nginx

Nous allons créer 3 fichiers :

1.  `Dockerfile` : Pour construire l'image.
2.  `entrypoint.sh` : Le script qui sera exécuté au démarrage du conteneur.
3.  `nginx.conf` : Un fichier de configuration simple pour Nginx.

#### 1\. Fichier : `Dockerfile`

Ce fichier installe OpenRC et Nginx, configure le service, et définit le script d'entrée comme point de démarrage.

```dockerfile
# 1. Utiliser l'image de base Alpine
FROM alpine:latest

# 2. Installer OpenRC et Nginx sans utiliser le cache
RUN apk add --no-cache openrc nginx

# 3. Créer le répertoire et le fichier nécessaires au bon fonctionnement d'OpenRC
#    dans un environnement conteneurisé.
RUN mkdir -p /run/openrc && \
    touch /run/openrc/softlevel

# 4. Ajouter le service Nginx au runlevel "default" pour qu'OpenRC sache qu'il
#    doit le gérer.
RUN rc-update add nginx default

# 5. Copier notre configuration Nginx personnalisée
COPY nginx.conf /etc/nginx/http.d/default.conf

# 6. Copier et donner les permissions d'exécution à notre script d'entrée
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# 7. Exposer le port 80 pour Nginx
EXPOSE 80

# 8. Définir le script d'entrée comme commande de démarrage du conteneur
ENTRYPOINT ["/entrypoint.sh"]
```

#### 2\. Fichier : `entrypoint.sh`

Ce script est le cœur de notre conteneur. Il initialise OpenRC, démarre Nginx, puis maintient le conteneur en vie en affichant les logs.

```sh
#!/bin/sh

# Sortir immédiatement si une commande échoue
set -e

# Démarrer les services système de base gérés par OpenRC.
# C'est l'équivalent d'une séquence de démarrage système.
openrc sysinit

# Démarrer le service nginx en utilisant OpenRC.
# Le service va se lancer en arrière-plan (daemonize).
echo "Démarrage du service Nginx via OpenRC..."
rc-service nginx start

# Commande finale pour garder le conteneur en cours d'exécution
# Nous affichons les logs de Nginx, ce qui est une bonne pratique
# pour voir ce qu'il se passe et pour que le conteneur ne s'arrête pas.
echo "Conteneur prêt. Affichage des logs de Nginx..."
tail -F /var/log/nginx/access.log /var/log/nginx/error.log
```

#### 3\. Fichier : `nginx.conf`

Une configuration Nginx très simple pour prouver que tout fonctionne.

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    root /var/www/localhost/htdocs;

    location / {
        try_files $uri $uri/ =404;
    }

    # Page d'accueil personnalisée
    location = / {
        default_type text/html;
        return 200 "<html><body style='font-family: sans-serif;'><h1>Bonjour depuis Alpine, Nginx et OpenRC !</h1></body></html>";
    }
}
```

-----

### Comment l'utiliser

1.  **Placez les 3 fichiers** (`Dockerfile`, `entrypoint.sh`, `nginx.conf`) dans le même répertoire.

2.  **Construisez l'image Docker** :
    Ouvrez un terminal dans ce répertoire et exécutez :

    ```bash
    docker build -t alpine-openrc .
    ```

3.  **Lancez un conteneur à partir de l'image** :
    Cette commande lance le conteneur en arrière-plan (`-d`) et mappe le port 80 du conteneur au port 8080 de votre machine hôte (`-p 8080:80`).

    ```bash
    docker run -d --name mon-alpine-nginx -p 8080:80 alpine-openrc
    ```

    Notez que, contrairement à `systemd`, nous n'avons **pas besoin de l'option `--privileged`**, ce qui rend cette approche plus sécurisée.

4.  **Vérifiez le résultat** :

      * **Dans votre navigateur** : Ouvrez [http://localhost:8080](https://www.google.com/search?q=http://localhost:8080). Vous devriez voir le message "Bonjour depuis Alpine, Nginx et OpenRC \!".
      * **Pour voir les logs** (ce que notre script `tail -F` affiche) :
        ```bash
        docker logs -f mon-alpine-nginx
        ```
        Vous verrez la sortie de votre script d'entrée.

Ce modèle est très efficace et réutilisable. Vous pouvez ajouter d'autres services (comme `php-fpm`, une base de données, etc.) en les installant via `apk`, en les ajoutant au runlevel avec `rc-update`, et en les démarrant dans votre `entrypoint.sh` avec `rc-service <nom_du_service> start`.
