# EC1 – Ingénierie de la cryptographie : travaux pratiques

**Étudiant(e) :** *TRAORE Inoussa* — **INE :** *N00984520211* — **Année :** 2026–2027
**Enseignant :** Dr. Zakaria Sawadogo

Ce dépôt contient le rapport et le code des trois TP du module EC1
(*Cryptologie, clés et certificats*).

| TP | Thème | Dossier |
|----|-------|---------|
| 1 | Chiffrement authentifié (AES-GCM) et dérivation de clés (Argon2id, HKDF) | `tp1/` |
| 2 | Infrastructure de gestion de clés avec OpenSSL (racine, intermédiaire, révocation) | `tp2/` |
| 3 | Serveur TLS nginx avec authentification mutuelle, supervision, renouvellement | `tp3/` |

Le rapport complet est dans `rapport/` (`rapport_complet_redige.pdf` et sa source LaTeX).

> **Sécurité :** aucune clé privée n'est versionnée (voir `.gitignore`). Les clés sont
> générées localement par les scripts. Les mots de passe des démonstrations sont fictifs.

## Structure

```
.
├── README.md
├── .gitignore
├── rapport/   rapport_complet_redige.tex  rapport_complet_redige.pdf
├── tp1/       coffre.py  test_coffre.py  demo.py
├── tp2/       racine.cnf  inter.cnf  pki.sh
└── tp3/       nginx_tp.conf  surveiller.py  renouveler.sh
```

## Prérequis

- Linux (testé sous Ubuntu 24.04), Python 3.10 ou plus récent, OpenSSL 3.
- TP 3 : nginx, curl, nmap (paquets de la distribution).

```bash
python3 -m venv ~/ec1 && source ~/ec1/bin/activate
pip install cryptography argon2-cffi pytest
openssl version        # doit afficher OpenSSL 3.x
```

## TP 1 – Mini-coffre de fichiers

Module `coffre.py` : format versionné `CFR1` (en-tête de 44 octets : identifiant,
paramètres Argon2id, sel, nonce), AES-256-GCM, données associées = en-tête + nom du fichier.

```bash
cd tp1
python -m pytest -v      # 10 tests : aller-retour, mauvais mot de passe, intégrité, renommage...
python demo.py           # démonstration : tailles, en-tête, rechiffrement
```

Fonctions principales : `chiffrer`, `dechiffrer`, `rechiffrer`, `cle_maitresse`, `sous_cle`.
Paramètres par défaut : Argon2id t=3, m=128 Mio, p=4 (environ 0,5 s selon la machine).

## TP 2 – Infrastructure de gestion de clés

Construit une PKI à deux niveaux : racine P-384 (10 ans), intermédiaire P-256
(5 ans, `pathlen:0`, contraintes de nom `tp.local`), puis serveur, Alice et Bob (90 jours).

```bash
cd tp2
./pki.sh                 # crée racine/, inter/, serveur.*, alice.*, bob.*, crls.pem
```

Validations (étape 5) :

```bash
V="openssl verify -CAfile racine/racine.crt -untrusted inter/inter.crt"
$V -verify_hostname www.tp.local   serveur.crt   # OK
$V -verify_hostname autre.example  serveur.crt   # échec : hostname mismatch
$V -purpose sslclient alice.crt                  # OK
```

Révocation d'Alice (étape 6) :

```bash
openssl ca -config inter.cnf -revoke alice.crt -crl_reason keyCompromise
openssl ca -config inter.cnf  -gencrl -out inter/crl/inter.crl
openssl ca -config racine.cnf -gencrl -out racine/crl/racine.crl
cat inter/crl/inter.crl racine/crl/racine.crl > crls.pem
openssl verify -crl_check_all -CRLfile crls.pem -CAfile racine/racine.crt \
  -untrusted inter/inter.crt alice.crt                      # certificate revoked
```

Pour relancer `pki.sh`, supprimer d'abord les dossiers `racine/` et `inter/`.

## TP 3 – Canal sécurisé et gestion des certificats

Prérequis : avoir exécuté le TP 2 (même dossier de travail).

```bash
# Fichiers du serveur (chaîne = certificat + intermédiaire, jamais la racine)
cat serveur.crt inter/inter.crt > serveur_chaine.crt
cat bob.crt inter/inter.crt > bob_chaine.crt
sudo mkdir -p /etc/nginx/tls
sudo install -m 600 serveur.key /etc/nginx/tls/serveur.key
sudo install -m 644 serveur_chaine.crt racine/racine.crt crls.pem /etc/nginx/tls/
echo "127.0.0.1 www.tp.local" | sudo tee -a /etc/hosts

# Configuration nginx
sudo cp ../tp3/nginx_tp.conf /etc/nginx/conf.d/tp.conf
sudo nginx -t && sudo systemctl reload nginx

# Tests
curl --cacert racine/racine.crt --cert bob_chaine.crt --key bob.key https://www.tp.local/
```

Résultat attendu : `Bonjour CN=bob@tp.local`. Sans certificat client, ou avec celui d'Alice
(révoqué), nginx répond `HTTP 400`.

Le certificat client doit être envoyé **avec sa chaîne** (`bob_chaine.crt`), sinon nginx ne
peut pas construire le chemin de certification.

**Supervision** (codes de sortie : 0 = OK, 1 = ALERTE < 30 jours, 2 = CRITIQUE < 7 jours) :

```bash
python3 tp3/surveiller.py /etc/nginx/tls ; echo "code = $?"
```

**Renouvellement** (le dossier `PKI` doit contenir `inter.cnf`, `inter/` et `racine/`) :

```bash
sudo PKI=$PWD ./tp3/renouveler.sh              # ne fait rien si le certificat reste valable > 30 jours
sudo PKI=$PWD SEUIL=100 ./tp3/renouveler.sh    # force un renouvellement pour tester
```

Le script vérifie la chaîne avant tout déploiement, archive l'ancienne version dans
`/etc/nginx/tls/archive/` et revient en arrière en cas d'échec. La planification quotidienne
(minuteur systemd ou cron) est décrite dans le rapport.

> Tous les tests réseau visent uniquement le serveur de laboratoire local (`www.tp.local`).

## Compiler le rapport

```bash
cd rapport
pdflatex rapport_complet_redige.tex && pdflatex rapport_complet_redige.tex
```

Ou importer le fichier `.tex` dans Overleaf (compilateur pdfLaTeX, langue française).
