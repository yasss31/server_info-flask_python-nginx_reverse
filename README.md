## Simple Server Information - Flask & Python - NginX untuk Reverse Proxy
#### Menggunakan solusi Serverless  Elastic Beanstalk di lingkungan **AWS Academy**.
---

![Arsitektur Project](img/arsitektur.png)

---

## I. Persiapan

Buat 2 SG:
- flaskSG : ijinkan inbound rule port 22, 5000 (Flask Python) dari anywhere-IPv4 (0.0.0.0/0).
- nginxSG : ijinkan inbound rule port 22, 80 (NginX) dari anywhere-IPv4 (0.0.0.0/0).


---

## II. Pembuatan Internal Server dan Bastion Host (DMZ)

### A. EC2 Internal : Server Information Flask-Python 

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
### B. EC2 Bastion Host : NginX Reverse Proxy

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
### A. Buat RDS

1. Buka Aurora and RDS
2. Klik create database
3. Choose a database creation method : Full Configuration
4. Engine type : MySQL
5. Templates : Sandbox
6. Availability and durability : otomatis terpilih Single-AZ DB instance deployment (1 instance)
7. DB instance identifier : database-1
8. Master username : (admin) boleh diganti
9. Credentials management : Self managed
10. Master password : (P4ssw0rd) boleh diganti
Confirm master password : (P4ssw0rd) boleh diganti


11. Public access : No, kalau butuh diakses dari luar buat jadi Yes
12. VPC security group (firewall) : Choose existing, pilih yang sudah dibuat tadi
13. Klik create database
14. Tunggu sampai mendapatkan End Point

---

### B. Membuat dan Konfigurasi S3 Bucket
S3 Bucket dapat dibuat dengan Web GUI Management Console seperti biasa, 


## Buat S3
1. Buka Amazon S3 (cari S3)
2. Klik Create bucket 
   - Bucket Type : General purpose
   - Bucket Name : nug-php-mysql-s3-env-key
   - Object Ownership
        - bebas pilih : <h2> ACLs disabled </h2> atau <h2> ACLs enabled </h2>
   - Block Public Access settings for this bucket
   - pastikan Block all public access TIDAK DICENTANG
   - jangan lupa CENTANG acknowledge that the current settings
3. klik Create bucket

---

## II. Deploy App ke Serverless ELastic Beanstalk

---

## Persiapan

### 1. Unduh file dari repo https://github.com/paknux/serverless-php-mysql-s3-env-key.git
4 buah file utama yang diperlukan: 
- index.php
- composer.json
- composer.lock
- .env

### 2. Edit Environment Variable .env
Environment variable dapat berupa file .env atau dapat merupakan environment dari OS. Editlah file .env dengan aplikasi text editor, sehingga memuat hal berikut ini:

### .env
````
DB_HOST=database-1.ccqnofwkwmzs.us-east-1.rds.amazonaws.com
DB_PORT=3306
DB_NAME=db_karyawan
DB_USER=admin
DB_PASS=P4ssw0rd

AWS_REGION=us-east-1
AWS_BUCKET=nug-php-mysql-s3-env-key
````


### 3. Zip (compress) 3 file terebut
Kompress 4 file tersebut menjadi .zip. Harus .zip, tidak boleh .rar atau format kompresi yang lain. <br>
Hasil kompresi misalnya menjadi app.zip

---

## Elastic Beanstalk

1. Buka Elastic Beanstalk

2. Klik create application

   Isi Application information dan Application name misal dengan nama <b>tokoorange</b>

3. Klik create


4. Klik create new environment

   - Environment tier : biarkan Web server environment
   - Environment name : tokoorange-env
   - Platform : PHP

   - Application code : Upload your code
      - Local file
      - Version label : ver
      - klik choose file
         tunjukkan lokasi app.zip

   - klik Next

5. Configure service access 
   - Service role : LabRole
   - EC2 instance profile : LabIbstanceProfile
   - EC2 key pair - optional : vockey

   - klik Next

6. Set up networking, database, and tags - optional
   tidak ada yang perlu diubah

   - klik Next


7. Configure instance traffic and scaling - optional 
   - EC2 security groups : pilih yang ijinkan port 80 443

   - klik Next


8. Configure updates, monitoring, and logging - optional 
   - Health reporting : Basic
   - Managed updates : guang centangnya (Disable) 

   - klik Next

9. Setelah melihat review, klik Create

---

## III. Pengujian
##### Gunakan browser
````
http://ip_public
````

![Hasil Pengujian](img/hasilpengujian.png)

---

## IV. Pengembangan
1. Menggunakan User Data yang akan dieksekusi pada saat pertama kali pembuatan instance EC2
2. Jika menjadi kebijakan perusahaan (untuk penghematan dll),  mungkin perlu memasang Server MySQL (instance EC2) sendiri 
3. Menggunakan solusi Serverless (Elastic Beanstalk) untuk mendeploy PHP
4. Menggunakan Lambda dan API Gateway (migrasi ke bahasa pemrograman lain seperti Node.js)
5. Menggunakan CloudFormation (yaml) untuk membuat stack automation
