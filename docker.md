#Docker

Docker je open soucrce projekat pokrenut od strane ljudi iz dotCloud-a. S tehnicke strane, Docker je napisan da bi olaksao koriscenje dvije postojece tehnologije:

- LXC: Linux Containers, koji dozvoljavaju izvrsavanje pojedinih procese na vecem nivou izolacije nego obicni Linux procesi. Kaze se da se proces izvrsava u kontejneru. Kontejneri obezbjedjuju izolaciju na novou:
	- Fajl sistema: kontejner moze pristupiti samo svom sandboxovanom fajl sistemu, osim ako nije podignut u tudjem.
	- User namespace: kontejner ima svoju korisnicku bazu.
	- Process namespace-a: unutar kontejnera su vidljivi samo procesi koji su njegov dio.
	- Network namespace-a: kontejner dobija svoj virtuelni mrezni uredjaj i virtuelnu IP adresu(tako da moze da koristi koji god port, bez kolizije).

- AUFS: advanced multi layered unification filesystem, koji se koristi za kreiranje union, copy-on-write fajl sistema. 

Ovo je lista Docker rezultata na deployment i operativne probleme:

- Izolacija: Docker izoluje palikacije na fajl i mreznom novou. Imate osjecaj kao da koristite pravu virtuelnu masinu, u tom smislu. 
- Reproduktivnost: Pripremite sistem onako kako odgovara, onda commit-ujte izmjene kao image. Sada mozete instancirati koliko god image-a i prenijeti ih na drugu masinu. 
- Bezbijednost: Docker kontejneri su bezbjedniji od obicnie izolacije procesa. 
-Ogranicenost resursa: Docker trenutno podrzava ograniceno koriscenje CPU, memorija takodje moze biti ogranicena. Ogranicavanje prostora na disku jos nije podrzano.
- Instalacija: Doker ima Docker Index - repozitorijum sa gotovim image-ima koji se instaliraju jednom komandom. 
- Uklanjanje: Kad aplikacija vise ne treba, unistimo njen kontejner. 
- Upgrades, downgrades: Boot up noviju verziju aplikacije, pa onda prebacite load balancer sa starog porta na novi. 
- Snapshotting, backing up: Docker podrzava commit-ovanje i tagovanje koji su momentalni. 

###Instalacija

