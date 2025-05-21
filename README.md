# Mise en place d'une Infrastructure à Clés Publiques (PKI) à deux niveaux  

## Introduction

Ce document détaille la mise en place d'une Infrastructure à Clés Publiques (PKI) à deux niveaux comprenant une autorité de certification racine (Root CA) et une autorité de certification intermédiaire (Intermediate CA), puis la configuration d'un serveur web Apache avec HTTPS.
Préparation de l'environnement

## Cahier des charges

- Créer une paire de clés et un certificat autosigné pour l'autorité de certification « root».
- Créer une paire de clés et un certificat électronique signé par l'autorité de certification « root » pour obtenir une autorité de certification intermédiaire (chaîne de certification.)
- Créer une paire de clés et un certificat électronique validé par l'AC « intermédiaire» pour un serveur.
- Présenter les certificats.
- Documenter (faire un fichier ms-word) des commandes utilisées.
- Implémenter dans un serveur web (Nginx ou Apache) le protocole HTTPS avec le certificat du serveur.
- Montrer son fonctionnement à l'aide d'une connexion avec un navigateur


Commençons par créer une structure de répertoires pour notre PKI :

```
# Création de la structure pour la Root CA
mkdir -p ~/pki/{root-ca,intermediate-ca,server,apache}
cd ~/pki
mkdir -p root-ca/{certs,crl,newcerts,private}
chmod 700 root-ca/private
touch root-ca/index.txt
echo 1000 > root-ca/serial
```

### Création de la structure pour l'Intermediate CA
mkdir -p intermediate-ca/{certs,crl,csr,newcerts,private}
chmod 700 intermediate-ca/private
touch intermediate-ca/index.txt
echo 1000 > intermediate-ca/serial
echo 1000 > intermediate-ca/crlnumber

Configuration OpenSSL
Fichier de configuration pour la Root CA

Créons le fichier de configuration pour la Root CA (~/pki/root-ca/openssl.cnf) :

```
# OpenSSL root CA configuration file

[ ca ]
default_ca = CA_Mich

[ CA_Mich ]
# Directory and file locations
dir               = ~/pki/root-ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem

# For certificate revocation lists
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 3650
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the 'req' tool
default_bits        = 4096
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Defaults
countryName_default             = FR
stateOrProvinceName_default     = France
localityName_default            = Paris
0.organizationName_default      = esgimich
organizationalUnitName_default  = Root CA
commonName_default              = Root CA
emailAddress_default            = admin@esgimich.fr

[ v3_ca ]
# Extensions for a typical CA
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
# Extensions for server certificates
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```

### Fichier de configuration pour l'Intermediate CA

Créons le fichier de configuration pour l'Intermediate CA (~/pki/intermediate-ca/openssl.cnf) :

```
# OpenSSL intermediate CA configuration file

[ ca ]
default_ca = CA_interm

[ CA_interm ]
# Directory and file locations
dir               = ~/pki/intermediate-ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The intermediate key and certificate
private_key       = $dir/private/intermediate.key.pem
certificate       = $dir/certs/intermediate.cert.pem

# For certificate revocation lists
crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_loose

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the 'req' tool
default_bits        = 4096
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Defaults
countryName_default             = FR
stateOrProvinceName_default     = France
localityName_default            = Paris
0.organizationName_default      = esgimich
organizationalUnitName_default  = IntermediateCA
commonName_default              = Intermediate CA
emailAddress_default            = admin@esgimich.fr

[ v3_ca ]
# Extensions for a typical CA
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
# Extensions for server certificates
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ crl_ext ]
authorityKeyIdentifier=keyid:always
```

### 1. Création de la Root CA
Génération de la clé privée pour la Root CA

```
cd ~/pki
openssl genrsa -aes256 -out root-ca/private/ca.key.pem 4096
chmod 400 root-ca/private/ca.key.pem
```  

Création du certificat auto-signé pour la Root CA

```
cd ~/pki
openssl req -config root-ca/openssl.cnf \
    -key root-ca/private/ca.key.pem \
    -new -x509 -days 3650 -sha256 -extensions v3_ca \
    -out root-ca/certs/ca.cert.pem
chmod 444 root-ca/certs/ca.cert.pem
```  

Vérification du certificat Root CA

```
openssl x509 -noout -text -in root-ca/certs/ca.cert.pem
```
![alt text](image.png)

### 2. Création de l'Intermediate CA
Génération de la clé privée pour l'Intermediate CA

```
cd ~/pki
openssl genrsa -aes256 -out intermediate-ca/private/intermediate.key.pem 4096
chmod 400 intermediate-ca/private/intermediate.key.pem
```

Création de la demande de signature de certificat (CSR) pour l'Intermediate CA

```
cd ~/pki
openssl req -config intermediate-ca/openssl.cnf -new -sha256 \
    -key intermediate-ca/private/intermediate.key.pem \
    -out intermediate-ca/csr/intermediate.csr.pem
```

Signature du certificat intermédiaire avec la Root CA

```
cd ~/pki
openssl ca -config root-ca/openssl.cnf -extensions v3_intermediate_ca \
    -days 3650 -notext -md sha256 \
    -in intermediate-ca/csr/intermediate.csr.pem \
    -out intermediate-ca/certs/intermediate.cert.pem
chmod 444 intermediate-ca/certs/intermediate.cert.pem
```

