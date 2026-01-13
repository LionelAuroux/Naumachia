# Guide d'Installation Naumachia - Challenge "Stuck in the Middle"

## 📋 Prérequis

### Système
- **OS**: Ubuntu 22.04 LTS ou Debian 12 (recommandé)
- **RAM**: Minimum 4 Go
- **CPU**: 2 cores minimum
- **Stockage**: 20 Go minimum
- **Réseau**: Port UDP 1194 ouvert (OpenVPN), Port TCP 3960 (Registrar API)

### Logiciels
- Docker Engine 24.0+
- Docker Compose V2 (plugin)
- Python 3.10+
- Git

---

## 🚀 Installation Étape par Étape

### 1. Installation de Docker

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation des dépendances
sudo apt install -y ca-certificates curl gnupg lsb-release

# Ajout de la clé GPG Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajout du repository Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installation de Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Ajout de l'utilisateur au groupe docker
sudo usermod -aG docker $USER

# ⚠️ IMPORTANT: Déconnectez-vous et reconnectez-vous pour que les changements prennent effet
```

### 2. Installation des Drivers Réseau Docker (CRITIQUE)

Ces drivers sont **obligatoires** pour que le challenge MITM fonctionne. Ce sont des **remote drivers** libnetwork installés comme services système.

#### 2.1 Installation du driver l2bridge

Ce driver permet le fonctionnement Layer 2 complet (ARP spoofing possible).

```bash
# Télécharger le script de service SysV
sudo curl -L https://raw.githubusercontent.com/nategraf/l2bridge-driver/master/sysv.sh -o /etc/init.d/l2bridge
sudo chmod +x /etc/init.d/l2bridge

# Télécharger le binaire du driver
sudo curl -L https://github.com/nategraf/l2bridge-driver/releases/latest/download/l2bridge-driver.linux.amd64 -o /usr/local/bin/l2bridge
sudo chmod +x /usr/local/bin/l2bridge

# Activer et démarrer le service
sudo update-rc.d l2bridge defaults
sudo service l2bridge start

# Vérifier que le driver fonctionne
sudo stat /run/docker/plugins/l2bridge.sock
```

Vous devriez voir :
```
  File: /run/docker/plugins/l2bridge.sock
  Size: 0               Blocks: 0          IO Block: 4096   socket
  ...
```

#### 2.2 Installation du driver static-ipam

Ce driver permet les assignations IP statiques et les subnets overlapping.

```bash
# Télécharger le script de service SysV
sudo curl -L https://raw.githubusercontent.com/nategraf/static-ipam-driver/master/sysv.sh -o /etc/init.d/static-ipam
sudo chmod +x /etc/init.d/static-ipam

# Télécharger le binaire du driver
sudo curl -L https://github.com/nategraf/static-ipam-driver/releases/latest/download/static-ipam-driver.linux.amd64 -o /usr/local/bin/static-ipam
sudo chmod +x /usr/local/bin/static-ipam

# Activer et démarrer le service
sudo update-rc.d static-ipam defaults
sudo service static-ipam start

# Vérifier que le driver fonctionne
sudo stat /run/docker/plugins/static.sock
```

Vous devriez voir :
```
  File: /run/docker/plugins/static.sock
  Size: 0               Blocks: 0          IO Block: 4096   socket
  ...
```

#### 2.3 Vérification des deux drivers

```bash
# Les deux sockets doivent exister
ls -la /run/docker/plugins/
# Devrait afficher :
# l2bridge.sock
# static.sock

# Vérifier les services
sudo service l2bridge status
sudo service static-ipam status
```

#### 2.4 Installation avec systemd (Alternative pour Ubuntu 20.04+)

Si votre système utilise systemd plutôt que SysV init, créez des fichiers unit :

**Pour l2bridge :**
```bash
sudo tee /etc/systemd/system/l2bridge.service << 'EOF'
[Unit]
Description=L2Bridge Docker Network Driver
After=docker.service
Requires=docker.service

[Service]
Type=simple
ExecStart=/usr/local/bin/l2bridge
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable l2bridge
sudo systemctl start l2bridge
```

**Pour static-ipam :**
```bash
sudo tee /etc/systemd/system/static-ipam.service << 'EOF'
[Unit]
Description=Static IPAM Docker Network Driver
After=docker.service
Requires=docker.service

[Service]
Type=simple
ExecStart=/usr/local/bin/static-ipam
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable static-ipam
sudo systemctl start static-ipam
```

**Vérification systemd :**
```bash
sudo systemctl status l2bridge
sudo systemctl status static-ipam
```

### 3. Clonage et Configuration de Naumachia

```bash
# Clonage du repository (ou copie de l'archive)
cd /opt
sudo git clone https://github.com/VOTRE_REPO/naumachia-fork.git naumachia
sudo chown -R $USER:$USER naumachia
cd naumachia

