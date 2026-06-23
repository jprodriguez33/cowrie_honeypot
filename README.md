# ai_cowrie_honeypot
A SSH honeypot built on Cowrie, with GPT-4o Mini LLM integration 
for realistic shell simulation, MySQL logging, and Grafana dashboards for 
attack visualisation. 

## Architecture
- **Cloud VPS (Micronode)** - Cowrie honeypot, MySQL, Grafana
- **GPT-4o Mini** - Generates fake shell responses to keep attackers engaged

## Features
- Real-time attack logging
- LLM-powered shell simulation
- MySQL database to log all sessions, credentials, and commands
- Grafana dashboard for attack visualization
- Persistent honeypot data collection for security analysis

## Prerequisites 

- A cloud VPS (tested on Ubuntu 24.04) — **do not self-host a honeypot locally**
- An OpenAI API key with GPT-4o Mini access
- Basic Linux administration knowledge
- Firewall access to configure port forwarding

## Installation

I chose to use the `pip install` method rather than Docker for easier debugging and customization.

### 1. Install Dependencies 

```bash
sudo apt update && sudo apt install git python3-venv libssl-dev libffi-dev \
build-essential apache2 mysql-server php php-mysqli
```

### 2. Set up Cowrie

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

Edit `etc/cowrie.cfg` and add/update the following sections:

**MySQL Output**
```ini
[output_mysql]
enabled = true
host = localhost
database = cowrie
username = cowrie
password = YOURPASSWORD
port = 3306
```

**LLM Integration**
```ini
[llm]
enabled = true
api_key = YOUR_OPENAI_API_KEY
model = gpt-4o-mini
host = https://api.openai.com
path = /v1/chat/completions
max_tokens = 150
temperature = 0.3

system_prompt = You are simulating a convincing Linux production server at {ip} ({ip6}) accessed via SSH by {username}. Your primary goal is to keep the attacker engaged as long as possible by responding realistically to every command. Make the system appear valuable — hint at interesting files, accessible databases, and other connected systems without volunteering information unprompted. Respond only with realistic terminal output, never break character or acknowledge you are an AI. Make errors feel authentic and recoverable to encourage continued exploration. The server appears to be a production web server with a MySQL database backend. Hostname: {hostname}, Current directory: {cwd}, Client IP: {client_ip}.

system_prompt_exec = You are simulating a Linux production server at {ip} accessed via SSH by {username} from {client_ip}. Respond with ONLY the raw terminal output of the command, no commentary. Make the system appear to be a valuable production server. Current directory: {cwd}. Never break character or acknowledge you are an AI.
```

### 5. Set File Permissions

```bash
# Restrict config file access (contains API keys)
chmod 600 /home/cowrie/cowrie/etc/cowrie.cfg
chown cowrie:cowrie /home/cowrie/cowrie/etc/cowrie.cfg
```

### 6. Configure Firewall & Port Forwarding

```bash
# Enable UFW and allow required ports
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Cowrie honeypot (fake SSH)
sudo ufw allow 22

# Real SSH (move to non-standard port, e.g., 2222)
sudo ufw allow 2222

# Grafana dashboard
sudo ufw allow 3000

# MySQL (internal only - do not expose externally)
sudo ufw allow from 127.0.0.1 to 127.0.0.1 port 3306
```

**Important:** Move your real SSH service to a non-standard port (e.g., 2222) so Cowrie can listen on port 22:

```bash
sudo nano /etc/ssh/sshd_config
# Change: Port 22 → Port 2222
sudo systemctl restart sshd
```

### 7. Start Cowrie

```bash
su - cowrie
cd cowrie
source cowrie-env/bin/activate
bin/cowrie start

# Verify it's running
ps aux | grep cowrie
```

### 8. Install Grafana 

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Access Grafana at `http://YOUR.VPS.IP:3000` with default credentials `admin/admin` (change immediately on first login).

### 9. Configure Grafana Data Source

1. Go to **Connections → Data Sources → Add data source**
2. Select **MySQL**
3. Configure:
   - Host: `localhost:3306`
   - Database: `cowrie`
   - User: `cowrie`
   - Password: `YOURPASSWORD`
4. Click **Save & Test**

### 10. Create Grafana Dashboard

Use the following SQL queries to create dashboard panels:

