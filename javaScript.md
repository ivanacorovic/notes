# Uvod u Ajax

Kada otvaramo neku stranicu u pretrazivacu, on salje zahtjev na server, obradjuje odgovor, pokupi sve assete (JavaScript fajlove, slike, stylesheet-ove) i naslozi stranu. Kad kliknemo na neki link na toj stranici, ponovo se desava ista stvar: request response cycle.

JavaScript takodje moze da radi istu stvar, ali, pored toga, ima sposobnost da update-uje informacije na stranici. Time izbjegava potrebu da dohvati cijelu stranicu sa servera. Ova tehniku zovemo Ajax. 

Rails radi sa CoffeeScript-om, tako da ce primjeri biti takvi. 

```
$.ajax(url: "/test").done (html) ->
$("#results").append html
```

Ovaj kod dohvata podatke sa "/test"-a i nadovezuje ih na div koji ima id "results".

Rails ima ugradjenu podrsku za ovu tehniku, pa ce pisanje ovog koda da se izbjegne. 

# Povuceni JavaScript

Rails koristi ovu tehniku i smatra se da he to best-practice.

Jednostavan nacin za pisanje JavaScripta:

```
<a href="#" onclick="this.style.backgroundColor='#990000'">Paint it red</a>
```

To je "inline" JavaScript. I to je ok, ako necemo da se mnogo stvari desi na klik, jer bi to pretrpalo kod:

```
<a href="#" onclick="this.style.backgroundColor='#009900';this.style.color='#FFFFFF';">Paint it green</a>
```

Mozemo da napisemo funkciju i pretvorimo je u CoffeeScript:

```
paintIt = (element, backgroundColor, textColor) ->
  element.style.backgroundColor = backgroundColor
  if textColor?
    element.style.color = textColor
```

A na stranici da bude:

```
<a href="#" onclick="paintIt(this, '#990000')">Paint it red</a>
```

To je malo bolje, ali, opet, ako imamo mnogo linkova sa istim efektom, i nije bas najsrecnije resenje. 

HTML5 ima data atribut koji se moze dodijeliti svakom elementu. To cemo iskoristiti:

```
paintIt = (element, backgroundColor, textColor) ->
  element.style.backgroundColor = backgroundColor
  if textColor?
    element.style.color = textColor
 
$ ->
  $("a[data-background-color]").click ->
    backgroundColor = $(this).data("background-color")
    textColor = $(this).data("text-color")
    paintIt(this, backgroundColor, textColor)
```

Na stranici ce biti ovakav kod:

```
<a href="#" data-background-color="#990000">Paint it red</a>
<a href="#" data-background-color="#009900" data-text-color="#FFFFFF">Paint it green</a>
<a href="#" data-background-color="#000099" data-text-color="#FFFFFF">Paint it blue</a>
```

Ovo je "pritajeni" (unobstructive) JavaScript jer se vise ne mijesa u html kod.


# Ugradjeni helperi

Rails "Ajax  helpers" su ustvari u dva dijela: JavaScript polovina i Ruby polovina.

