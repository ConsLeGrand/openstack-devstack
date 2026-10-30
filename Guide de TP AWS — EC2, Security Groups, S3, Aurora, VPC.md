# Guide de TP AWS — EC2, Security Groups, S3, Aurora, VPC

Oct 30, 2026 · @sihum

## Introduction

Ce guide permet de refaire seul les 7 TP vus en séance : EC2, groupes de sécurité, user data, S3, Aurora et VPC. Chaque TP suit le même schéma : **objectif → étapes → capture → test attendu**.

**Prérequis**

- Un compte AWS (ou AWS Academy Learner Lab) avec accès à la console.
- Une région fixe pour tout le TP : par exemple `us-east-1` (Learner Lab) ou `eu-west-3` (Paris). Tout se crée dans la même région.
- Un terminal : PowerShell / Windows Terminal (Windows), Terminal (macOS/Linux). `ssh` et `curl` sont inclus.
- Une paire de clés `.pem` (créée au TP1).

**Conventions**

- `<IP_PUBLIQUE>` = adresse IPv4 publique de l'instance (onglet *Détails* de l'instance).
- `<IP_PRIVEE>` = adresse IPv4 privée (ex. `172.31.x.x`).
- Les blocs gris sont des commandes à taper. Les captures sont des schémas de la console : les libellés peuvent légèrement varier selon la version de la console.
- Type d'instance : `t2.micro` ou `t3.micro` (éligible offre gratuite).

## TP1 — Créer une instance Amazon Linux

**Objectif :** lancer une machine virtuelle EC2 sous Amazon Linux 2023 et récupérer son IP publique. Une instance = un serveur loué à la seconde ; l'AMI est le modèle du disque système.

**Étapes**

1. Console → **EC2** → **Instances** → **Lancer une instance**.
2. Nom : `tp1-amazon-linux` (repère 1). AMI : **Amazon Linux 2023** (repère 2).
3. Type : `t2.micro` (ou `t3.micro`).
4. Paire de clés → **Créer une paire de clés** : nom `cle-tp`, type RSA, format **.pem**. Le fichier se télécharge : le ranger dans un dossier `TP`.
5. Paramètres réseau : laisser par défaut (VPC par défaut, IP publique activée, groupe `launch-wizard-1` avec SSH).
6. **Lancer l'instance** → **Afficher toutes les instances**.
7. Attendre l'état **En cours d'exécution** et **2/2 vérifications réussies**, puis noter l'**IPv4 publique**.

&#91;image: Nom de l'instance et choix de l'AMI Amazon Linux 2023\]

&#91;image: Type d'instance et création de la paire de clés .pem\]

&#91;image: Instance en cours d'exécution et IP publique à noter\]

**Résultat attendu :** l'instance est *running*, avec une IP publique (ex. `54.210.33.17`) et une IP privée `172.31.x.x`.

Préparer la clé une fois pour toutes :

```bash
# macOS / Linux
chmod 400 cle-tp.pem

# Windows (PowerShell, dans le dossier de la clé)
icacls .\cle-tp.pem /inheritance:r
icacls .\cle-tp.pem /grant:r "$($env:USERNAME):(R)"
```

## TP2 — Groupe de sécurité : tout bloquer, puis ouvrir ping et SSH

**Objectif :** prouver qu'un groupe de sécurité (SG) est un pare-feu « liste blanche » : sans règle entrante, rien n'entre. Le SG est **stateful** : la réponse à un trafic autorisé ressort automatiquement.

### Partie A — aucune règle entrante

1. **EC2** → **Groupes de sécurité** → **Créer un groupe de sécurité**.
2. Nom `sg-tp2`, VPC par défaut. **Ne rien ajouter** dans les règles entrantes. Laisser la règle sortante par défaut → **Créer** (repères 1-2).
3. Attacher le SG à l'instance du TP1 : sélectionner l'instance → **Actions** → **Sécurité** → **Modifier les groupes de sécurité** → ajouter `sg-tp2`, **retirer** `launch-wizard-1` → **Enregistrer**.

