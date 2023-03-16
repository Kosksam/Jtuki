# Jtuki
NGINX palvelin Ubuntu


Web-palveluita ja niiden kuormituksen tasausta verkon järjestelmään listättiin NGINX-palvelin. Se toimii sekä web-palvelimena että välityspalvelimena (reverse proxy) kahteen muuhun www-palvelimeen.

https://github.com/NikoKoskinen/jarjestelmatuki/blob/main/nginx3.png

Organisaation ulkopuolelle näkyväksi web-palvelimeksi määritellään NGINX-palvelin. Palomuurisäännöt mahdollistavat liikenteen koneen http ja https portteihin. Palvelimelle voidaan sijoittaa yksinkertaisia staattisia www-sivuja. Dynaamiset, esim: tietokantaa käyttävät www-sivut on sijoitettu kahdelle identtiselle palvelimelle. NGINX-palvelin jakaa liikenteen tasan näiden kahden palvelimen välillä, jolloin kumpikaan dynaamisista www-palveluista ei tukehdu internetistä tuleviin sivupyyntöihin. suoraa liikennettä internistä näihin kahteen palvelimeen ei sallita.
Palvelimen asetuksista

Palvelimen asetusten osalta tärkein tiedosto on nginx.conf. Se määrittelee palvelimen käyttämät hakemistot ja tiedostotyypit sekä käytössä olevat osoitteet ja porttinumerot. Tiedostossa on joukko määreitä (directive), joilla palvelimen toimintaa ohjataan. Seuraavassa taulukossa tärkeimmät määreet:
Määre 	käyttötarkoitus
include 	kirjaston tai tiedoston lataus osaksi konfiguraatiota, esim. include mime.types lataa määrityksen erilaisista tunnistettavista tiedostotyypeistä
listen 	kuuntelevan portin määritys, esim. listen80
root 	hakemisto, joka toimii sivuston juurihakemistona, esim. root /var/sysmaintenance määrittelee hakemiston, jonka alle sivut on tallennettu
location 	url-määritys, esim. location /services määrittelee, mihin hakemistoon http-kysely ohjataan. Staattisessa palvelimessa tämä yleensä yhdistetään root-hakemistoon.
alias 	ohjaus toiseen hakemistoon location määrityksen yhteydessä
try_files 	määritys, minkä nimisiä tiedostoja voidaan käyttää url:n lataamiseen
upstream 	välityspalvelinlohko, jonka sisään määritellään taustapalvelimet (backend servers)
proxy_pass 	uudelleenohjauskomento välityspalvelinlohkoon

Seuraavassa esimerkissä on määritelty taustajärjestelmiksi kaksi palvelinta joiden geo-hakemistosta löytyvät Rasekon 3D-karttatiedot:

user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
}

http {
	sendfile on;
	tcp_nopush on;
	types_hash_max_size 2048;
	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	# SSL Settings
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;
        
        # Log settings
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	gzip on;
	include /etc/nginx/conf.d/*.conf;
	
	# Backend server definitions
	upstream backendserver {
		server 192.168.192.65:8080;
		server 192.168.192.70:8080;
	}

	# Server definitions
	server {
		listen 80;
		root /var/www/sysmaintenance;
	
		# Redirect to backend servers from /geo
		location /map {
			proxy_pass http://backendserver/geo;
		}
                location /geo {
			proxy_pass http://192.168.192.75:8080/geoserver;
	}
}

Taustapalvelimet

Kuormituksen tasausta varten luotiin 2 kpl Apache 2 -webpalvelimia. Ensimmäisessä palvelimessa käyttöjärjestelmänä on Debian 11 ja toisessa Linux Mint 21 Vanessa. Apache 2 -palvelinten konfiguraatiotiedostojen ja www root -hakemistojen polut ilmenevät seuraavasta taulukosta:
Palvelin 	Hakemisto tai tiedosto 	Polku
www1 	www root 	var/www/html
www1 	geo-hakemisto 	var/www/html/geo
www1 	apache2.conf 	/etc/apache2/apache2.conf
www1 	port.conf 	/etc/apache2/port.conf
www2 	www root 	var/www/html
www2 	geo-hakemisto 	var/www/html/geo
www2 	apache2.conf 	/etc/apache2/apache2.conf
www2 	port.conf 	/etc/apache2/port.conf

Molemmat palvelimet käyttävät http-porttina 8080-porttia. Salatun liikenteen https (443) portit on otettu pois käytöstä, koska taustapalvelun liikenteen salaaminen lisää latausviivetta. Tarvittaessa NGINX-palvelimelle voi asentaa varmenteen ja ottaa salauksen käyttöön. Sivustojen ylläpitäjää varten koneisiin asennettiin Webmin-sovellus, joka kuuntelee TCP-porttia 10000.

Linux Mint 21 -versio vaatii paketin asentamisen lataamalla asennustiedoston erikseen osoitteesta: https://sourceforge.net/projects/webadmin/files/webmin/2.013/webmin_2.013_all.deb/download?use_mirror=nav

Latauksen jälkeen se asennetaan komennolla sudo apt-get install ./webmin_2.013_all.deb

Debian koneeseen webmin asennetaan root-käyttäjänä seuraavasti:

apt-get install curl
curl -o setup-repos.sh https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh
sh setup-repos.sh
apt-get install webmin
