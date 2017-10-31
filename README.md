# Squid Proxy Server
![1](https://github.com/renisakhairiyah/proxysquid/blob/master/squid.jpg)
### Sekilas Tentang
   Proxy atau biasa disebut Proxy server merupakan server yang diletakkan di antara suatu aplikasi client dan aplikasi server yang dihubungi.Proxy server yang diletakkan diantara aplikasi client dan aplikasi server tersebut dapat digunakan untuk mengendalikan maupun memonitor lalu-lintas paket data yang melewatinya. Aplikasi client dapat berupa browser web, client FTP, dan sebagainya. Aplikasi server dapat berupa server web, server FTP dan sebagainya.
   Selain fungsi *caching*, proxy server juga dapat digunakan untuk membuat kebijakan keamanan di jaringan lokal. Aplikasi proxy server yang cukup populer adalah Squid. Squid merupakan aplikasi *server* yang stabil dengan *performance* yang tinggi dan juga merupakan aplikasi *web* proxy yang fleksibel untuk digunakan sebagai *web cache*.

### Instalasi
Topologi Jaringan 

 ![2](https://github.com/renisakhairiyah/proxysquid/blob/master/Topologi.png)


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
#### Konfigurasi Squid Proxy
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
#### Konfigurasi Squid Log Report: Sarg 
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
Pengaturan dilakukan pada *web browser* di komputer *client*. Pengaturan ini diperlukan karena pada implementasi saat ini tidak menerapkan transparent proxy. 
![3](https://github.com/renisakhairiyah/proxysquid/blob/master/client-2.PNG)

### Monitoring Log Proxy: Sarg
Monitoring dapat dilakukakan di *web browser* dengan mengetikkan uri `172.16.1.254/squid-reports/`.
![4](https://github.com/renisakhairiyah/proxysquid/blob/master/report.PNG)

![5](https://github.com/renisakhairiyah/proxysquid/blob/master/report1.PNG)

![6](https://github.com/renisakhairiyah/proxysquid/blob/master/report2.PNG)

### Pembahasan
**Cara Kerja**
Cara kerja proxy server adalah ketika ada *client* yang mengkses suatu alamat web, misalkan *Client* A mengakses ke alamat www.github.com, maka proxy akan menyimpan file-file halaman web tersebut ke dalam *cache* lokal proxy tersebut. Ketika ada *client* lain, misalkan *Client* B mengakses halaman web yang sama dari www.github.com maka proxy server akan melakukan pengecekan ke server yang dituju (www.github.com), apakah obyek yang disimpan di *cache local proxy* masih sama dengan yang ada di server web tujuan, apabila tidak ada perubahan obyek pada server maka proxy akan memberikan obyek yang diminta ke *client* B tersebut, dan apabila ternyata telah ada perubahan barulah proxy server memintakannya ke server (www.github.com) untuk *Client* B yang mengakses server web tersebut, sementara itu file yang diberikan kepada client tersebut juga akan disimpan di direktori *cache* pada proxy sever, dan begitu seterusnya sehingga secara tidak langsung metode ini akan menghemat *bandwidth* dan mempercepat koneksi internet.
  
**Kelebihan Proxy Server Squid adalah sebagai berikut:**
1. Kestabilannya untuk menangani sebuah jaringan yang berskala besar.
2. Adanya *content caching* memungkinkan untuk mengakses ulang web menjadi lebih cepat dan dapat menghemat *bandwidth*.
3. Kemampuan *filtering*.
4. Gratis, dibawah *General Public License*(GPL/GNU).

**Kelemahan Proxy server diantaranya:**
1. Penggunaan proxy server squid menggunakan jumlah RAM yang cukup besar
2. Pengaksesan ke alamat yang belum masuk ke proxy jadi lebih lambat karena harus melalui proxy terlebih dahulu
3. Bila proxy server terlambat melakukan *update cache*, maka *client* akan mendapatkan *content* yang belum diperbaharui
4. Squid tidak dapat memblokir HTTPS

### Referensi
Arjuni S. Perancangan Dan Implementasi Proxy Server Dan Manajemen Bandwidth Menggunakan Linux Ubuntu Server [Internet]. [diacu 2017 Oktober 29]. Tersedia dari: http://www.academia.edu/4045548/JURNAL_PA_PERANCANGAN_DAN_IMPLEMENTASI_PROXY_SERVER_DAN_MANAJEMEN_BANDWIDTH_MENGGUNAKAN_LINUX_UBUNTU_SERVER

Rifqi M. Mengenal Proxy Server [Internet]. [diacu 2017 Oktober 29]. Tersedia dari: http://masrifqi.staff.ugm.ac.id/doc/Squid-proxy-server.pdf

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