Ubuntu Precise 12.04 (LTS) (64-bit)
[link](http://docs.docker.io/installation/ubuntulinux/#ubuntu-precise-1204-lts-64-bit)
```
	# install the backported kernel
	sudo apt-get update
	sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring

	# reboot
	sudo reboot
```

Provjerite da li APT sistem moze da radi sa https URL-ovim. Fajl ``/usr/lib/apt/methods/https`` bi trebalo da postoji. Ako ga nema, instalirajte paket:

```
  apt-get update
  apt-get install apt-transport-https
```
Dodajte docker repozitorijum:

```
	sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9

	sudo sh -c "echo deb https://get.docker.io/ubuntu docker main\
	> /etc/apt/sources.list.d/docker.list"
	sudo apt-get update
	sudo apt-get install lxc-docker
```
Provjera:

``sudo docker run -i -t ubuntu /bin/bash``

###Dockerfiles

Dockerfiles se mogu posmatrati kao image reprezentacija. Obezbjedjuju jednostavnu sitaksu za pravljenje slika i odlican su nacin za automatizaciju skripotava za njihovo kreiranje. 

Sve Docker instrukcije izgledaju ovako:

`` INSTRUCTION argument ``

####FROM
Prva instrukcija svakog Dockerfile-a mora biti:

`` FROM  image ``

Time postavljamo image koja ce da bude u osnovi. 
Ako image nije pronadjen na hostu, docker ce poci na docker image index, pronaci i skinuti je. 

```
	# Usage: FROM [image name]
	FROM ubuntu
```

####MAINTAINER

Moze se nalaziti bilo gdje u fajlu, praktikuje se da je pri pocetku, posle FROM. Navodi ko je autor. 

```
	# Usage: MAINTAINER [name]
	MAINTAINER authors_name
```

####RUN

Centralna komanda koja pokrece izvrsavanje i to iz image-a. Sve promjene se commituju i kreiraju novi sloj (layer). 

```
	# Usage: RUN [command]
	RUN aptitude install -y riak
```

####USER

Setuje UID korisnika.

```
	# Usage: USER [UID]
	USER 751
```

####VOLUME

Koristi se da omoguci pristup host masini iz kontejnera.

```
	# Usage: VOLUME ["/dir_1", "/dir_2" ..]
	VOLUME ["/my_files"]
```

####WORKDIR

Odredjuje gdje ce se izvrsiti komande CMD.

```
	# Usage: WORKDIR /path
	WORKDIR ~/
```

####ENTRYPOINT

Ove komande se izvrsavaju pri pokretanju Dockerfile-a. 

```
	# Usage: ENTRYPOINT [command]
		ENTRYPOINT echo “Hello world”  
```

####EXPOSE

,,Eksponira" port. 

```
	#Usage: EXPOSE [port]
	EXPOSE 4444

```
##Uvodni tutorijal

```
	md dockerfiles
	cd dockerfiles
	cat Dockerfile
``` 
Dockerfile:

```
	FROM ubuntu
	# make sure the package repository is up to date
	RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list
	RUN apt-get update
	apt-get install -y memcached
```

```
	docker build . 
```

Ili, ako dajemo ime image-u koji gradimo:

```
	docker build -t memcached .  #TAG
```

```
	MAINTAINER Ivana Corovic, ivanacorovic55@gmail.com
```
####Komentari

Kao u Rubiju (#)

[Provjera](https://www.docker.io/learn/dockerfile/level1/)

```
	ENTRYPOINT echo “Whale You Be My Container?”  #ovo ce se pojaviti cim pokrenete kontejner
	ENTRYPOINT ["wc", "-l"] # broji linije fajla
	ENTRYPOINT ["memcached", "-u", "daemon"] # pokrenuce memcached kao korisnik deamon
	# gornja linija je  ekvivalentna sledecim:
	ENTRYPOINT ["memcached"]
	USER daemon
```

```
	# expose memcached port
	EXPOSE 11211
```


`` docker ``

![Docker komande](/home/ivana/Downloads/docker.jpg) 

`` docker pull fcat/rails-getting-started ``

`` docker run -d -p 5000:3000 fcat/rails-getting-started `` #port 3000 forwarded to 5000

`` CONTAINER_ID=$(docker run -d -p 5000:3000 fcat/rails-getting-started) `` # uobicajen nacin da se sacuva kontejner

`` docker start $CONTAINER_ID `` #stop/wait

Komande:

- start:      Start/inicijalizuj kontejner
- stop:       Stop pokrenuti kontejner
- restart:    Restart
- destroy:    Stop i remove
- ssh:        Start bash shell u pokrenutom direktorijumu
- logs:       Docker logs za taj kontejner
- bootstrap:  Bootstrap kontejner za config na osnovu template-a

Primjer:

`` docker run -t -i 5000:3000 fcat/rails-getting-started /bin/bash `` #run -t -i startuje interaktivnu sesiju


```
	mkdir -p $APPS/kafka/data
	mkdir -p $APPS/kafka/logs
	sudo docker rm kafka > /dev/null 2>&1
	KAFKA=$(docker run \
		-d \
		-p 9092:9092 \
		-v $APPS/kafka/data:/data \
		-v $APPS/kafka/logs:/logs \
		-name kafka \
		-link zookeeper:zookeeper \
		server:4444/kafka)
	echo "Started KAFKA in container $KAFKA"
```

1-2.) ``mkdir -p $APPS/kafka/data (+logs)`` pravi direktorijum na doker hostu (ne unutar kontejnera) da bi se sacuvali podaci i logovi ako se kontejner restartuje. Mora da se kreira direktorijum jer javlja gresku u suprotnom. 

3.) Pokrecemo ``doker rm`` da uklonimo kotejner da bi mogli ponovo da iskoristimo ime. Kontejneri ostaju u fajl sistemu i kad su ugaseni, tako da ovim izbjegavamo konflikte. 

4.) ``docker run`` komandu cuvamo u promjenljuvoj $KAFKA u slucaju da nam kasnije zatreba. Na ovaj nacin cuvamo ID instance startovanog kontejnera.

5.) ``-d`` pokrece kontejner kao damon.

6.) ``-p`` podesava redirekciju portova. Podrazumijevani port Kafke je 9092: prvi je za dokcker host,a  drugi za kontejner. 

7.) ``-v`` omogucava da host i kontejner dijele prostor, da se sinhronizuje dio fajl sistema. Dozvoljava kontejneru da cuva log i ostale podatke za buducu analizu, kao i ucitavanje testnih podataka. Takodje obezbjedjuje prostor koji moze da se obrise, ako je potrebno.


8.) ``-name`` je nova funkcionalnost imenovanja u Dokeru. Omogucava da instancama dajemo pogodnija imena i da liknujemo kontejnere po njima.  

9.) ``-link zookeeper:zookeeper`` komanda omogucava linkovanje kontejnera. Inace su kontejneri izolovani, ali Docker 0.6.5 omogucava likovanje kontejnera, radi medjusobne komunikacije i dijeljenih promjenljivih. U ovom slucaju ce Kafka moci da nadje IP adresu Zookeeper-a i znati gdje da sacuva svoje informacije koje cekaju unutar Zookeeper-a. Mogucnost linkovanja moze se koristiti za kreiranje klastera. 

10.) Poslednja linija samo govori koji docker image pokrecemo: privatni repozitorijum koji se nalazi na serveru, na portu 4444 pod tag imenom "kafka". 