&#91;image: Groupe de sécurité créé sans aucune règle entrante\]

&#91;image: Remplacer launch-wizard-1 par sg-tp2 sur l'instance\]

4. Tester depuis votre poste :

```bash
ping <IP_PUBLIQUE>
ssh -i cle-tp.pem ec2-user@<IP_PUBLIQUE>
```

&#91;image: Ping et SSH en échec : délai dépassé\]

**Résultat attendu :** ping 100 % de perte, SSH `Connection timed out`. Le SG jette les paquets sans répondre, d'où un *timeout* et non un refus.

### Partie B — autoriser ping et SSH

1. `sg-tp2` → onglet **Règles entrantes** → **Modifier les règles entrantes**.
2. Ajouter : **SSH** (TCP 22) et **Tout ICMP - IPv4**, source **Mon IP** (ou `0.0.0.0/0` en salle) → **Enregistrer**.
3. Relancer les mêmes commandes. La règle s'applique immédiatement, sans redémarrage.

&#91;image: Règles entrantes SSH et ICMP\]

&#91;image: Ping et SSH réussis\]

**Résultat attendu :** ping 0 % de perte, invite `[ec2-user@ip-172-31-… ~]$`. Taper `exit` pour quitter.

## TP3 — Serveur web nginx par user data, HTTP bloqué puis autorisé

**Objectif :** automatiser l'installation d'un serveur au démarrage (user data) et vérifier que le port 80 dépend du SG. Le script user data s'exécute **une seule fois**, en root, au premier boot.

**Étapes**

1. Créer d'abord un SG `sg-web` avec **uniquement SSH (22)** en entrant.
2. **Lancer une instance** : nom `tp3-web`, Amazon Linux 2023, `t2.micro`, clé `cle-tp`, groupe existant `sg-web`.
3. Dérouler **Détails avancés** tout en bas → champ **Données utilisateur**, coller :

```bash
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable --now nginx
echo "Test server Web" > /usr/share/nginx/html/index.html
```

4. **Lancer l'instance**, attendre 2 à 3 minutes (le temps que le script tourne).

&#91;image: Champ Données utilisateur et SG sans HTTP\]

### Test 1 — HTTP bloqué

Ouvrir `http://<IP_PUBLIQUE>` dans le navigateur (bien **http**, pas https).

&#91;image: Navigateur : site inaccessible, délai dépassé\]

**Résultat attendu :** `ERR_CONNECTION_TIMED_OUT`. nginx tourne, mais le SG bloque le port 80.

### Test 2 — HTTP autorisé

1. `sg-web` → **Modifier les règles entrantes** → **Ajouter une règle** : **HTTP** (TCP 80), source **N'importe où-IPv4** `0.0.0.0/0` → **Enregistrer**.
2. Rafraîchir la page.

&#91;image: Ajout de la règle HTTP 80\]

&#91;image: Page « Test server Web » affichée\]

**Résultat attendu :** la page affiche `Test server Web`.

**Si la page reste vide ou en erreur** après ouverture du port, vérifier le script en SSH :

```bash
sudo systemctl status nginx
sudo cat /var/log/cloud-init-output.log | tail -20
curl localhost
```

## TP4 — Deux instances, deux groupes de sécurité

**Objectif :** contrôler le trafic **entre instances** du même VPC. Le SG de la cible décide : on l'ouvre en prenant comme source le SG de l'autre instance (ou le CIDR du réseau).

&#91;image: Architecture du TP4 : instance A (sg-A) vers instance B (sg-B)\]

**Préparation**

