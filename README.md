
# Guide de configuration Asterisk sur Ubuntu et sécurisation avec OpenVPN

## Étape 1 : Installation de Asterisk sur Ubuntu

1. **Mettre à jour le système** :
   ```bash
   sudo apt update
   sudo apt upgrade
   ```

2. **Installer les dépendances nécessaires** :
   ```bash
   sudo apt install wget build-essential subversion libncurses5-dev libssl-dev libxml2-dev libsqlite3-dev uuid-dev
   ```

3. **Télécharger et extraire Asterisk** :
   ```bash
   cd /usr/src
   sudo wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-18-current.tar.gz
   sudo tar zxvf asterisk-18-current.tar.gz
   cd asterisk-18.*
   ```

4. **Installer les modules de support** :
   ```bash
   sudo contrib/scripts/install_prereq install
   sudo contrib/scripts/get_mp3_source.sh
   ```

5. **Configurer et compiler Asterisk** :
   ```bash
   sudo ./configure
   sudo make menuselect
   sudo make
   sudo make install
   sudo make samples
   sudo make config
   sudo ldconfig
   ```

6. **Démarrer Asterisk** :
   ```bash
   sudo systemctl start asterisk
   sudo systemctl enable asterisk
   ```

## Étape 2 : Configuration de base d'Asterisk

1. **Configurer les utilisateurs SIP** :
   - Éditez le fichier `sip.conf` :
     ```bash
     sudo nano /etc/asterisk/sip.conf
     ```
   - Ajoutez les lignes suivantes :
     ```ini
     [general]
     context=default
     allowoverlap=no
     bindport=5060
     bindaddr=0.0.0.0
     disallow=all
     allow=ulaw
     allow=alaw
     allow=gsm

     [6001]
     type=friend
     context=phones
     host=dynamic
     secret=6001_password
     ```

2. **Configurer le dial plan** :
   - Éditez le fichier `extensions.conf` :
     ```bash
     sudo nano /etc/asterisk/extensions.conf
     ```
   - Ajoutez les lignes suivantes :
     ```ini
     [default]
     exten => 6001,1,Dial(SIP/6001,20)
     exten => 6001,n,Voicemail(6001,u)
     exten => 6001,s,Hangup()
     ```

3. **Redémarrer Asterisk pour appliquer les modifications** :
   ```bash
   sudo systemctl restart asterisk
   ```

## Étape 3 : Installation et configuration de OpenVPN

1. **Installer OpenVPN** :
   ```bash
   sudo apt install openvpn easy-rsa
   ```

2. **Configurer OpenVPN** :
   - **Créer le répertoire pour Easy-RSA** :
     ```bash
     make-cadir ~/openvpn-ca
     cd ~/openvpn-ca
     ```

   - **Configurer les variables Easy-RSA** :
     Éditez le fichier `vars` :
     ```bash
     nano vars
     ```
     Modifiez les lignes suivantes pour correspondre à vos informations :
     ```bash
     export KEY_COUNTRY="US"
     export KEY_PROVINCE="CA"
     export KEY_CITY="SanFrancisco"
     export KEY_ORG="Fort-Funston"
     export KEY_EMAIL="me@myhost.mydomain"
     export KEY_OU="MyOrganizationalUnit"
     export KEY_NAME="EasyRSA"
     ```

   - **Construire le PKI, le CA et le serveur** :
     ```bash
     source vars
     ./clean-all
     ./build-ca
     ./build-key-server server
     ./build-dh
     openvpn --genkey --secret keys/ta.key
     ```

   - **Créer les certificats et clés pour les clients** :
     ```bash
     ./build-key client1
     ```

   - **Configurer OpenVPN** :
     ```bash
     sudo cp ~/openvpn-ca/keys/ca.crt /etc/openvpn/
     sudo cp ~/openvpn-ca/keys/server.crt /etc/openvpn/
     sudo cp ~/openvpn-ca/keys/server.key /etc/openvpn/
     sudo cp ~/openvpn-ca/keys/dh2048.pem /etc/openvpn/
     sudo cp ~/openvpn-ca/keys/ta.key /etc/openvpn/
     sudo nano /etc/openvpn/server.conf
     ```

     Ajoutez les lignes suivantes à `server.conf` :
     ```ini
     port 1194
     proto udp
     dev tun
     ca ca.crt
     cert server.crt
     key server.key
     dh dh2048.pem
     tls-auth ta.key 0
     cipher AES-256-CBC
     auth SHA256
     keepalive 10 120
     user nobody
     group nogroup
     persist-key
     persist-tun
     status openvpn-status.log
     log /var/log/openvpn.log
     verb 3
     ```

   - **Activer le transfert IP** :
     ```bash
     sudo nano /etc/sysctl.conf
     ```
     Décommentez la ligne suivante :
     ```bash
     net.ipv4.ip_forward=1
     ```
     Appliquez la modification :
     ```bash
     sudo sysctl -p
     ```

   - **Configurer les règles iptables** :
     ```bash
     sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
     sudo iptables-save | sudo tee /etc/iptables/rules.v4
     ```

   - **Démarrer OpenVPN** :
     ```bash
     sudo systemctl start openvpn@server
     sudo systemctl enable openvpn@server
     ```

3. **Configurer le client OpenVPN** :
   - **Créer un fichier de configuration pour le client** :
     ```bash
     nano client.ovpn
     ```
     Ajoutez les lignes suivantes :
     ```ini
     client
     dev tun
     proto udp
     remote your_server_ip 1194
     resolv-retry infinite
     nobind
     user nobody
     group nogroup
     persist-key
     persist-tun
     ca ca.crt
     cert client1.crt
     key client1.key
     remote-cert-tls server
     tls-auth ta.key 1
     cipher AES-256-CBC
     auth SHA256
     verb 3
     ```

   - **Transférer les fichiers nécessaires (ca.crt, client1.crt, client1.key, ta.key, client.ovpn) sur le client OpenVPN**.

   - **Lancer le client OpenVPN** :
     ```bash
     sudo openvpn --config client.ovpn
     ```

## Étape 4 : Installation et configuration de Linphone

1. **Installer Linphone** :
   ```bash
   sudo apt install linphone
   ```

2. **Configurer Linphone** :
   - Ouvrez Linphone et configurez un compte SIP avec les paramètres suivants :
     - **SIP Identity** : sip:6001@<ip_asterisk>
     - **Username** : 6001
     - **Password** : 6001_password
     - **Domain** : <ip_asterisk>

## Étape 5 : Vérification et tests

1. **Connectez Linphone à votre serveur Asterisk via le tunnel OpenVPN**.
2. **Passez un appel de test** en composant `6001`.

En suivant ces étapes, vous devriez être en mesure de configurer Asterisk sur Ubuntu, le sécuriser avec OpenVPN et utiliser Linphone pour passer des appels. N'hésitez pas à ajuster les configurations en fonction de vos besoins spécifiques.
