# Cyber project

# 🔐 Raspberry Pi Network Intrusion Detection System (NIDS) with Suricata + ELK Stack

This project turns a Raspberry Pi into a functioning IDS (Intrusion Detection System) that detects and forwards logs to a central dashboard on your laptop. You’ll use Suricata for packet inspection, Filebeat for log shipping, and Elasticsearch + Kibana (ELK Stack) to visualize everything in real time.

---

## 🛠️ Tools Used

- Raspberry Pi (monitoring device)
- Suricata (IDS)
- Filebeat (log shipper)
- Docker + Docker Compose
- Elasticsearch & Kibana (hosted on laptop)

---

## 📦 Step 1: Set Up Suricata on the Raspberry Pi

1. SSH into your Pi and create a directory for Suricata.
2. Inside the newly created Suricata directory, create 3 more directories:
    
    ```bash
    mkdir -p ~/suricata/{etc,lib,logs}
    
    ```
    
3. Then create a `docker-compose.yml` file (or deploy with Portainer) using this configuration:
    
    ```yaml
    version: "3.8"
    
    services:
      suricata:
        image: jasonish/suricata:latest
        container_name: suricata
        network_mode: host # Gives the container direct access to the Pi network interface.
        privileged: true  # Enables correct logging on Raspberry Pi OS
        cap_add: # Grants suricata necessary privileges.
          - NET_ADMIN
          - NET_RAW
          - SYS_NICE
        volumes: # Binds local directories to the container directories
          - /home/pi/suricata/logs:/var/log/suricata
          - /home/pi/suricata/etc:/etc/suricata
          - /home/pi/suricata/lib:/var/lib/suricata
        command: -i eth0 # Enabes listening mode on eth0
        restart: unless-stopped
    ```
    
4. Deploy the container and check if Suricata is writing logs:
    
    ```bash
    cd ~/suricata/logs
    cat eve.json | head -n 5
    ```
    
    If the file has output, Suricata is capturing traffic.
    

---

## 🐳 Step 2: Install Docker Desktop on Your Laptop

1. Visit [https://www.docker.com/pricing](https://www.docker.com/pricing) and download Docker Desktop.
2. Choose the **Personal plan** and install it.
3. Make sure you select **WSL 2** integration during installation setup.

---

## 📊 Step 3: Deploy the ELK Stack Using Docker

1. On your laptop, create a folder called `elk-stack`.
2. Inside it, create a file named `docker-compose.yml` and paste:
    
    ```yaml
    version: '3.7'
    
    services:
      elasticsearch: # Deploys the elasticsearch container
        image: docker.elastic.co/elasticsearch/elasticsearch:8.12.2
        container_name: elasticsearch
        environment:
          - discovery.type=single-node
          - xpack.security.enabled=false
        ports:
          - "9200:9200"
        volumes:
          - esdata:/usr/share/elasticsearch/data
        networks:
          - elk
    
      kibana: # Deploys the kibana container
        image: docker.elastic.co/kibana/kibana:8.12.2
        container_name: kibana
        environment:
          - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
        ports:
          - "5601:5601"
        networks:
          - elk
    
    volumes:
      esdata:
    
    networks:
      elk:
        driver: bridge
    
    ```
    
3. Open PowerShell or Windows CMD, and `cd` into the folder, and run:
    
    ```bash
    docker-compose up -d
    ```
    
4. After a few minutes, the Docker Desktop should show two containers:
    - `elasticsearch` (port 9200)
    - `kibana` (port 5601)
        
        ![image.png](image.png)
        
5. Confirm everything’s working:
    - Go to `http://localhost:9200` for Elasticsearch
        
        ![image.png](image%201.png)
        
    - Go to `http://localhost:5601` for Kibana
        
        ![image.png](image%202.png)
        
        > If prompted in Kibana, choose "Explore on my own"
        > 

---

## 📤 Step 4: Install and Configure Filebeat on the Pi

1. On the Pi, install Filebeat:
    
    ```bash
    curl -L -O <https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.2-arm64.deb>
    sudo dpkg -i filebeat-8.12.2-arm64.deb
    ```
    
2. Open the config file:
    
    ```bash
    sudo nano /etc/filebeat/filebeat.yml
    ```
    
3. Find the input section and **comment out** the default filestream config by adding `#` at the beginning of each line.
    
    ![image.png](image%203.png)
    
4. At the bottom of that section, add this Suricata input:
    
    ```yaml
    filebeat.inputs:
      - type: log
        enabled: true
        paths:
          - /home/pi/suricata/logs/eve.json # Your path
        json.keys_under_root: true
        json.add_error_key: true
    ```
    
    ![image.png](image%204.png)
    
5. Scroll down to the `output.elasticsearch` section and replace the `localhost`with your **laptop's local IP**:
    
    ```yaml
    output.elasticsearch:
      hosts: ["<http://192.168.1.199:9200>"] # Your IP.
      protocol: "http"
    ```
    
    ![image.png](image%205.png)
    
6. Save and exit the file (`CTRL + S` then `CTRL + X`).
7. Restart Filebeat:
    
    ```bash
    sudo systemctl restart filebeat
    ```
    
8. Watch logs live to confirm data is flowing:
    
    ```bash
    sudo journalctl -u filebeat -f
    ```
    
    ![image.png](image%206.png)
    

---

## 📈 Step 5: View Suricata Logs in Kibana

1. Open your browser and go to: [http://localhost:5601](http://localhost:5601/)
2. Click the ☰ menu → **Stack Management**
3. Go to **Index Management → Data Streams** and make sure you see something like `filebeat-*`
    
    ![image.png](image%207.png)
    
4. Now go to **Discover** and click **“Create data view”**
    
    ![image.png](image%208.png)
    
    - Set the pattern to `filebeat-*`
    - Choose `@timestamp` as the time field
        
        ![image.png](image%209.png)
        
5. You should now see real-time logs from Suricata including:
    - DNS requests
    - Alerts
    - Protocol details
    - Source/destination IPs
    
    ![image.png](image%2010.png)
    

---

## ⚠️ Important Note

Suricata running on the Pi will **only see traffic that reaches the Pi's network interface**. So unless:

- You enable **port mirroring (SPAN)** on your switch/router, or
- You route traffic through the Pi (inline bridge mode),

It won't see traffic for other devices on the network. These methods are outside the scope of this tutorial.

---

## 🧪 What’s Next

Now that the stack is working:

- We will use`nmap` or other tools to generate alerts from Kali VM.
- Use Kibana’s query bar to search logs with filters.
- Click here to view that:
