
# DDoS attack detection with IDS, SIEM and SOAR

This project aims to establish a robust cybersecurity infrastructure capable of effectively detecting and responding to Distributed Denial of Service (DDoS) attacks. By integrating an Intrusion Detection System (IDS) for real-time malicious activity identification, a Security Information and Event Management (SIEM) system for centralized security log analysis, and a Security Orchestration, Automation, and Response (SOAR) platform to automate incident response, we will build a multi-layered defense system. This combined approach will not only enable early detection of DDoS threats but also facilitate automated and coordinated reactions, thereby minimizing impact on service availability.

![image](https://github.com/user-attachments/assets/824af555-9299-438f-b5d0-8e4673da6bc7)

## Objective 

This project focuses on establishing a comprehensive and automated defense mechanism against Distributed Denial of Service (DDoS) attacks. I will simulate a DDoS attack within a controlled environment, ensuring it's successfully detected by an Intrusion Detection System (IDS). The alerts generated by the IDS will then be forwarded to Splunk (SIEM) for centralized logging, correlation, and analysis. Finally, Shuffle (SOAR) will be utilized to orchestrate an automated response to the detected incident, demonstrating a proactive and efficient mitigation strategy.

## Skills learned

<div>
  
- Ability to design and deploy a multi-layered cybersecurity architecture for DDoS detection and response.
- Hands-on experience with configuring and fine-tuning an Intrusion Detection System (IDS) to identify DDoS attack signatures.
- Proficiency in integrating security tools, specifically forwarding IDS alerts to a SIEM platform (Splunk) for centralized visibility.
- Skills in creating SIEM rules and dashboards for effective monitoring and correlation of security events.
- Practical experience with Security Orchestration, Automation, and Response (SOAR) platforms (Shuffle) to build automated playbooks for incident response.
- Improved understanding of DDoS attack methodologies and their impact on network availability.
  
</div>

## Tools

<div>

<img src="https://img.shields.io/badge/Shuffle-SOAR_Automation-F57C00?style=for-the-badge&logo=shuffle&logoColor=white" />
<img src="https://img.shields.io/badge/Suricata-Intrusion_Detection-CF2C1D?style=for-the-badge&logo=suricata&logoColor=white" />
<img src="https://img.shields.io/badge/Splunk-SIEM_Platform-14833b?style=for-the-badge&logo=splunk&logoColor=white" />

</div>

## PREREQUISITES 

- Ubuntu 20.04 or 22.04 VM (4 vCPU, 6 GB RAM, 30 GB disk)
- Internet access, open ports: 8000 (Splunk), 3001 (Shuffle), 8088 (Splunk HEC)

## CREATE GITHUB REPOSITORY
- 1 - Go to https://github.com and create an empty repository: DDoS-Detection-Lab
- 2 - Check "Add a README.md".
- 3 - Copy the repository URL (e.g. https://github.com/tonpseudo/DDoS-Detection-Lab.git)
- 4 - Clone the repository locally :
  - git clone https://github.com/tonpseudo/DDoS-Detection-Lab.git
  - cd DDoS-Detection-Lab
    
## SPLUNK (SIEM) INSTALLATION
- mkdir ~/siem-soar-lab && cd ~/siem-soar-lab
- wget -O splunk.deb “https://download.splunk.com/products/splunk/releases/8.2.6/linux/splunk-8.2.6-a6fe1ee8894b-linux-2.6-amd64.deb”
- sudo dpkg -i splunk.deb
- sudo /opt/splunk/bin/splunk start --accept-license --answer-yes
- Create admin login/password
- Access Splunk: http://localhost:8000

<p align="center">
  <img src="https://github.com/user-attachments/assets/f292562e-a8a7-4227-a7ea-80c4abab1f74" alt="Splunk Interface" />
</p>
<p align="center"><b>Figure 1 : Splunk Interface</b></p>


## SPLUNK CONFIGURATION FOR HEC (HTTP Event Collector)
- Settings > Data Inputs > HTTP Event Collector > New Token : « ids_ddos_token » > index = ddos_logs

<p align="center">
  <img src="https://github.com/user-attachments/assets/953e2f00-3cb7-478d-aca7-354c07aba7e0" alt="Setting a New Token" />
</p>
<p align="center"><b>Figure 2 : Setting a New Token</b></p>

- Settings > Data Inputs > Global Settings > Enable HEC (SSL disable)

<p align="center">
  <img src="https://github.com/user-attachments/assets/14657fbf-4210-4137-8fc8-087041e2ab01" alt="Enabling HEC" />
</p>
<p align="center"><b>Figure 3 : Enabling HEC</b></p>

- Keep sending URL: http://localhost:8088
  
## INSTALLING SURICATA (IDS)
- sudo apt install suricata -y
- sudo systemctl enable suricata && sudo systemctl start suricata

# Configuration to log to a Splunk file
By default, Suricata writes its JSON logs to this file: /var/log/suricata/eve.json
- Check that it exists with :
 -ls -l /var/log/suricata/eve.json

<p align="center">
  <img src="https://github.com/user-attachments/assets/81e934ec-2817-47b3-9665-9f80d99db59c" alt="Verify if eve.json exist" />
</p>
<p align="center"><b>Figure 4 : Verify if eve.json exist</b></p>


- Edit the Suricata configuration file:
 - sudo nano /etc/suricata/suricata.yaml
 - Look for the following section in the file (outputs > eve-log):

<p align="center">
  <img src="https://github.com/user-attachments/assets/63409b37-077b-4d80-9dba-d6f701b2f7aa" alt="Surricata YAML file" />
</p>
<p align="center"><b>Figure 5 : Surricata YAML file</b></p>

   - enabled: yes
   - filetype: regular
   - filename: /var/log/suricata/eve.json

## SIMULATION OF A DDoS ATTACK
To check that everything is properly configured, we're going to simulate a DOS attack on the VM using another machine.
- From another machine or terminal, run :
  - sudo apt install hping3 -y
  - sudo hping3 -S --flood -p 80 <IP_VM_with_Suricata>
  - Go to Slunk interface > Applications > Search and Reporting > Search for "index=ddos_logs sourcetype=surricata:json"

<p align="center">
  <img src="https://github.com/user-attachments/assets/b80e5fb9-45c1-48cb-a535-87acb17ea43c" alt="Surricata logs in Splunk" />
</p>
<p align="center"><b>Figure 6 : Surricata logs in Splunk</b></p>

# ADD LOG IDS TO SPLUNK
- sudo /opt/splunk/bin/splunk add monitor /var/log/suricata/eve.json -index ddos_logs -sourcetype suricata:json
- sudo /opt/splunk/bin/splunk restart

# SHUFFLE (SOAR) INSTALLATION
- sudo apt update && sudo apt install docker.io docker-compose git -y
- sudo systemctl enable docker && sudo systemctl start docker
- cd ~/siem-soar-lab
- git clone https://github.com/frikky/Shuffle.git
- cd Shuffle
- sudo docker-compose up -d

<p align="center">
  <img src="https://github.com/user-attachments/assets/1080d337-617e-4a1d-a2d6-8c0d0ae66438" alt="Shuffle interface" />
</p>
<p align="center"><b>Figure 7 : Shuffle interface</b></p>

- Go to http://localhost:3001 > create admin account

# ADD SPLUNK IN SHUFFLE
- Shuffle > Apps > New App > Search "Splunk"
- Base URL : http://localhost:8088
- Token : copy the HEC token created

<p align="center">
  <img src="https://github.com/user-attachments/assets/ca6d8010-d0d3-42ad-add9-36de94f4385f" alt="Setting Splunk on Shuffle" />
</p>
<p align="center"><b>Figure 8 : Setting Splunk on Shuffle</b></p>


# Create a Webhook on Shuffle
- Go to Shuffle > Workflow > Webhook >
- copy the Webhook

# Create a Alert to detect an DDOS attack on Slunk
- Go to Slunk > Applications > Search and Reporting > search for " index=ddos_logs sourcetype=suricata:json event_type=flow
| stats count by src_ip
| where count > 100" (designed to identify source IP addresses (src_ip) that generate an unusually high volume of traffic, a common indicator of a DDoS attack)
- Once the search done > Register At > Alert
- More actions > Webhook > Configure as below and add the Shuffle's Webhook link

<p align="center">
  <img src="https://github.com/user-attachments/assets/7d17f8b8-f79f-416f-a86b-f6f4aadcfd20" alt="Alert's configuration for DDOS with Webhook" />
</p>
<p align="center"><b>Figure 9 : Alert's configuration for DDOS with Webhook</b></p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/216fd7a2-4ba8-4c1d-8785-be9fc201ef54" alt="Alert available" />
</p>
<p align="center"><b>Figure 10 : Alert available</b></p>

# TEST
- Run hping3 to simulate an attack
- Wait for Suricata to log suspicious activity
- Check in Splunk (Search):
- index="ddos_logs" sourcetype="suricata:json"
- Check alerts generated:
- index="ddos_logs" sourcetype="soar_alert"
  
<p align="center">
  <img src="https://github.com/user-attachments/assets/1536c509-dac0-4045-93dc-4bf19638bde9" alt="Received alerts on Shuffle from Splunk" />
</p>
<p align="center"><b>Figure 11 : Received triggered alerts on Shuffle from Splunk </b></p>
