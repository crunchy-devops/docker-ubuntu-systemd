Absolument. C'est une excellente approche lorsque le paquet système est obsolète. Installer `puppet-agent` via les `gem` de Ruby nous donne la dernière version, mais cela requiert une étape de compilation dans le `Dockerfile`.

Nous allons modifier notre `Dockerfile` précédent pour :
1.  Installer les outils de compilation et les dépendances de développement (`build-base`, `ruby-dev`).
2.  Installer `puppet` via `gem`.
3.  Supprimer les outils de compilation pour que l'image finale reste légère.
4.  Créer un script de service OpenRC personnalisé pour notre agent Puppet installé par `gem`.

---

### Fichiers Requis

Nous aurons besoin de 4 fichiers cette fois-ci :
1.  `Dockerfile` (modifié pour la compilation)
2.  `puppet.conf` (inchangé)
3.  `puppet.initd` (**nouveau** : le script de service pour OpenRC)
4.  `entrypoint.sh` (inchangé)

---

### 1. Dockerfile (avec compilation Gem)

Ce `Dockerfile` est plus complexe. Il installe temporairement des outils de build, compile la gem, puis nettoie ces outils dans la même commande `RUN` pour ne pas alourdir l'image.

```dockerfile
# Utiliser l'image de base Alpine
FROM alpine:latest

# Arguments pour la création de l'utilisateur SSH
ARG SSH_USER=devops
ARG SSH_PASSWORD=changeme

# Installe les dépendances de compilation et d'exécution en une seule commande
# puis nettoie les dépendances de compilation pour garder l'image légère.
RUN apk add --no-cache \
        # Dépendances d'exécution (runtime)
        bash \
        openrc \
        openssh \
        ruby \
        ruby-io-console \
        shadow && \
    # Dépendances de compilation (build-time) dans un paquet virtuel
    apk add --no-cache --virtual .build-deps \
        build-base \
        ruby-dev \
        libffi-dev && \
    # Installation de Puppet via Gem
    echo "Installation de Puppet via Gem..." && \
    gem install puppet --no-document && \
    # Suppression des dépendances de compilation
    echo "Nettoyage des dépendances de build..." && \
    apk del .build-deps

# --- Configuration du Serveur SSH (inchangé) ---
RUN ssh-keygen -A && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    adduser -D -s /bin/bash ${SSH_USER} && \
    echo "${SSH_USER}:${SSH_PASSWORD}" | chpasswd

# --- Configuration de l'agent Puppet (inchangé) ---
RUN mkdir -p /etc/puppetlabs/puppet
COPY puppet.conf /etc/puppetlabs/puppet/puppet.conf

# --- Configuration d'OpenRC (modifié) ---
RUN mkdir -p /run/openrc && touch /run/openrc/softlevel
# Copier notre nouveau script de service personnalisé pour puppet
COPY puppet.initd /etc/init.d/puppet
# Rendre le script de service exécutable
RUN chmod +x /etc/init.d/puppet
# Ajouter les services au runlevel par défaut
RUN rc-update add sshd default
RUN rc-update add puppet default

# --- Point d'entrée (inchangé) ---
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
EXPOSE 22
ENTRYPOINT ["/entrypoint.sh"]
```

---

### 2. puppet.initd (Nouveau script de service)

Puisque nous n'utilisons pas le paquet `apk` de Puppet, nous devons fournir notre propre script pour qu'OpenRC sache comment démarrer et arrêter l'agent.

```sh
#!/sbin/openrc-run
# Script de service OpenRC pour l'agent Puppet installé via Gem

# Le démon puppet va créer ce fichier PID
pidfile="/var/run/puppet/agent.pid"

# La commande à exécuter pour démarrer le service
command="/usr/bin/puppet"
command_args="agent --daemonize --logdest syslog"
command_background=true

depend() {
    need net
    after firewall
}
```

---

### 3. puppet.conf (Inchangé)

Ce fichier reste identique.

```ini
# /etc/puppetlabs/puppet/puppet.conf
[main]
server = puppet.example.com
runinterval = 30m
vardir = /var/lib/puppet
ssldir = $vardir/ssl
rundir = /var/run/puppet
logdir = /var/log/puppet
```

---

### 4. entrypoint.sh (Inchangé)

Ce script reste également identique, car il se contente d'appeler `rc-service` qui utilisera maintenant notre nouveau script `puppet.initd`.

```sh
#!/bin/sh
set -e
openrc sysinit
echo "Démarrage du service SSHD..."
rc-service sshd start
echo "Démarrage de l'agent Puppet..."
rc-service puppet start
echo "Services démarrés. Affichage des logs..."
tail -F /var/log/messages /var/log/puppet/agent.log
```

---

### Comment l'utiliser

La procédure est la même que précédemment :

1.  **Placez les 4 fichiers** dans un répertoire.
2.  **Construisez l'image** :
    ```bash
    docker build -t alpine-ssh-puppet-gem .
    ```
3.  **Lancez le conteneur** :
    ```bash
    docker run -d --name my-managed-node-gem -p 2222:22 alpine-ssh-puppet-gem
    ```

### Vérification

En plus des vérifications précédentes, vous pouvez maintenant vérifier la version de Puppet, qui devrait être beaucoup plus récente :

```bash
docker exec my-managed-node-gem puppet --version
```

Cette commande exécutera l'agent Puppet installé par `gem` et affichera sa version, vous confirmant que l'installation personnalisée a bien fonctionné.
