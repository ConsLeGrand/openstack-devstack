# Installation Openstack avec la methode Devstack
1. Prerequis:
- Ubuntu 24.04 propre (VM ou machine physique)
- Minimum recommandé :
    - 4 vCPU
    - 8 Go RAM (16 Go idéal)
    - 20–40 Go disque
- Deux (2) cartes réseau, dont une avec accès à Internet.

La première étape consiste à installer les prérequis nécessaires au bon fonctionnement de DevStack.
```bash
sudo apt update
sudo apt install snapd
sudo snap install hello-world
hello-world
sudo apt install git-all
git --version
```

2. Préparation de l'environment 
Nous allons maintenant préparer et configurer l’environnement système pour installer  DevStack

```bash
sudo useradd -s /bin/bash -d /opt/stack -m stack
sudo chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo -u stack -i
git clone https://opendev.org/openstack/devstack
cd devstack/
nano local.conf
```
Copiez et utilisez le contenu suivant dans le fichier local.conf

```yaml
[[local|localrc]]

ADMIN_PASSWORD=<definir-le-mot-de-passe>
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
HOST_IP=<ip interface choisi>
```
Dans le cas où vous souhaiteriez déployer les composants Swift, Ceilometer et Aodh, veuillez vous référer au contenu ci-dessous.

```yaml
[[local|localrc]]
ADMIN_PASSWORD=<votre mot de passe>

DATABASE_PASSWORD=$ADMIN_PASSWORD

RABBIT_PASSWORD=$ADMIN_PASSWORD

SERVICE_PASSWORD=$ADMIN_PASSWORD

HOST_IP=<ip machine>

# Activer Swift
enable_service s-proxy s-object s-container s-account

# Backend stockage objet
SWIFT_REPLICAS=1
SWIFT_HASH=swift
SWIFT_TEMPURL_KEY=$ADMIN_PASSWORD


# Enable Heat (Orchestration)
enable_plugin heat https://opendev.org/openstack/heat

# Enable Gnocchi (Metric Storage)
enable_plugin gnocchi https://github.com/gnocchixyz/gnocchi

# Enable Ceilometer (Data Collection) and Aodh (Alarming)
CEILOMETER_BACKENDS=gnocchi
enable_plugin ceilometer https://opendev.org/openstack/ceilometer
enable_plugin aodh https://opendev.org/openstack/aodh

# Optional: Add heat services specifically
enable_service h-eng h-api h-api-cfn h-api-cw
```
3. Installation 
Exécutez la commande suivante pour lancer l’installation.

```bash
./stack.sh
```
![Page acceuil OpenStack](./images/img1.png)
NB : En cas d’erreur durant l’installation, identifiez et installez les dépendances manquantes, puis relancez le script

4. Vérification de la version installer 
Exécutez la commande suivante pour vérifier la version installée.

```bash
openstack --version
```
Une fois l’installation terminée, les informations de connexion et les identifiants nécessaires sont fournis.
![Page acceuil OpenStack](./images/img2.png)
![Page acceuil OpenStack](./images/img3.png)

5. Commandes utiles:
- Relancer les services :
```bash
 ./rejoin-stack.sh
```
- Arrêter :
```bash
./unstack.sh
```
- Nettoyer complètement :
```bash
./clean.sh
```