#Logstasher

##Podesavanje aplikacije

Dodajte u:

Gemfile:

```
  gem 'logstasher'
```

development.rb:

```
  # Enable the logstasher logs for the current environment
  config.logstasher.enabled = true

  # This line is optional if you do not want to suppress app logs in your <environment>.log
  config.logstasher.suppress_app_log = false
 
  # Enable logging of controller params
  config.logstasher.log_controller_parameters = true
```


##Instalacija i podesavanje:

```
  curl -O https://download.elasticsearch.org/logstash/logstash/logstash-1.4.0.tar.gz

  tar zxvf logstash-1.4.0.tar.gz
  cd logstash-1.4.0

  nano quickstart.conf
```
quickstart.conf:

```
  input {
    file {
      type => "rails logs"
      path => "YOUR_APP_PATH/log/logstash_development.log"
      codec => json {
        charset => "UTF-8"
      }
    }
  }

  output {
    # Print each event to stdout.
    stdout {
      codec => rubydebug
    }
   
    elasticsearch {
      host => localhost
    }
  }
```

## Pokretanje: 

```
  # pokrenuti iz vase aplikacije
  rails s

  sudo service elastcsearch start

  bin/logstash -f quickstart.conf
```
Idite na: 

`` localhost``

Kliknite na : 

``Logstash Dashboard``

