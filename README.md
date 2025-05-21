# =========================================================
# 📁 Projet : Détection d'une attaque DDoS avec IDS, SIEM et SOAR
# 🎯 Objectif : Simuler une attaque DDoS détectée par un IDS, remontée à Splunk (SIEM), puis traitée via Shuffle (SOAR)
# =========================================================

# === [0] PRÉREQUIS ===
## VM Ubuntu 20.04 ou 22.04 (4 vCPU, 6 Go RAM, 30 Go disque)
## Accès Internet, ports ouverts : 8000 (Splunk), 3001 (Shuffle), 8088 (Splunk HEC)

# === [1] CRÉATION DU DÉPÔT GITHUB ===
## 1. Aller sur https://github.com et créer un dépôt vide : DDoS-Detection-Lab
## 2. Coche "Add a README.md"
## 3. Copier l'URL du dépôt (ex: https://github.com/tonpseudo/DDoS-Detection-Lab.git)
## 4. Cloner le dépôt localement :
##    git clone https://github.com/tonpseudo/DDoS-Detection-Lab.git
##    cd DDoS-Detection-Lab

# === [2] INSTALLATION DE SPLUNK (SIEM) ===

## mkdir ~/siem-soar-lab && cd ~/siem-soar-lab
## wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/8.2.6/linux/splunk-8.2.6-a6fe1ee8894b-linux-2.6-amd64.deb"
## sudo dpkg -i splunk.deb
## sudo /opt/splunk/bin/splunk start --accept-license --answer-yes
## Créer login/mot de passe admin
## Accéder à Splunk : http://localhost:8000

![image](https://github.com/user-attachments/assets/f292562e-a8a7-4227-a7ea-80c4abab1f74)

# === [3] CONFIGURATION DE SPLUNK POUR HEC (HTTP Event Collector) ===
## Settings > Data Inputs > HTTP Event Collector > New Token : "ids_ddos_token" > index = ddos_logs
![image](https://github.com/user-attachments/assets/953e2f00-3cb7-478d-aca7-354c07aba7e0)

## Settings > Data Inputs > Global Settings > Enable HEC (SSL désactivé pour le lab)
![image](https://github.com/user-attachments/assets/14657fbf-4210-4137-8fc8-087041e2ab01)

## Conserver l'URL d'envoi : http://localhost:8088

# === [4] INSTALLATION DE SURICATA (IDS) ===
## sudo apt install suricata -y
## sudo systemctl enable suricata && sudo systemctl start suricata

# Configuration pour journaliser dans un fichier lisible par Splunk
Par défaut, Suricata écrit ses logs JSON dans ce fichier :
/var/log/suricata/eve.json
Vérifie qu'il existe avec :
ls -l /var/log/suricata/eve.json
![image](https://github.com/user-attachments/assets/81e934ec-2817-47b3-9665-9f80d99db59c)

Édite le fichier de configuration Suricata :
sudo nano /etc/suricata/suricata.yaml
Cherche la section suivante dans le fichier (outputs > eve-log) :
![image](https://github.com/user-attachments/assets/c2c6c326-ed3a-4678-9618-2fbc9ed9f4ef)
enabled: yes
filetype: regular
filename: /var/log/suricata/eve.json
![image](https://github.com/user-attachments/assets/63409b37-077b-4d80-9dba-d6f701b2f7aa)


# Le fichier de log est souvent : /var/log/suricata/eve.json

# === [5] SIMULATION D'UNE ATTAQUE DDoS ===
sudo apt install hping3 -y
# Depuis une autre machine ou terminal, lancer :
# sudo hping3 -S --flood -p 80 <IP_VM_avec_Suricata>

# === [6] AJOUT DES LOGS IDS DANS SPLUNK ===
sudo /opt/splunk/bin/splunk add monitor /var/log/suricata/eve.json -index ddos_logs -sourcetype suricata:json
sudo /opt/splunk/bin/splunk restart

# === [7] INSTALLATION DE SHUFFLE (SOAR) ===
sudo apt update && sudo apt install docker.io docker-compose git -y
sudo systemctl enable docker && sudo systemctl start docker
cd ~/siem-soar-lab
git clone https://github.com/frikky/Shuffle.git
cd Shuffle
sudo docker-compose up -d
![image](https://github.com/user-attachments/assets/1080d337-617e-4a1d-a2d6-8c0d0ae66438)

# Accéder à http://localhost:3001 > créer compte admin
![image](https://github.com/user-attachments/assets/8aa74f57-9c76-45df-b68b-23a579509485)

# === [8] AJOUTER SPLUNK DANS SHUFFLE ===
# Shuffle > Apps > New App > Rechercher "Splunk"
# Base URL : http://localhost:8088
# Token : copier le token HEC créé
![image](https://github.com/user-attachments/assets/ca6d8010-d0d3-42ad-add9-36de94f4385f)

# === [9] CRÉATION DU PLAYBOOK SOAR (Détection DDoS) ===

# Le playbook agit sur détection de l'événement DDoS et déclenche une alerte Splunk
# YAML à importer dans Shuffle

name: Alerte_DDOS_IDS
apps:
  - name: splunk
steps:
  - name: Déclencheur_DDoS
    app: builtin
    type: eventlistener
    trigger:
      query: 'index="ddos_logs" sourcetype="suricata:json" "flow" AND "attack"'
      interval: 30
    next: Envoyer_Alerte
  - name: Envoyer_Alerte
    app: splunk
    action: send_event
    parameters:
      index: "ddos_logs"
      sourcetype: "soar_alert"
      event: '"Alerte DDoS détectée par IDS (Suricata) envoyée à {{now}}"'

# === [10] TEST ===
# Lancer hping3 pour simuler une attaque
# Attendre que Suricata log l'activité suspecte
# Vérifier dans Splunk (Recherche) :
# index="ddos_logs" sourcetype="suricata:json"
# Vérifier les alertes générées :
# index="ddos_logs" sourcetype="soar_alert"