1. Créer `sg-A` : entrant **SSH (22)** depuis Mon IP.
2. Créer `sg-B` : entrant **SSH (22)** depuis Mon IP seulement (pas d'ICMP, pas d'HTTP).
3. Lancer **Instance A** (`tp4-A`) avec `sg-A`.
4. Lancer **Instance B** (`tp4-B`) avec `sg-B` et le même user data nginx qu'au TP3.
5. Noter l'**IP privée** de B (ex. `172.31.30.12`).
6. Se connecter en SSH sur **A** : tous les tests partent de A vers l'IP privée de B.

### Test 1 — ping et HTTP bloqués

```bash
ping -c 4 <IP_PRIVEE_B>
curl -m 5 http://<IP_PRIVEE_B>
```

&#91;image: Depuis A : ping 100 % perdu et curl en timeout\]

**Résultat attendu :** 100 % de perte, `curl: (28) Connection timed out`. Même dans le même réseau, sg-B bloque.

### Test 2 — ping et HTTP autorisés depuis A

1. `sg-B` → **Modifier les règles entrantes** → ajouter :
   - **Tout ICMP - IPv4**, source **Personnalisé** → taper `sg-` et choisir `sg-A`.
   - **HTTP** (80), source `sg-A`.
2. **Enregistrer**, puis relancer les deux commandes depuis A.

&#91;image: Règles de sg-B avec sg-A comme source\]

&#91;image: Depuis A : ping et curl réussis\]

**Résultat attendu :** ping 0 % de perte et `curl` renvoie `Test server Web`.

À retenir : source `sg-A` = « toute instance qui porte sg-A », même si son IP change. Source `172.31.0.0/16` = « tout le VPC ». La première est plus restrictive, donc préférable.

## TP5 — Bucket S3 pour site web statique

**Objectif :** héberger un site HTML sans serveur. Activer l'hébergement ne suffit pas : il faut une **stratégie de compartiment** qui autorise `s3:GetObject` au public.

**Fichier à préparer** — `index.html` sur votre poste :

```html
<!DOCTYPE html>
<html lang="fr">
<head><meta charset="utf-8"><title>TP5</title></head>
<body>
  <h1>Bienvenue sur mon site statique</h1>
  <p>Hébergé sur Amazon S3 — TP5.</p>
</body>
</html>
```

**Étapes**

1. **S3** → **Créer un compartiment**. Nom unique au monde, ex. `site-tp-<prenom>-2026`.
2. **Décocher** « Bloquer tous les accès publics » et cocher la case d'avertissement → **Créer**.
3. Ouvrir le bucket → **Charger** → ajouter `index.html` → **Charger**.
4. Onglet **Propriétés** → tout en bas **Hébergement de site Web statique** → **Modifier** → **Activer**, document d'index `index.html` → **Enregistrer**.
5. Copier le **point de terminaison** affiché en bas de l'onglet Propriétés.

&#91;image: Création du bucket avec accès public débloqué\]

&#91;image: Activation de l'hébergement statique et point de terminaison\]

### Test 1 — sans permission s3:GetObject

Ouvrir le point de terminaison dans le navigateur.

&#91;image: Erreur 403 Forbidden AccessDenied\]

**Résultat attendu :** `403 Forbidden — AccessDenied`. Le bucket n'est plus *bloqué*, mais rien n'est encore *autorisé*.

### Test 2 — avec permission s3:GetObject

