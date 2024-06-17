```
00- echo "192.168.5.241 hub.packops.local" >> /etc/hosts
```
01- Change External url to http (in http mode behind nginx )
vim common/config/core/env

1- ## download Harbor installaer
```
wget https://github.com/goharbor/harbor/releases/download/v2.9.1/harbor-offline-installer-v2.9.1.tgz
tar xvf harbor-offline-installer-v2.9.1.tgz
```


# 2- Get Certificate with Lets Encrypt 
```
 sudo certbot certonly --standalone --preferred-challenges http --agree-tos --email mrnickfetrat@gmail.com -d  hub.packops.dev
 
 ```
 
#  3- Generate Certificate on Server 

generate  certificate and add it in  harbor.yml
```
mkdir /opt/cert
cd /opt/cert

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=hub.packops.local" \
 -key ca.key \
 -out ca.crt
openssl genrsa -out hub.packops.local.key 4096
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=hub.packops.local" \
    -key hub.packops.local.key \
    -out hub.packops.local.csr

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=hub.packops.local
DNS.2=hub.packops.local
DNS.3=hostname
EOF
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in hub.packops.local.csr \
    -out hub.packops.local.crt



```
# 4- Trust CA on Client && Docker Login Config
```
docker run -it  --privileged  --name my-dind-container11 -v /var/run/docker.sock:/var/run/docker.sock docker:dind sh
echo "192.168.6.252 hub.packops.local" >> /etc/hosts
openssl s_client -showcerts -connect hub.packops.local:443 </dev/null | openssl x509 -outform PEM > ca.crt
cat ca.crt >> /etc/ssl/certs/ca-certificates.crt
mkdir ~/.docker/
echo '{
        "auths": {
                "hub.packops.local": {
                        "auth": "YWRtaW46SGFyYm9yMTIzNDU="
                }
        }
}' > ~/.docker/config.json
docker login hub.packops.local
```

# 5- Add configs in harbor.yml
```
https:
  # https port for harbor, default is 443
    port: 443
  # The path of cert and key files for nginx
    certificate: /opt/cert/hub.packops.local.crt
    private_key: /opt/cert/hub.packops.local.key



```


# 6  install docker 
```
sudo mkdir -p /etc/apt/keyrings

 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update &&  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-compose
```
# 7 Bring up harbor 

```
bash prepare
bash install.sh
```

``If you harbor is in Local make sure your domain name match with your server (make a host for you domain ) in my case 192.168.4.210 hub.packops.dev >> /etc/hosts ``

