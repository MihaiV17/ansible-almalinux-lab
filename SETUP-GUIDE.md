Mașina de control (Master): AlmaLinux 9 - IP: 192.168.1.198
Mașina target: AlmaLinux 9 - IP: 192.168.1.150
Conexiune: Rețea locală între mașinile virtuale
📦 Pas 1: Instalarea Ansible pe mașina master
Pe mașina master (192.168.1.198):
bash
# Actualizează sistemul
sudo dnf update -y

# Instalează EPEL repository
sudo dnf install epel-release -y

# Instalează Ansible
sudo dnf install ansible -y

# Verifică instalarea
ansible --version
🔑 Pas 2: Configurarea SSH Keys pentru autentificare
2.1. Generarea cheilor SSH pe master
bash
# Pe mașina master (192.168.1.198)
ssh-keygen -t rsa -b 4096

# Acceptă toate opțiunile default (apasă Enter pentru toate)
# Nu seta parolă pentru cheie (Enter la prompt-ul pentru parolă)
2.2. Copierea cheii publice pe target
 Folosind ssh-copy-id (automată):

bash
# Activează temporar autentificarea cu parolă pe target
# Pe target (192.168.1.150):
sudo nano /etc/ssh/sshd_config
# Setează: PasswordAuthentication yes
sudo systemctl restart sshd

# Pe master (192.168.1.198):
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.1.150

# Setează permisiunile corecte
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
chown root:root /root/.ssh/authorized_keys
2.3. Testarea conexiunii SSH
bash
# Pe master (192.168.1.198)
ssh root@192.168.1.150

# Ar trebui să te conectezi fără să ceară parolă
# Ieși cu: exit
📋 Pas 3: Configurarea Inventory-ului Ansible
3.1. Crearea directorului de proiect
bash
# Pe master (192.168.1.198)
mkdir ~/ansible-project
cd ~/ansible-project
3.2. Crearea fișierului inventory
bash
nano inventory
Conținutul fișierului inventory:

ini
[almalinux_servers]
ansible-master ansible_host=192.168.1.198 ansible_connection=local
almalinux-target ansible_host=192.168.1.150 ansible_user=root

[local]
ansible-master ansible_host=192.168.1.198 ansible_connection=local

[remote]
almalinux-target ansible_host=192.168.1.150 ansible_user=root

# Variabile pentru grupuri
[almalinux_servers:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[remote:vars]
ansible_user=root
ansible_ssh_private_key_file=/root/.ssh/id_rsa
ansible_become=yes
ansible_become_method=sudo
3.3. Testarea conectivității Ansible
bash
# Test ping pe toate mașinile
ansible -i inventory almalinux_servers -m ping

# Test comenzi simple
ansible -i inventory almalinux_servers -m command -a "hostname"
ansible -i inventory almalinux_servers -m command -a "uptime"
Rezultat așteptat:

ansible-master | SUCCESS => {"ping": "pong"}
almalinux-target | SUCCESS => {"ping": "pong"}
☕ Pas 4: Instalarea Java 11 cu Ansible
4.1. Crearea playbook-ului pentru Java
bash
nano install-java.yml
Conținutul playbook-ului:

yaml
---
- name: Install Java 11 on AlmaLinux servers
  hosts: almalinux_servers
  become: yes
  tasks:
    - name: Install OpenJDK 11
      dnf:
        name: java-11-openjdk
        state: present
      
    - name: Install OpenJDK 11 Development Kit
      dnf:
        name: java-11-openjdk-devel
        state: present
        
    - name: Verify Java installation
      command: java -version
      register: java_version
      
    - name: Display Java version
      debug:
        var: java_version.stderr
4.2. Rularea playbook-ului
bash
ansible-playbook -i inventory install-java.yml
4.3. Verificarea instalării
bash
# Verifică versiunea Java pe ambele mașini
ansible -i inventory almalinux_servers -m command -a "java -version"

# Verifică pachetele instalate
ansible -i inventory almalinux_servers -m command -a "rpm -qa | grep java-11-openjdk"
🌐 Pas 5: Instalarea și configurarea Nginx
5.1. Rezolvarea conflictelor de port
Problemă întâlnită: Apache (httpd) poate fi deja instalat și să folosească portul 80.

bash
# Verifică ce servicii folosesc portul 80
ansible -i inventory almalinux_servers -m command -a "netstat -tlnp | grep :80"

# Dacă Apache rulează, oprește-l
ansible -i inventory almalinux-target -m systemd -a "name=httpd state=stopped enabled=no" --become
5.2. Crearea playbook-ului pentru Nginx
bash
nano install-nginx.yml
Conținutul complet al playbook-ului este în fișierul install-nginx.yml

5.3. Rularea playbook-ului
bash
ansible-playbook -i inventory install-nginx.yml
5.4. Verificarea instalării Nginx
bash
# Verifică statusul serviciului
ansible -i inventory almalinux_servers -m command -a "systemctl status nginx"

# Verifică că portul 80 e folosit de Nginx
ansible -i inventory almalinux_servers -m command -a "netstat -tlnp | grep :80"

# Testează pagina web
ansible -i inventory almalinux_servers -m uri -a "url=http://localhost return_content=yes"
🌍 Pas 6: Verificarea finală
6.1. Testarea prin browser
Accesează în browser:

Mașina master: http://192.168.1.198
Mașina target: http://192.168.1.150
6.2. Verificarea prin comenzi
bash
# Test complet al serviciilor
ansible -i inventory almalinux_servers -m command -a "systemctl is-active nginx"
ansible -i inventory almalinux_servers -m command -a "java -version"

# Test conectivitate web
curl http://192.168.1.198
curl http://192.168.1.150
❗ Probleme întâlnite și soluții
1. Permission denied la SSH
Cauza: Cheia SSH nu e configurată corect sau permisiunile sunt greșite. Soluție:

Verifică că cheia e pe o singură linie în authorized_keys
Setează permisiuni: chmod 600 ~/.ssh/authorized_keys
2. Port 80 ocupat (Address already in use)
Cauza: Apache sau alt serviciu web rulează pe portul 80. Soluție:

bash
ansible -i inventory almalinux-target -m systemd -a "name=httpd state=stopped" --become
3. SELinux blochează SSH
Cauza: SELinux poate bloca accesul SSH keys. Soluție:

bash
restorecon -Rv /root/.ssh/
4. Ansible nu găsește inventory
Cauza: Rulezi din directorul greșit sau inventory nu există. Soluție:

Verifică că ești în directorul cu inventory: ls -la
Folosește calea completă: ansible -i ~/ansible-project/inventory
📊 Rezultate finale
După finalizarea tuturor pașilor:

✅ SSH Keys: Autentificare automată între servere
✅ Java 11: Instalat pe ambele servere
✅ Nginx: Configurat și funcțional pe portul 80
✅ Firewall: Configurat pentru trafic HTTP
✅ Pagini web: Accesibile prin browser pe ambele IP-uri
🔧 Comenzi utile pentru administrare
bash
# Restart servicii pe toate mașinile
ansible -i inventory almalinux_servers -m systemd -a "name=nginx state=restarted" --become

# Actualizează toate pachetele
ansible -i inventory almalinux_servers -m dnf -a "name='*' state=latest" --become

# Verifică spațiul pe disk
ansible -i inventory almalinux_servers -m command -a "df -h"

# Verifică memoria
ansible -i inventory almalinux_servers -m command -a "free -h"