Vérification du certificat intermédiaire

```
openssl verify -CAfile root-ca/certs/ca.cert.pem \
    intermediate-ca/certs/intermediate.cert.pem
```

Création de la chaîne de certificats

```
cd ~/pki
cat intermediate-ca/certs/intermediate.cert.pem \
    root-ca/certs/ca.cert.pem > intermediate-ca/certs/ca-chain.cert.pem
chmod 444 intermediate-ca/certs/ca-chain.cert.pem
```

### 3. Création du certificat pour le serveur web
Génération de la clé privée pour le serveur

```
cd ~/pki
openssl genrsa -out server/server.key.pem 2048
chmod 400 server/server.key.pem
```

Création de la demande de signature de certificat (CSR) pour le serveur

Créons un fichier de configuration pour le CSR du serveur avec support SAN (~/pki/server/server.cnf) :

```
[req]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
C = FR
ST = France
L = Paris
O = esgimich
OU = WebServer
CN = localhost

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
```

Générons le CSR avec ce fichier de configuration :

```
cd ~/pki
openssl req -config server/server.cnf \
    -key server/server.key.pem \
    -new -sha256 -out server/server.csr.pem
```  

Signature du certificat serveur avec l'Intermediate CA

```
cd ~/pki
openssl ca -config intermediate-ca/openssl.cnf \
    -extensions server_cert -days 365 -notext -md sha256 \
    -in server/server.csr.pem \
    -out server/server.cert.pem
chmod 444 server/server.cert.pem
```  

Vérification du certificat serveur

```
cd ~/pki
openssl verify -CAfile intermediate-ca/certs/ca-chain.cert.pem \
    server/server.cert.pem
```  

Création de la chaîne complète pour le serveur

```
cd ~/pki
cat server/server.cert.pem \
    intermediate-ca/certs/intermediate.cert.pem \
    root-ca/certs/ca.cert.pem > server/fullchain.pem
chmod 444 server/fullchain.pem
```  

4. Configuration d'Apache avec HTTPS
Installation d'Apache et activation du module SSL

```
sudo apt-get update
sudo apt-get install apache2
sudo a2enmod ssl
sudo systemctl restart apache2
```  

Préparation des certificats pour Apache

```
cd ~/pki
sudo mkdir -p /etc/ssl/esgimich
sudo cp server/server.key.pem /etc/ssl/esgimich/
sudo cp server/server.cert.pem /etc/ssl/esgimich/
sudo cp server/fullchain.pem /etc/ssl/esgimich/
```  

### Configuration du site SSL dans Apache

Créons ou modifions le fichier de configuration pour notre site HTTPS :

```
sudo nano /etc/apache2/sites-available/default-ssl.conf
```  

Contenu du fichier :

```
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin webmaster@esgimich.fr
        ServerName localhost
        
        DocumentRoot /var/www/html
        
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        
        SSLEngine on
        SSLCertificateFile /etc/ssl/esgimich/server.cert.pem
        SSLCertificateKeyFile /etc/ssl/esgimich/server.key.pem
        SSLCertificateChainFile /etc/ssl/esgimich/fullchain.pem
        
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
            SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
            SSLOptions +StdEnvVars
        </Directory>
    </VirtualHost>
</IfModule>
```  

Activation du site SSL et redémarrage d'Apache

```
sudo a2ensite default-ssl.conf
sudo systemctl restart apache2
```  

### Création d'une page de test

```
echo "<html><body><h1>HTTPS fonctionne avec notre PKI !</h1></body></html>" | sudo tee /var/www/html/index.html
```  

### 5. Test de la configuration HTTPS
Ajout de l'entrée DNS locale

Pour tester localement, ajoutons une entrée dans le fichier /etc/hosts :

```
echo "127.0.0.1 localhost" | sudo tee -a /etc/hosts
```  

### Test avec un navigateur

Ouvrez un navigateur et accédez à https://localhost. Comme nous utilisons notre propre autorité de certification, il faudra ajouter le certificat racine aux autorités de confiance du navigateur pour éviter les avertissements de sécurité.

Pour Firefox, allez dans :

- Paramètres > Vie privée et sécurité > Certificats

- Afficher les certificats > Autorités

- Importer... > Sélectionnez le fichier ~/pki/root-ca/certs/ca.cert.pem

- Cochez "Confirmer cette AC pour identifier des sites web"

Vérification de la chaîne de certificats

Après avoir configuré votre navigateur pour faire confiance à votre Root CA, vous devriez pouvoir accéder à https://localhost sans avertissement de sécurité. Vous pouvez vérifier la chaîne de certificats en cliquant sur le cadenas dans la barre d'adresse et en examinant les détails du certificat.

On voit alors une chaîne de trois certificats :

- Le certificat du serveur (localhost)

- Le certificat de l'Intermediate CA

- Le certificat de la Root CA

Conclusion

Nous avons mis en place avec succès une Infrastructure à Clés Publiques (PKI) à deux niveaux comprenant une autorité de certification racine, une autorité de certification intermédiaire, et un certificat pour serveur web. Nous avons également configuré Apache pour utiliser ce certificat en HTTPS.

Cette infrastructure peut être utilisée dans un environnement de production pour sécuriser les communications web internes à une organisation ou pour des tests avant de déployer des certificats émis par une autorité de certification publique.