[rails.js](https://github.com/rails/jquery-ujs/blob/master/src/rails.js) obezbjedjuje JavaScript polovinu, a Ruby view helperi dodaju odgovarajuce tagove DOM-u (Document Objec Model). CoffeeScript u rails.js-u ceka te atribute i pokrece odgovarajuce handler-e.

## form_for

```
<%= form_for(@post, remote: true) do |f| %>
  ...
<% end %>
```

Generise:

```
<form accept-charset="UTF-8" action="/posts" class="new_post" data-remote="true" id="new_post" method="post">
  ...
</form>
```

Akcenat je na ``remote="true"``. To znaci da ce submit obraditi Ajax, a ne uobicajeni mehanizam pretrazivaca. 

Ukoliko zelimo da ispisemo nesto nakon submit-ovanja:

```
$(document).ready ->
  $("#new_post").on("ajax:success", (e, data, status, xhr) ->
    $("#new_post").append xhr.responseText
  ).on "ajax:error", (e, xhr, status, error) ->
    $("#new_post").append "<p>ERROR</p>"
``` 
## form_tag

```
<%= form_tag('/posts', remote: true) do %>
  ...
<% end %>
```

Generise:

```
<form accept-charset="UTF-8" action="/posts" data-remote="true" method="post">
  ...
</form>
```

Ostalo je kao za form_for.


## link_to

```
<%= link_to "a post", @post, remote: true %>
```

Generise:

```
<a href="/posts/1" data-remote="true">a post</a>
```

Evo primjera na kom se vidi kako da se poveze sa Ajax-om brisanje posta:

```
<%= link_to "Delete post", @post, remote: true, method: :delete %>
```

```
$ ->
  $("a[data-remote]").on "ajax:success", (e, data, status, xhr) ->
    alert "The post was deleted."
```

##  button_to

```
<%= button_to "A post", @post, remote: true %>
```

Generise:

```
<form action="/posts/1" class="button_to" data-remote="true" method="post">
  <div><input type="submit" value="A post"></div>
</form>
```

Posto je to samo forma, vazi isto sto i za form_for.

# Server-Side 

Da bi Ajax radio kako treba, ponesto mora da se odradi i na strani servera. Da li ce Ajax da vrati JSON ili HTML, vidjecemo kako se odredjuje.

## Jednostavan primjer

Na stranici prikazujemo niz korisnika, a imamo i formu za kreiranje novog korisnika. Index akcija ce izgledati ovako:

```
class UsersController < ApplicationController
  def index
    @users = User.all
    @user = User.new
  end
  # ...
```

app/views/users/index.html.erb:

```
<b>Users</b>
 
<ul id="users">
<%= render @users %>
</ul>
 
<br>
 
<%= form_for(@user, remote: true) do |f| %>
  <%= f.label :name %><br>
  <%= f.text_field :name %>
  <%= f.submit %>
<% end %>
```

 app/views/users/_user.html.erb:

 ```
 <li><%= user.name %></li>
 ```

```
#app/controllers/users_controller.rb
def create
  @user = User.new(params[:user])
 
  respond_to do |format|
    if @user.save
      format.html { redirect_to @user, notice: 'User was successfully created.' }
      format.js   {}
      format.json { render json: @user, status: :created, location: @user }
    else
      format.html { render action: "new" }
      format.json { render json: @user.errors, status: :unprocessable_entity }
    end
  end
end
```

Blok "respod_to" omogucava da kontroleru da odgovori na Ajax zahtjev.
Sada imamo odgovarajuci app/views/users/create.js.erb fajl:

```
$("<%= escape_javascript(render @user) %>").appendTo("#users");
```

# Turbolinks

Rails 4 koristi Turbolinks gem. Ovaj gem koristi Ajax da ubrza prikazivanje stranica u vecini aplikacija.

## Kako Turbolinks radi

On nakaci klik handler na sve ``<a>``  elemente na strani. Ako pretrazivac podrzava metod [pushState()](https://developer.mozilla.org/en-US/docs/Web/Guide/API/DOM/Manipulating_the_browser_history#The_pushState%28%29.C2.A0method), Tubolinks uputi Ajax zahtjev za stranu, obradi odgovor, i zamijeni citavu ``<body>`` sekciju tijelom odgovora. Tada ce iskoristiti PushState da promijeni URL na tacnu adresu.

Treba dodati gem turbolinks u gemfajlu, i staviti ``//=require turbolinks`` u ``app/assets/javascripts/application.js``.

Ako zelimo da iskljucimo Turbolinks na nekim linkovima, mozemo to uraditi ovako:

```
<a href="..." data-no-turbolink>No turbolinks here</a>.
```

## Page Change Events

Turbolinks override-uje normalno ucitavanje stranice, pa funkcije ``$(docunment).ready`` nece raditi. Zamijenite ih ovim:

```
$(document).on "page:change", ->
  alert "page has loaded!"
```