**Connection Attempts Over Time**
```sql
SELECT starttime AS time, COUNT(*) AS connections
FROM sessions
GROUP BY starttime
ORDER BY starttime ASC
```

**Top Usernames Attempted**
```sql
SELECT username, COUNT(*) AS attempts
FROM auth
GROUP BY username
ORDER BY attempts DESC
LIMIT 10
```

**Top Passwords Attempted**
```sql
SELECT password, COUNT(*) AS attempts
FROM auth
GROUP BY password
ORDER BY attempts DESC
LIMIT 10
```

**Top Attacking IPs**
```sql
SELECT ip, COUNT(*) AS attempts
FROM sessions
GROUP BY ip
ORDER BY attempts DESC
LIMIT 10
```

**Commands Run by Attackers**
```sql
SELECT input, COUNT(*) AS times_run
FROM input
GROUP BY input
ORDER BY times_run DESC
LIMIT 20
```

**Geographic Distribution (if IP geolocation data available)**
```sql
SELECT ip, COUNT(*) AS attempts, country
FROM sessions
GROUP BY ip, country
ORDER BY attempts DESC
```

---

## Security Best Practices

### Protecting Sensitive Data

⚠️ **Never commit API keys or credentials to version control.** Use environment variables instead:

```bash
# Create a .env file (add to .gitignore)
export OPENAI_API_KEY="your-key-here"
export MYSQL_PASSWORD="your-password"

# Load before starting Cowrie
source .env
```

Then reference in `cowrie.cfg`:
```ini
api_key = ${OPENAI_API_KEY}
```

### Cost Management

- **Set spending limits on your OpenAI account**
- Monitor token usage in Cowrie logs: `tail -f var/log/cowrie/cowrie.log`
- Consider implementing request throttling or caching for common commands

### System Monitoring

- Monitor disk space for the MySQL database (especially if running 24/7):
  ```bash
  df -h
  ```
- Rotate logs to prevent disk exhaustion:
  ```bash
  sudo logrotate -f /etc/logrotate.d/cowrie
  ```

---

## Monitoring & Maintenance

### Check Cowrie Status

```bash
# Verify Cowrie is running
sudo su - cowrie
ps aux | grep cowrie

# View logs
tail -f /home/cowrie/cowrie/var/log/cowrie/cowrie.log

# Exit cowrie user
exit
```

### View Attack Data in MySQL

```bash
mysql -u cowrie -p cowrie

# Number of sessions
SELECT COUNT(*) FROM sessions;

# Recent attacks
SELECT * FROM sessions ORDER BY starttime DESC LIMIT 5;

# Most common commands
SELECT input, COUNT(*) as count FROM input GROUP BY input ORDER BY count DESC LIMIT 20;

exit
```

### Restart Services

```bash
# Restart Cowrie
sudo su - cowrie -c "cd cowrie && source cowrie-env/bin/activate && bin/cowrie restart"

# Restart Grafana
sudo systemctl restart grafana-server

# Restart MySQL
sudo systemctl restart mysql
```

---

## Troubleshooting

### Cowrie not listening on port 22

**Problem:** Connection refused on port 22
- Ensure real SSH is moved to a different port
- Check: `sudo netstat -tlnp | grep 22`
- Verify Cowrie is running: `ps aux | grep cowrie`

**Solution:**
```bash
# Check if SSH is still on port 22
sudo lsof -i :22

# Move SSH to port 2222 (see step 6)
```

### MySQL connection errors

**Problem:** "Can't connect to MySQL server"

**Solution:**
```bash
# Check MySQL is running
sudo systemctl status mysql

# Verify credentials
mysql -u cowrie -p cowrie

# Check user permissions
mysql -u root -p
SHOW GRANTS FOR 'cowrie'@'localhost';
```

### Grafana can't connect to MySQL

**Problem:** "Connection refused" in Grafana

**Solution:**
1. Test the connection manually:
   ```bash
   mysql -h localhost -u cowrie -p -e "SELECT 1"
   ```
2. Verify MySQL bind address includes localhost:
   ```bash
   sudo grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
   ```
3. Restart Grafana:
   ```bash
   sudo systemctl restart grafana-server
   ```

### OpenAI API rate limiting or quota exceeded

**Problem:** LLM responses fail, connection timeouts