# Installation des dépendances Python
pip3 install -r requirements.txt

# Configuration du mot de passe Redis (IMPORTANT: changez-le!)
nano .env
# Modifiez REDIS_PASSWORD avec un mot de passe fort

# Configuration du flag du challenge
# Modifiez CTF_FLAG avec votre propre flag
```

### 4. Configuration du Domaine/IP

Éditez `config.yml` et modifiez :

```yaml
# Remplacez par votre domaine ou IP publique
domain: ctf.votredomaine.com

challenges:
    middle:
        port: 1194  # Port UDP pour OpenVPN
```

### 5. Génération de la Configuration

```bash
# Exécution du script de configuration
python3 configure.py

# Ce script va :
# - Générer le docker-compose.yml principal
# - Créer les certificats PKI pour OpenVPN
# - Configurer les fichiers de challenge
```

### 6. Lancement de Naumachia

```bash
# Build des images Docker
docker compose build

# Lancement en mode détaché
docker compose up -d

# Vérification du statut
docker compose ps
```

Vous devriez voir tous les services en état "running":

- `openvpn-middle`
- `redis`
- `manager`
- `registrar`

### 7. Test de Fonctionnement

```bash
# Vérification des logs
docker compose logs -f openvpn-middle

# Test de l'API Registrar
curl http://localhost:3960/middle/list
```

---

## 🎮 Utilisation

### Génération d'une Configuration VPN pour un Joueur

```bash
# Via le CLI registrar
./registrar-cli middle add player1

# Récupération de la config
./registrar-cli middle get player1 > player1.ovpn
```

### Via l'API REST (pour intégration CTFd)

```bash
# Création d'un certificat
curl "http://localhost:3960/middle/add?cn=player1"

# Récupération de la config OpenVPN
curl "http://localhost:3960/middle/get?cn=player1" > player1.ovpn
```

### Connexion du Joueur

Le joueur télécharge son fichier `.ovpn` et se connecte :

```bash
# Linux/Mac
sudo openvpn --config player1.ovpn

# Windows: importer dans OpenVPN GUI
```

Une fois connecté, le joueur est sur le réseau 172.30.0.0/28 et peut :

- Scanner le réseau: `nmap -sn 172.30.0.0/28`
- Lancer l'attaque ARP: `arpspoof -i tap0 -t 172.30.0.3 172.30.0.2`
- Capturer le trafic: `tcpdump -i tap0 -w capture.pcap`

---

## 🔧 Intégration avec CTFd

### Installation du Plugin Naumachia

```bash
cd /path/to/ctfd/CTFd/plugins
git clone https://github.com/nategraf/ctfd-naumachia-plugin.git naumachia
```

### Configuration du Plugin

Éditez `CTFd/plugins/naumachia/config.py`:

```python
REGISTRAR_URL = "http://VOTRE_IP:3960"
# Si TLS activé:
# REGISTRAR_URL = "https://VOTRE_IP:3960"
# REGISTRAR_CA = "/path/to/ca.crt"
# REGISTRAR_CERT = "/path/to/client.crt"
# REGISTRAR_KEY = "/path/to/client.key"
```

### Création du Challenge dans CTFd

1. Admin → Challenges → New Challenge
2. Type: **Naumachia**
3. Name: `Stuck in the Middle`
4. Naumachia Challenge: `middle`
5. Description:
   ```
   Alice et Bob communiquent sur un réseau local.
   Interceptez leur conversation pour récupérer le flag.
   
   Indice: ARP spoofing pourrait être utile...
   
   Téléchargez votre configuration VPN ci-dessous.
   ```
6. Points: 200 (ajustez selon difficulté)
7. Flag: `flag{stuck_in_the_middle_with_you}`

---

## 🔒 Sécurité et Production

### Checklist Sécurité

- [ ] Changer `REDIS_PASSWORD` dans `.env`
- [ ] Activer TLS pour le registrar si exposé sur Internet
- [ ] Configurer un firewall (UFW/iptables)
- [ ] Limiter l'accès au port 3960 (registrar) au serveur CTFd uniquement
- [ ] Mettre en place des backups

### Configuration Firewall (UFW)

```bash
# OpenVPN (joueurs)
sudo ufw allow 1194/udp

# Registrar (uniquement depuis CTFd)
sudo ufw allow from IP_CTFD to any port 3960

