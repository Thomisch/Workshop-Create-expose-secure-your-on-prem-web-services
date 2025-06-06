FROM ubuntu:22.04

# Mettre à jour le système et installer les outils de base + fail2ban
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y \
    curl \
    wget \
    vim \
    nano \
    net-tools \
    iputils-ping \
    openssh-server \
    sudo \
    htop \
    tree \
    unzip \
    git \
    fail2ban \
    rsyslog \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Créer un utilisateur non-root pour la sécurité
RUN useradd -m -s /bin/bash -G sudo secureuser && \
    echo 'secureuser:password123' | chpasswd

# Configurer SSH
RUN mkdir /var/run/sshd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

# Créer les fichiers de log nécessaires
RUN mkdir -p /var/log && \
    touch /var/log/auth.log && \
    touch /var/log/fail2ban.log && \
    chmod 640 /var/log/auth.log && \
    chmod 640 /var/log/fail2ban.log

# Configuration fail2ban simple (sans rsyslog au début)
RUN mkdir -p /etc/fail2ban && \
    mkdir -p /var/run/fail2ban && \
    echo '[DEFAULT]\nbantime = 600\nfindtime = 600\nmaxretry = 3\n\n[sshd]\nenabled = true\nport = ssh\nfilter = sshd\nlogpath = /var/log/auth.log\nmaxretry = 3' > /etc/fail2ban/jail.local

# Script de démarrage simplifié
RUN echo '#!/bin/bash\nset -e\n\n# Créer les fichiers de log si nécessaire\ntouch /var/log/auth.log\ntouch /var/log/fail2ban.log\n\n# Démarrer rsyslog en arrière-plan\necho "Starting rsyslog..."\n/usr/sbin/rsyslogd\nsleep 2\n\necho "Starting SSH..."\nservice ssh start\nsleep 2\n\necho "Starting fail2ban..."\nservice fail2ban start\nsleep 3\n\necho "All services started successfully"\n\nexec "$@"' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

# Créer un répertoire de travail
WORKDIR /home/secureuser

# Exposer le port SSH
EXPOSE 22

# Utiliser le script d'entrée
ENTRYPOINT ["/entrypoint.sh"]

# Commande par défaut
CMD ["tail", "-f", "/dev/null"]