1. Onglet **Autorisations** → **Stratégie de compartiment** → **Modifier**, coller (remplacer le nom du bucket) :

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LectureSitePublic",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::NOM-DE-VOTRE-BUCKET/*"
    }
  ]
}
```

2. **Enregistrer les modifications**, puis rafraîchir la page du site.

&#91;image: Stratégie de compartiment autorisant s3:GetObject\]

&#91;image: Site statique affiché\]

**Résultat attendu :** la page s'affiche. Erreur fréquente : oublier `/*` à la fin de l'ARN → la policy est refusée ou le 403 persiste.

## TP6 — Base de données Aurora (MySQL) et test de connexion

**Objectif :** créer un cluster Aurora compatible MySQL et s'y connecter depuis une instance EC2 du même VPC. Aurora est une base **managée** : AWS gère stockage, sauvegardes et réplication ; on ne se connecte jamais en SSH à la base.

> Aurora n'est **pas** dans l'offre gratuite : supprimer le cluster en fin de séance.

**Étapes**

1. **Aurora and RDS** → **Bases de données** → **Créer une base de données**.
2. **Création standard** → moteur **Aurora (compatible MySQL)** → modèle **Développement/Test**.
3. Identifiant du cluster `tp-aurora`, utilisateur `admin`, gestion **Autogéré**, saisir un mot de passe (le noter).
4. Classe d'instance : la plus petite proposée (ex. `db.t3.medium`), ou **Serverless v2** si disponible.
5. **Ne pas créer de réplica Aurora**.
6. Connectivité → **Se connecter à une ressource de calcul EC2** → choisir l'instance du TP1. Accès public : **Non**.
7. **Créer la base de données** et attendre l'état **Disponible** (10 à 15 min).
8. Ouvrir `tp-aurora` → **Connectivité et sécurité** → copier le point de terminaison **Écrivain**.

&#91;image: Choix du moteur Aurora MySQL et identifiants\]

&#91;image: Classe d'instance et connexion à la ressource EC2\]

&#91;image: Cluster disponible et point de terminaison écrivain\]

L'option « Se connecter à une ressource de calcul EC2 » crée deux SG liés : le port **3306** de la base est ouvert uniquement depuis l'instance choisie.

### Test — connexion et requêtes

En SSH sur l'instance EC2 :

```bash
sudo dnf install -y mariadb105
mysql -h <ENDPOINT_ECRIVAIN> -u admin -p
```

Puis dans le client MySQL :

```sql
CREATE DATABASE ecole;
USE ecole;
CREATE TABLE etudiant (id INT PRIMARY KEY AUTO_INCREMENT, nom VARCHAR(50));
INSERT INTO etudiant (nom) VALUES ('Awa'), ('Moussa');
SELECT * FROM etudiant;
```

&#91;image: Connexion à Aurora et résultat du SELECT\]

**Résultat attendu :** le `SELECT` renvoie 2 lignes. Si la connexion reste bloquée, c'est le SG de la base : vérifier que la règle 3306 a bien pour source le SG de l'instance.

## TP7 — Créer une VPC : sous-réseaux, passerelle, routage, instance

**Objectif :** construire à la main le réseau que le VPC par défaut fournit tout fait. Un sous-réseau est **public** uniquement si sa table de routage envoie `0.0.0.0/0` vers une passerelle Internet.

&#91;image: Architecture du TP7 : vpc-tp7 avec sous-réseau public et privé\]

### Étape 1 — la VPC

**VPC** → **Vos VPC** → **Créer un VPC** → **VPC uniquement**, nom `vpc-tp7`, CIDR `10.0.0.0/16` → **Créer**.

&#91;image: Création de vpc-tp7 en 10.0.0.0/16\]

### Étape 2 — les sous-réseaux

1. **Sous-réseaux** → **Créer un sous-réseau** → VPC `vpc-tp7`.
2. `subnet-public` : `10.0.1.0/24`. **Ajouter un nouveau sous-réseau** → `subnet-prive` : `10.0.2.0/24` → **Créer**.
3. Sélectionner `subnet-public` → **Actions** → **Modifier les paramètres du sous-réseau** → cocher **Activer l'attribution automatique d'adresses IPv4 publiques**.

&#91;image: Création des deux sous-réseaux et IP publique automatique\]

### Étape 3 — la passerelle Internet

**Passerelles Internet** → **Créer** (`igw-tp7`) → **Actions** → **Attacher au VPC** → `vpc-tp7`. État attendu : **Attached**.

&#91;image: Attachement de igw-tp7 à vpc-tp7\]

### Étape 4 — le routeur (table de routage)

1. **Tables de routage** → **Créer une table de routage** : `rt-public`, VPC `vpc-tp7`.
2. Onglet **Routes** → **Modifier les routes** → **Ajouter une route** : `0.0.0.0/0` → **Passerelle Internet** `igw-tp7` → **Enregistrer**.
3. Onglet **Associations de sous-réseau** → **Modifier** → cocher `subnet-public` → **Enregistrer**.

`subnet-prive` reste sur la table principale (route `local` seule) : il est privé.

&#91;image: Route 0.0.0.0/0 vers igw-tp7 et association du sous-réseau public\]

### Étape 5 — instance dans la VPC et test

1. **Lancer une instance** `tp7-web` → **Paramètres réseau** → **Modifier**.
2. VPC `vpc-tp7`, sous-réseau `subnet-public`, IP publique **Activer**.
3. Créer un SG `sg-tp7` : SSH + ICMP depuis Mon IP (les SG du VPC par défaut n'apparaissent pas : un SG appartient à un seul VPC).
4. Lancer, puis tester :

```bash
ssh -i cle-tp.pem ec2-user@<IP_PUBLIQUE>
hostname -I          # doit afficher 10.0.1.x
ping -c 3 8.8.8.8    # sortie Internet via igw-tp7
```

&#91;image: Paramètres réseau : vpc-tp7, subnet-public, IP publique activée\]

&#91;image: SSH réussi, IP 10.0.1.x et ping vers Internet\]

**Résultat attendu :** IP privée en `10.0.1.x`, SSH et ping Internet OK.

**Test bonus :** lancer une instance dans `subnet-prive`. Elle n'a pas d'IP publique et ne sort pas sur Internet : c'est la preuve que la route vers l'IGW fait la différence.

## Nettoyage et dépannage

**Nettoyage en fin de séance** (dans cet ordre, sinon AWS refuse les suppressions dépendantes) :

1. Aurora : supprimer l'instance puis le cluster `tp-aurora` (décocher le snapshot final).
2. EC2 : **Résilier** toutes les instances des TP.
3. S3 : **Vider** le bucket, puis le **Supprimer**.
4. Groupes de sécurité créés (`sg-tp2`, `sg-web`, `sg-A`, `sg-B`, `sg-tp7`).
5. VPC : détacher et supprimer `igw-tp7`, puis supprimer `vpc-tp7` (supprime aussi sous-réseaux et tables de routage).

**Dépannage rapide**

| Symptôme | Cause probable | Correction |
| --- | --- | --- |
| SSH ou ping : *timeout* | Pas de règle entrante, ou « Mon IP » a changé | Vérifier le SG, remettre « Mon IP » ou `0.0.0.0/0` |
| `Permission denied (publickey)` | Mauvais utilisateur ou mauvaise clé | Utilisateur `ec2-user`, clé `cle-tp.pem` du lancement |
| `UNPROTECTED PRIVATE KEY FILE` | Droits trop larges sur la clé | `chmod 400` ou commandes `icacls` du TP1 |
| Page web en timeout | Port 80 fermé ou URL en https | Règle HTTP 80, taper `http://` |
| nginx absent | User data mal collé (manque `#!/bin/bash`) | Lire `/var/log/cloud-init-output.log`, relancer une instance |
| S3 : 403 AccessDenied | Pas de policy, ou ARN sans `/*` | Policy `s3:GetObject` sur `arn:aws:s3:::bucket/*` |
| Aurora : connexion bloquée | SG de la base sans 3306 depuis l'instance | Règle MySQL/Aurora 3306, source = SG de l'instance |
| Instance VPC sans Internet | Pas d'IGW, de route `0.0.0.0/0` ou d'IP publique | Reprendre les étapes 2 à 4 du TP7 |

*Les captures de ce guide sont des schémas fidèles aux écrans de la console ; les libellés exacts peuvent varier légèrement selon la langue et la version de la console.*