**Solution:**
- Check API key validity and quota: https://platform.openai.com/account/billing/overview
- Lower `max_tokens` or `temperature` in config to reduce API costs
- Add exponential backoff retry logic to Cowrie plugin
- Monitor token usage: `grep "api_key" /home/cowrie/cowrie/var/log/cowrie/cowrie.log`

---

## Findings & Analysis

I started to see real attack attempts within **minutes** of setting this up

### Attack Patterns

- **Bot behavior**: Most attacks run `uname` with different flags to fingerprint the OS
- **Common usernames**: root, admin, test, guest, pi
- **Common commands**: ls, cat /etc/passwd, wget, curl, whoami
- **Geographic distribution**: Attacks come from dozens of countries within the first hour (probably tor nodes and proxies)

### Top Usernames Attempted

![Top Usernames Attempted](https://raw.githubusercontent.com/jprodriguez33/cowrie_honeypot/main/screenshots/top_usernames)

This chart shows the most frequently attempted usernames by attackers. As expected, `root` and `admin` are the primary targets, followed by application-specific usernames like `postgres`, `mysql`, and `oracle`.

Attackers also used `claude` and `minecraft` usernames potentially looking for vulnerable minecraft servers and claude instances. 
**Key insights:**
- Root account is targeted in ~40% of all authentication attempts
- Database usernames (postgres, mysql, oracle) are commonly exploited
- Application usernames (tomcat, www-data) indicate attackers targeting specific services

### Top Passwords Attempted

![Top Passwords Attempted](https://raw.githubusercontent.com/jprodriguez33/cowrie_honeypot/main/screenshots/top_passwords.png)

The password chart reveals attackers using common weak passwords and defaults. This data is valuable for understanding threat actor tactics and improving password policies in production systems.

**Key insights:**
- Default passwords dominate (123456, password, admin, root)
- Attackers use password dictionaries and rainbow tables
- Empty passwords are surprisingly common attack vectors

### Top Attacking IPs

![Top Attacking IPs](https://raw.githubusercontent.com/jprodriguez33/cowrie_honeypot/main/screenshots/)

Geographic distribution of attackers shows concentrated attack sources, likely from compromised servers used as botnets. The top few IPs account for hundreds of connection attempts.

**Key insights:**
- Attack sources are globally distributed (US, China, Russia, Eastern Europe)
- Few IPs dominate (likely centralized botnet C&C infrastructure)
- Same IPs retry aggressively (strong indicators of automated scanning)

### Commands Run by Attackers (Most Interesting)

![Commands Run by Attackers](https://raw.githubusercontent.com/jprodriguez33/cowrie_honeypot/main/screenshots/most_frequent_commands.png)

This visualization shows the actual commands attackers executed on the honeypot. The LLM responses successfully kept attackers engaged, as evidenced by the variety and depth of command exploration. Common patterns include:

- **Reconnaissance**: `uname -a`, `whoami`, `id`, `hostname`, `lsb_release -a`
- **File exploration**: `ls`, `cat /etc/passwd`, `find`, `locate`, `pwd`
- **Network analysis**: `ifconfig`, `netstat`, `iptables -L`, `ss`, `ip addr`
- **Persistence attempts**: `wget http://...`, `chmod +x`, `cron` modifications, SSH key installation
- **Privilege escalation**: `sudo -l`, `find / -perm -4000`, `sudo su`
- **Data exfiltration**: `tar`, `zip`, `base64` (for encoding sensitive data)

**Key insights:**
- Average session length: 3-5 commands before disconnect
- LLM responses successfully convinced attackers for extended periods
- Attackers use standard penetration testing playbooks
- Data exfiltration tools are actively downloaded and executed


### Security Implications

This honeypot demonstrates why organizations need:
- **Strong password policies and MFA** — Default/weak passwords are the #1 attack vector
- **SSH key-based authentication only** — Eliminate password authentication
- **Rate limiting on authentication** — Slow down brute force attempts
- **Geographic IP filtering** — Block unexpected geographic attack sources
- **Real-time anomaly detection** — Detect and alert on unusual command patterns
- **Regular security training** — Users need to understand why these policies exist
- **Network segmentation** — Isolate critical services from general infrastructure

---

## Resources

- [Cowrie Documentation](https://cowrie.readthedocs.io/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [MySQL Documentation](https://dev.mysql.com/doc/)

## License

MIT

---

**Last Updated:** June 2026