# Activer le firewall
sudo ufw enable
```

### Monitoring

```bash
# Logs en temps réel
docker compose logs -f

# Logs d'un service spécifique
docker compose logs -f manager

# Statut des containers
docker compose ps

# Ressources utilisées
docker stats
```

---

## 🐛 Dépannage

### Les drivers réseau ne démarrent pas

```bash
# Vérifier que Docker est bien démarré
sudo systemctl status docker

# Vérifier les logs des drivers
sudo journalctl -u l2bridge -f
sudo journalctl -u static-ipam -f

# Vérifier que les binaires sont exécutables
ls -la /usr/local/bin/l2bridge
ls -la /usr/local/bin/static-ipam

# Redémarrer les services
sudo service l2bridge restart
sudo service static-ipam restart
# ou avec systemd :
sudo systemctl restart l2bridge
sudo systemctl restart static-ipam
```

### Les sockets des drivers n'existent pas

```bash
# Vérifier le dossier des plugins Docker
ls -la /run/docker/plugins/

# Si le dossier n'existe pas, le créer
sudo mkdir -p /run/docker/plugins/

# Redémarrer Docker puis les drivers
sudo systemctl restart docker
sudo service l2bridge restart
sudo service static-ipam restart
```

### Le challenge ne démarre pas

```bash
# Vérifier les logs du manager
docker compose logs manager

# Vérifier que les drivers sont actifs
sudo service l2bridge status
sudo service static-ipam status

# Vérifier les sockets
ls -la /run/docker/plugins/

# Reconstruire les images
docker compose build --no-cache
docker compose up -d
```

### Le joueur ne peut pas faire d'ARP spoofing

```bash
# Vérifier que le driver l2bridge est utilisé
docker network ls
docker network inspect <nom_du_reseau>

# Le driver doit être "l2bridge", pas "bridge"
# Exemple de sortie correcte :
#   "Driver": "l2bridge",
#   "IPAM": {
#       "Driver": "static",
```

### Erreur "network driver not found" ou "plugin not found"

```bash
# Les drivers doivent être démarrés AVANT de créer les réseaux
docker compose down

# Vérifier que les services tournent
sudo service l2bridge status
sudo service static-ipam status

# Redémarrer si nécessaire
sudo service l2bridge restart
sudo service static-ipam restart

# Attendre quelques secondes puis relancer
sleep 5
docker compose up -d
```

### Erreur au démarrage du driver (permission denied)

```bash
# Vérifier les permissions
sudo chmod +x /usr/local/bin/l2bridge
sudo chmod +x /usr/local/bin/static-ipam

# Vérifier que Docker peut accéder aux sockets
sudo ls -la /run/docker/plugins/
```

### Les containers du challenge ne communiquent pas

```bash
# Vérifier le réseau créé
docker network ls | grep middle
docker network inspect <project>_default

# Vérifier que les containers sont sur le bon réseau
docker inspect <container_id> | grep -A 20 Networks
```

---

## 📁 Structure des Fichiers

```
naumachia/
├── .env                    # Variables d'environnement (Redis, flags)
├── config.yml              # Configuration Naumachia
├── configure.py            # Script de génération
├── docker-compose.yml      # Généré par configure.py
├── challenges/
│   └── middle/
│       ├── docker-compose.yml
│       ├── alice/
│       │   ├── Dockerfile
│       │   └── alice.py
│       └── bob/
│           ├── Dockerfile
│           └── bob.py
├── manager/                # Orchestration des clusters
├── openvpn/                # Serveur VPN
├── registrar/              # API de gestion des certificats
├── redis/                  # Configuration Redis
└── templates/              # Templates Jinja2
```

---

## 📞 Support

En cas de problème:
1. Vérifiez les logs: `docker compose logs`
2. Consultez les issues GitHub du projet original: https://github.com/nategraf/Naumachia/issues
3. Discord Naumachia: https://discord.gg/gH9ZgeT

---

## 📚 Ressources

- [Naumachia Original](https://github.com/nategraf/Naumachia)
- [Naumachia Challenges](https://github.com/nategraf/Naumachia-challenges)
- [Plugin CTFd](https://github.com/nategraf/ctfd-naumachia-plugin)
- [l2bridge Driver](https://github.com/nategraf/l2bridge-driver) - Driver réseau Layer 2
- [Static IPAM Driver](https://github.com/nategraf/static-ipam-driver) - Driver IPAM statique

### Releases des drivers (binaires)
- [l2bridge releases](https://github.com/nategraf/l2bridge-driver/releases)
- [static-ipam releases](https://github.com/nategraf/static-ipam-driver/releases)
