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




