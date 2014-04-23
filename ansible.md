# VirtualBox

[download](https://www.virtualbox.org/wiki/Linux_Downloads)

``sudo dpkg -i virtualbox-4.3_4.3.10-93012~Ubuntu~raring_amd64.deb`` 

Oracle VM VirtualBox se instalira na host operativnom sistemu kao aplikacja. Svaka virtuelna masina (Guest) moze biti ucitana i pokrenuta sa svojim virtuelnim okruzenje. Koristicemo VirtualBox u ovom tutorijalu za simuliranje rada servera. 

# Vagrant

Vagrant obezbjedjuje prenosivo radno okruzenje koje se lako konfigurise, moze se reprodukovati, a izgradjeno je na standardnim tehnologijama i kontrolisano od strane jednog konzistentnog toka u cilju maksimizovanja produktivnosti i fleksibilnosti tima. Vagrant se izvrsava na VMware, AWS, VirtualBox i drugim virtuelnim masinama. 

[Dokumantacija](http://docs.vagrantup.com/v2/)

[Download](https://www.vagrantup.com/download-archive/v1.5.1.html) -download (kod mene radi bas ova verzija (v1.5.1), ne najnovija v1.5.2) 

### Instalacija:
``sudo dpkg -i vagrant_1.5.1_x86_64.deb``

``vagrant init``  - kreira Vagrantfile

	VAGRANTFILE_API_VERSION = "2"

	Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
	  
	  config.vm.define :web do |web|
	    web.vm.box = "precise32"
	    web.vm.box_url = "http://files.vagrantup.com/precise32.box"
	    web.vm.network :private_network, ip: "10.33.33.33"
	    web.vm.network :forwarded_port, guest: 80, host: 8080

	    web.vm.hostname = "dev.416.bike"

	    web.vm.provider :virtualbox do |vb|
	      vb.customize ["modifyvm", :id, "--memory", "1024"]
	    end

	    web.vm.provision :ansible do |ansible|
	      ansible.playbook = "build-server.yml"
	      ansible.inventory_path = "hosts-vagrant"
	      ansible.verbose = "vvvv"
	      ansible.ask_sudo_pass = true

	      # https://github.com/mitchellh/vagrant/issues/3096
	      ansible.limit = 'all'
	    end
	  end
	end


``vagrant up`` - skidanje (ucitavanje) virtuelne masine koja je navedena u Vagrantfile-u

``vagrant provision``

``vagrant halt`` 

``vagrant destroy``


# Ansible

[Documentacija](http://docs.ansible.com/)

Ansible je IT alatka za automatizaciju. Moze da konfigurise sisteme, izvrsiti "deployment" i upravlajti slozenijim IT zadacima. 

[Instalacija](http://docs.ansible.com/intro_installation.html#latest-releases-via-apt-ubuntu)

	$ sudo apt-add-repository ppa:rquillo/ansible
	$ sudo apt-get update
	$ sudo apt-get install ansible

## Uvod

Ansible ima tri osnovne cjeline:

- vars
- roles
- inventory files

### vars

Vars je folder koji sadrzi promjenljive. Jedan ovakav fajl je:

vars/defaults.yml	

	---
	# Unix user name to deploy as
	deploy: deployer
	ruby_version: 2.1.1
	rbenv_makeopts: ""

U playbook-u u navodimo da koristimo bas promjenljive iz ovog fajla na sledeci nacin:

	 vars_files:
   	- vars/defaults.yml

Upotrebljavamo promjenljivu ``{{deploy}}``.

Promjenljive mozemo definisati i na raznim drugim mjestima, i njihov prioritet opada na sledeci nacin:

- -e promjenljive

		roles:
   		- { name: apache, http_port: 8080 }

- promjenljive definisane u inventory-ju
- cinjenice otkrivene o sistemu
- role defaults

Otprilike, sto je dalje definisana promjeljiva, manji joj je prioritet. 


### inventory files

Inventory file je obican ini fajl. U njemu navodimo hostove koje koristimo. Mozemo ih navesti u grupama: 

	[group1]
	www.host1
	host2
	[group2]
	33.33.33.33
	variable1=127.23.22.22 #defining a variable 

Jedan host moze da se pojavi u vise grupa, mozemo da kombinujemo grupe.. 
Na kraju, ako se ``role`` izvrsava za sve hostove, navodimo:

	- hosts: all



### Playbooks

U ``roles`` folder stavljamo sve playbook-ove.

![Struktura roles foldera](/home/ivana/Downloads/roles.jpg)

- handlers sluzi za notifikacije koje se izvrsavaju najvise jednom i to na kraju play-a
- file za ``file`` dio
- template se navodi u ``template`` dijelu, kao source
- task sadrzi play-eve, i on mora sadrzati ``main.yml`` fajl

Playbooks su  Ansiblov jezik. Opisuju propise po kojima zelite da se udaljeni sistem ponasa, ili predstavljaju skup koraka generalnog IT procesa.
Ako su moduli alatke u radionici, playbooks su dizajn planovi.Koristimo ih da navedemo sta zelimo da se izvrsi. To sy .yml fajlovi. 
Sintaksa je dosta jednostavna, sto se najbolje vidi na primjeru: 


	- name: Remove /opt/vagrant_ruby
	    file: name=/opt/vagrant_ruby state=absent
	    sudo: yes

Ovaj play ce ukloniti fajl /opt/vagrant_ruby. Stavka ``name`` se ne izvrsava, samo je tu kao komentar. ``sudo`` oznacava da ce kod izvrsavanja ovog taska biti potreban pasvord. ``file`` i ``sudo`` su moduli. 

	- name: run_service_name
	    service: name=service_name state=running
			notify: run_service_name

Ovakav play koji ima ``notify`` zahtijeva da postoji i handler fajl koji ce odraditi notifikaciju pod navedenim imenom:

	- name: run_service_name
    	service: name=service_name state=running

Sledeci play ce demonstrirati koriscenje promjenljivih:

	- name: Install postgresql, libpq-dev, python-psycopg2
	  	action: apt pkg={{item}} state=latest update_cache=true  
	  	sudo: yes
	  	with_items:
		  - postgresql
		  - libpq-dev
		  - python-psycopg2

Za svaki item ce se izvsiti ova komanda. 

### 

### Ansible Modules 

Moduli su akcije koje mozemo da izvrsimo u play-u. Zvanicni moduli su pisani u Python-u. Mogu se pisati i novi. Ovo su najcesci:

- command: izvrsava komandu na udaljenom cvoru

```
	# Example from Ansible Playbooks
	- command: /sbin/shutdown -t now

	# Run the command if the specified file does not exist
	- command: /usr/bin/make_database.sh arg1 arg2 creates=/path/to/database
```
	- apt: upravlja apt paketima

```
	# Update repositories cache and install "foo" package
	- apt: pkg=foo update_cache=yes

	# Remove "foo" package
	- apt: pkg=foo state=absent

	# Install the package "foo"
	- apt: pkg=foo state=present

	# Install the version '1.00' of package "foo"
	- apt: pkg=foo=1.00 state=present

	# Update the repository cache and update package "nginx" to latest version using default release squeeze-backport
	- apt: pkg=nginx state=latest default_release=squeeze-backports update_cache=yes

	# Install latest version of "openjdk-6-jdk" ignoring "install-recommends"
	- apt: pkg=openjdk-6-jdk state=latest install_recommends=no

	# Update all packages to the latest version
	- apt: upgrade=dist

	# Run the equivalent of "apt-get update" as a separate step
	- apt: update_cache=yes
```

- service: upravlja servisima

```
	# Example action to start service httpd, if not running
	- service: name=httpd state=started

	# Example action to stop service httpd, if running
	- service: name=httpd state=stopped

	# Example action to restart service httpd, in all cases
	- service: name=httpd state=restarted

	# Example action to reload service httpd, in all cases
	- service: name=httpd state=reloaded

	# Example action to enable service httpd, and not touch the running state
	- service: name=httpd enabled=yes
```

- copy: kopira fajlove na udaljene lokacije

	```copy: src=/srv/myfiles/foo.conf dest=/etc/foo.conf owner=foo group=foo```

- file: podesava atribute fajla

	```file: path=/etc/foo.conf owner=foo group=foo mode=0644```

	```file: src=/file/to/link/to dest=/path/to/symlink owner=foo group=foo state=link```

[Spisak modula](http://docs.ansible.com/list_of_all_modules.html)

## Pripremanje servera

### Izmjene koje treba napraviti:

``ansible-ubuntu-rails-server/vars/sampleapp.yml`` - preimenovati u ``application_name.yml``
(ovo realno i nije neophodno, samo je pregednije)

```
	server_name: dev.416.bike
	full_app_name: application_name_production

	# Postgresql database settings

	database_host: localhost
	database_port: 5432
	database_name: application_name_production

	database_user: application_name
	database_password: "{{ lookup('password', inventory_dir + '/credentials/' + database_user + '.postgresql.txt length=20') }}"
```

vars/defaults.yml:

```
	# Unix user name to deploy as
	deploy: deployer

	# Crypted password for 'deploy' Unix user.
	# To generate, use generate-crypted-password.py
	password: generisani password 
	# Ruby version to install
	ruby_version: 2.1.0
	rbenv_makeopts: ""

	# Papertrail logging
	# e.g. @logs.papertrailapp.com:1234
	papertrail_log_dest: "@logs.papertrailapp.com:46606"
```

hosts-digitalocean:

```
	[webservers]
	greenfield.416.bike ansible_ssh_host=IP_on_the_server
```

build-server.yml:
```
    - hosts: all
 vars_files:
    - vars/defaults.yml
    - vars/application_name.yml
      remote_user: "{{ deploy }}"
```

build-server.yml:

```
   ### Site specific roles:
        - { role: prepare_site_pricemeter, tags: ['prepare_site_pricemeter'] }
```

folder  ``roles/prepare_site_sampleapp`` preimenovati u ``prepare_site_application_name``
fajl ``roles/prepare_site_application_name/templates/sampleapp_production.nginx.j2`` preimenovati u ``appication_name.nginx.j2``


	$ git clone https://github.com/ivanacorovic/ansible_capistrano_server.git
	$ cd ansible-ubuntu-rails-server


sudo_password: (to je pasvord korisnika deploy). On, u kodu, mora biti heshiran, tj, generisemo ga tako sto se pozicioniramo u support folder i odradimo:

```
$ mkpasswd --method=SHA-512

```

To sto dobijemo stavimo kao pasvord deploy korisnika. U toku izvrsavanja, ova f-ja trazi da unesemo pasvord u ``vars/defaults.yml``, i tu rijec pisemo kad treba da se unese sudo_password. 

	$ vagrant up # ako vrati gresku, ne brinite
	$ vagrant provision
	(and now we wait...)

## Pripremanje aplikacije

Udjite u folder Vase aplikacije. 
Klonirajte projekat koji sluzi kao primjer i izvrsite potrebne izmjene. 

	$ cd ..
	$ git clone https://github.com/ivanacoroviserver_pricemeter.git

Postarajte se da izmijenite cijeli config/deploy folder tako da ima sadrzaj kao u klonovanom projektu. 

server je 'dev.416.bike' ako vrsimo deploymentu na VMBox. Ako radimo sa serverom, onda tu staviti 'greenfield.416.bike'.
production.rb:

```
	server 'dev.416.bike', user: 'deployer', roles: %w{web app db}
	.
	.
	.
	set :deploy_to, "/home/#{fetch(:deploy_user)}/apps/#{fetch(:full_app_name)}"
```

fajl deploy.rb (izmijenite ovaj dio tako da su to podaci koji se odnose na Vasu aplikaciju):

```
	set :application, "your_app_name"
	set :deploy_user, "deployer"
	set :scm, "git"
	set :repo_url, "git address of the app you're now changing"
	set :rbenv_type, :user
	set :rbenv_ruby, '2.1.0'
  . 
  .
  .
  set :tests, ["spec"]

```
Postarajte se da je lib/capistrano folder i sve u njemu bude isto u Vasoj i aplikaciji sa primjera.

Capfile takodje treba da bude kao u primjeru. 
	
	$ cd your_app
	$ rvm gemset use new_gemset --create
	$ bundle install
	$ cp config/database.yml config/database.yml.example
	$ git add -u
	$ git commit -m "deployment commit"
	$ git push origin master
	$ bundle exec rake db:test:prepare
	$ cap production deploy




Q&A

What is inventory file? syntax and options for inventory files? what is default inventory file?


What is playbook? syntax and options for playbooks?


What are modules? What modules provide? Which are some common or often required modules?



How ansible communicates with remote machines?


What is the user which ansible uses by default to connect remote ssh? How to tell ansible which user to connect to remote machine?


What is the difference when you define sudo: yes for play and sudo: yes for particular task?

How many times handlers are invoked when they are notified multiple times?



Explain role directory organization structure?


Why following does not work?

```
- hosts: app_servers
  vars:
      app_path: {{ base_path }}/22
```


How will you retrieve remote host eth0 macaddress in some playbook template file?


How will you dynamically provide frontend proxy server with IP addresses of all of the app servers for which it proxies the traffic? (see variables - groups)

How will you specify that particular task needs to be performed only if previous task was successfully performed? (see conditionals)
