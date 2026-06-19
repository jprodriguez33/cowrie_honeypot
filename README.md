# ai_cowrie_honeypot
A SSH honeypot built on Cowrie, with GPT-4o Mini LLM integration 
for realistic shell simulation, MySQL logging, and Grafana dashboards for 
attack visualisation. 

## Architecture
- **Cloud VPS (Micronode)** - Cowrie honeypot, mySQL, Grafana
- **GPT-4o Mini** - Generates fake shell response to keep attackers engaged

## Features
- real-time logging
- llm-powered shell simulation
- MySQL database to log all sessions, credentials, and commands
- grafana dashboard for attack visualization

## Prerequisites 

- a cloud VPS (tested on Ubuntu 24.04) please don't try to self-host a honeypot locally
- an OpenAI API key with GPT-4o Mini access
- linux basics

## Installation

I chose to use the `pip install` method rather than using a docker container. 

### 1. Install Dependencies 

```bash

sudo apt update && sudo apt install git python3-venv libssl-dev libffi-dev \\
build-essential apache2 mysql-server php php-mysqli

```
### 2. Set up cowrie

```bash
git clone https://github.com/cowrie/cowrie /home/cowrie/cowrie
cd /home/cowrie/cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install -r requirements.txt
pip install mysql-connector-python
cp etc/cowrie.cfg.dist etc/cowrie.cfg

```
### 3. Set up MySQL 

```bash
mysql -u root -p
```

```sql
CREATE DATABASE cowrie;
CREATE USER 'cowrie'@'localhost' IDENTIFIED BY 'YOURPASSWORD';
GRANT ALL ON cowrie.* TO 'cowrie'@'localhost';
FLUSH PRIVILEGES;
exit
```
Load the Cowrie schema:

```bash
mysql -u cowrie -p cowrie < /home/cowrie/cowrie/docs/sql/mysql.sql
```

### 4. Configure Cowrie

edit `etc/cowrie.cfg` and add/update the following sections:

MySQL Output 
```ini
[output\_mysql]
enabled = true
host = localhost
database = cowrie
username = cowrie
password = YOURPASSWORD
port = 3306
```
LLM Integration

```ini
[llm]
enabled = true
api\_key = YOUR\_OPENAI\_API\_KEY
model = gpt-4o-mini
host = https://api.openai.com
path = /v1/chat/completions
max\_tokens = 150
temperature = 0.3

system\_prompt = You are simulating a convincing Linux production server at {ip} ({ip6}) 
accessed via SSH by {username}. Your primary goal is to keep the attacker engaged as long 
as possible by responding realistically to every command. Make the system appear valuable 
— hint at interesting files, accessible databases, and other connected systems without 
volunteering information unprompted. Respond only with realistic terminal output, never 
break character or acknowledge you are an AI. Make errors feel authentic and recoverable 
to encourage continued exploration. The server appears to be a production web server with 
a MySQL database backend. Hostname: {hostname}, Current directory: {cwd}, Client IP: {client\_ip}.

system\_prompt\_exec = You are simulating a Linux production server at {ip} accessed via SSH 
by {username} from {client\_ip}. Respond with ONLY the raw terminal output of the command, 
no commentary. Make the system appear to be a valuable production server. Current directory: 
{cwd}. Never break character or acknowledge you are an AI.
```
### 5. Start Cowrie

```bash
su - cowrie
cd cowrie
source cowrie-env/bin/activate
bin/cowrie start
```

### 6. Install Grafana 

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo ufw allow 3000
```
access Grafana at `http://YOUR.VPS.IP:3000` with default creds `admin/admin`

### 7. Configure Grafana
1. go to connections -> Data Sources -> add data source
2. select MySQL
3. Set host to `localhost:3306`, database to `cowrie`, `cowrie `
4. click **Save & Test**

### 8. Grafana Dashboard Queries

Connection Attempts Over Time

```sql
SELECT starttime AS time, COUNT(\*) AS connections
FROM sessions
GROUP BY starttime
ORDER BY starttime ASC
```

Top Usernames Attempted 

```sql
SELECT username, COUNT(\*) AS attempts
FROM auth
GROUP BY username
ORDER BY attempts DESC
LIMIT 10
```

Top Passwords Attempted

```sql
SELECT password, COUNT(\*) AS attempts
FROM auth
GROUP BY password
ORDER BY attempts DESC
LIMIT 10
```

Top Attacking IPs

```sql

SELECT ip, COUNT(\*) AS attempts
FROM sessions
GROUP BY ip
ORDER BY attempts DESC
LIMIT 10
```

Commands Run by Attackers (the most interesting in my opinion)

```sql

SELECT input, COUNT(\*) AS times\_run
FROM input
GROUP BY input
ORDER BY times\_run DESC
LIMIT 20
```

# Notes 
- Move your real SSH to a non-standard port so cowrie listens on port 22
- Make sure you set spending limits on your OpenAI account
