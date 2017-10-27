# Squid Proxy Server
![3](https://github.com/renisakhairiyah/proxysquid/blob/master/squid.jpg)
### Sekilas Tentang

(Penjelasan singkat proxy server)

### Instalasi
Topologi Jaringan 

 ![1](https://github.com/renisakhairiyah/proxysquid/blob/master/Topologi.png)


Kebutuhan Sistem:
- Ubuntu Server 16.04 LTS 
- Proxy server : Squid
- Web server : Apache2
- Log report proxy : Sarg

#### Konfigurasi Router MikroTik
1. Menambahkan IP address secara statik untuk interface lokal
    ```sh
    ip address add address=172.16.1.1./24 interface=lokal 
    ```
2. Menambahkan IP address secara dinamis untuk interface internet
    ```sh 
    ip dhcp-client add interface=internet add-default-route=yes 
    use-peer-dns=yes use-peer-ntp=yes default-route-distance=0
    ```
3. Membuat DHCP Server untuk client pada interface lokal
    ```sh
    ip dhcp-server> setup
    
    Select interface to run DHCP server on
    dhcp server interface: lokal
    
    Select network for DHCP addresses
    dhcp address space: 171.16.1.0/24
    
    Select gateway for given network
    gateway for dhcp network: 172.16.1.1
    
    Select pool of ip addresses given out by DHCP server
    addresses to give out: 192.168.1.2-192.168.0.253
    
    Select DNS servers
    dns servers: 172.16.1.1
   
    Select lease time
    lease time: 3d
    ```
4. Menambahkan rule untuk NAT sehingga dapat mengakses jaringan internet melalui interface internet
    ```sh
    ip firewall nat action=masquerade chain=srcnat out-interface=internet
    ```
5. Mengatur DNS server menjadi 8.8.8.8 agar client me-*request* DNS query
    ```sh
    ip dns set server=8.8.8.8 allow-remote-requests=yes
    ```

#### Konfigurasi Server
Meng-*update* package Ubuntu Server, kemudian *install* kebutuhan sistem seperti Squid, Apache2, dan Sarg. 
```sh
# apt-get update
# apt-get -y install squid
# apt-get -y install sarg
# apt-get -y install apache2 
```
##### Konfigurasi Squid Proxy
1. Mengatur konfigurasi squid pada file `/etc/squid/squid.conf`
    ```sh
    #baris 989: Mendefinisikan ACL lan dari network 172.16.1.0/24
    acl lan src 172.16.1.0/24
    
    #baris 1165: Menambahkan daftar situs yang akan diblok
    acl blok dstdomain "/etc/squid/blok.acl"
    
    #baris 1166: Tidak mengizinkan daftar situs yang akan diblok untuk 
    mengakses protokol http
    http_access deny blok

    #baris 1188: Memberi izin untuk protokol http untuk dapat diakses 
    melalui lan
    http_access allow lan
    
    #baris 5157: 
    request_header_access Referer deny all
    request_header_access X-Forwarded-For deny all
    request_header_access Via deny all
    request_header_access Cache-Control deny all
    
    #baris 5511: Menambahkan hostname
    visible_hostname proxy.komdatjarkom

    #baris 7625: Menginisialisasikan proxy tidak sebagai forwarded
    forwarded_for off
    ```
2. Membuat file `/etc/squid/blok.acl` untuk menyimpan daftar situs yang akan diblok
   ```sh
   nano /etc/squid/blok.acl
      ftp.debian.org
      facebook.com
   ```
3. Menjalankan service squid
    ```sh
    #systemctl restart squid
    ```
##### Konfigurasi Squid Log Report: Sarg 
1. Mengatur konfigurasi Sarg pada `file /etc/sarg/sarg.conf`
    ```sh
    #baris 120 jangan diberi komentar dan baris 121 beri komentar
    output_dir /var/www/html/squid-reports
    #output_dir /var/lib/sarg
    
    #baris 132: Agar dapat menggunakan daftar IP yang terdapat pada resolve
    resolve_ip yes
    
    #baris 377: Mengganti karakter sehingga menggunakan UTF-8
    charset UTF-8
    ```
2. Menambahkan daftar *host* yang tidak akan dimasukkan ke *log reports*, kemudian disimpan di file `/etc/sarg/exclude_hosts`
    ```sh
    127.0.0.1
    www.google.co.id
    ```    
3. Menambahkan file di `/etc/apache2/conf-available/sarg.conf` untuk menyimpan *log* dari network 172.16.1.0/24
    ```sh
    <Directory "var/www/html/squid-reports">
        Require local
        Require ip 172.16.1.0/24
    </Directory>
    ```
4. Membuat direktori untuk menyimpan *report* dari Squid 
    ```sh 
    # mkdir /var/www/html/squid-reports
    ```
5. Menjalankan service Sarg
    ```sh
    # a2enconf sarg
    # systemctl restart apache2
    ```
    
### Cara Pemakaian
Pengaturan dilakukan pada *web browser* di komputer *client*. 
![2](https://github.com/renisakhairiyah/proxysquid/blob/master/client-2.PNG)

### Pembahasan

### Referensi


[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)


   [dill]: <https://github.com/joemccann/dillinger>
   [git-repo-url]: <https://github.com/joemccann/dillinger.git>
   [john gruber]: <http://daringfireball.net>
   [df1]: <http://daringfireball.net/projects/markdown/>
   [markdown-it]: <https://github.com/markdown-it/markdown-it>
   [Ace Editor]: <http://ace.ajax.org>
   [node.js]: <http://nodejs.org>
   [Twitter Bootstrap]: <http://twitter.github.com/bootstrap/>
   [jQuery]: <http://jquery.com>
   [@tjholowaychuk]: <http://twitter.com/tjholowaychuk>
   [express]: <http://expressjs.com>
   [AngularJS]: <http://angularjs.org>
   [Gulp]: <http://gulpjs.com>

   [PlDb]: <https://github.com/joemccann/dillinger/tree/master/plugins/dropbox/README.md>
   [PlGh]: <https://github.com/joemccann/dillinger/tree/master/plugins/github/README.md>
   [PlGd]: <https://github.com/joemccann/dillinger/tree/master/plugins/googledrive/README.md>
   [PlOd]: <https://github.com/joemccann/dillinger/tree/master/plugins/onedrive/README.md>
   [PlMe]: <https://github.com/joemccann/dillinger/tree/master/plugins/medium/README.md>
   [PlGa]: <https://github.com/RahulHP/dillinger/blob/master/plugins/googleanalytics/README.md>
