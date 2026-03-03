# Simple Server Information - Flask & Python - NginX untuk Reverse Proxy

---

# I. Pengujian pada VPC Default

## A. Persiapan

Buat 2 SG:
- flaskSG : ijinkan inbound rule port 22, 5000 (Flask Python) dari anywhere-IPv4 (0.0.0.0/0).
- nginxSG : ijinkan inbound rule port 22, 80 (NginX) dari anywhere-IPv4 (0.0.0.0/0).


---

## B. Pembuatan Internal Server dan Bastion Host (DMZ)

### B.1. EC2 Internal : Server Information Flask-Python 

AMI yang digunakan adalah Amazon Linux

- Name : internal
- AMI : Amazon Linux 2023 (kernel-6.1)
- Instance type : t2.nano
- key pair : vockey
- Security Group : flaskSG (22 dan 5000)


Masukkan ini sebagai script user data (di menu Advanced Details)

````
#!/bin/bash
# 1. Update sistem dan install dependencies
dnf update -y
dnf install python3-pip git -y

# 2. Berpindah ke home directory ec2-user
cd /home/ec2-user

# 3. Clone repository
# Pastikan nama folder hasil clone sesuai (biasanya sesuai nama repo atau folder tertentu)
git clone https://github.com/paknux/server_info-flask_python-nginx_reverse.git serverinfo-flask
cd serverinfo-flask

# 4. Install library Python
pip3 install flask psutil gunicorn

# 5. Buat Systemd Service File agar Flask jalan otomatis setelah restart
cat <<EOF | sudo tee /etc/systemd/system/flaskapp.service
[Unit]
Description=Gunicorn instance to serve Flask Server Info
After=network.target

[Service]
User=ec2-user
Group=ec2-user
WorkingDirectory=/home/ec2-user/serverinfo-flask
# Cari lokasi gunicorn otomatis
ExecStart=$(which gunicorn) --workers 3 --bind 0.0.0.0:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 6. Reload, Start, dan Enable Service
sudo systemctl daemon-reload
sudo systemctl start flaskapp
sudo systemctl enable flaskapp

# 7. Pastikan hak akses folder benar
sudo chown -R ec2-user:ec2-user /home/ec2-user/serverinfo-flask
````



---
### B.2. EC2 Bastion Host : NginX Reverse Proxy

- Name : bastion
- AMI : Ubuntu Server 24.04 LTS (HVM)
- Instance type : t2.nano
- key pair : vockey
- Security Group : nginxSG (22 dan 80)

1. Update dan install paket yang dibutuhkan
````
sudo apt update
sudo apt install nginx links -y
````


2. Edit file konfigurasi nginx untuk sites 
````
sudo nano /etc/nginx/sites-available/reverse-proxy
````

isi dengan

````
server {
    listen 80 default_server;
    listen [::]:80 default_server; # Untuk dukungan IPv6

    server_name _; # Karakter underscore berarti "tangkap semua domain"

    location / {
        proxy_pass http://172.16.x.x:5000; # IP Private Information Server kita (Amazon Linux)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
````

3. buat symbolic link dari available ke enabled
````
sudo ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/
````

4. Hapus konfigurasi default nginx

````
sudo rm /etc/nginx/sites-enabled/default
````

5. Test konfigurasi NginX dan restart NginX
````
sudo nginx -t
sudo systemctl restart nginx
````

---

## C. Pengujian
##### Gunakan browser

Pengujian Server Informasi Internal Flask-Python
````
http://ip_public_sever_internal:5000 
````

Pengujian Bastion Host NginX Reverse Proxy Server
````
http://ip_public_bastion_host 
````

---

# II. Penempatan Internal Server dan Bastion Host di VPC kantor
1. VPC akan dibuat secara manual untuk memperjelas pemahaman
2. Network VPC kantor : 10.100.0.0/16
3. subnet kantor-public : 10.100.1.0/24
4. subnet kantor-private : 10.100.2.0/24
5. tempatkan EC2 bastion (Nginx) di kantor-public
6. tempatkan EC2 internal (Flask-Python) di kantor-private

## A. Buat VPC kantor
1. Cari menu VPC
2. Klik tombol oranye : Create VPC
- VPC settings : pilih VPC only
- Name : kantor 
- IPv4 CIDR : 10.100.0.0/16
- klik tombol oranye Create VPC di bawah
- hasil dapat dilihat di menu kiri Your VPCs

## B. Buat SG di VPC kantor

Buat 2 SG di VPC kantor:
- flaskSG-kantor : ijinkan inbound rule port 22, 5000 (Flask Python) dari anywhere-IPv4 (0.0.0.0/0).
- nginxSG-kantor : ijinkan inbound rule port 22, 80 (NginX) dari anywhere-IPv4 (0.0.0.0/0).



1. Menggunakan User Data yang akan dieksekusi pada saat pertama kali pembuatan instance EC2
2. Jika menjadi kebijakan perusahaan (untuk penghematan dll),  mungkin perlu memasang Server MySQL (instance EC2) sendiri 
3. Menggunakan solusi Serverless (Elastic Beanstalk) untuk mendeploy PHP
4. Menggunakan Lambda dan API Gateway (migrasi ke bahasa pemrograman lain seperti Node.js)
5. Menggunakan CloudFormation (yaml) untuk membuat stack automation
