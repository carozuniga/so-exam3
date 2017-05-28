Universidad ICESI
## Informe Examen 3
**Nombre:**  Carolina Zúñiga

**Código:**  A00315292

**Url Repositorio**: https://github.com/carozuniga/so-exam3/tree/master/A00315292

### Solución a las preguntas

3. Para la instalación y configuración del stack de sensu (cliente, servidor, api, uchiwa dashboard y rabbitmq como bus de mensajería) se siguieron los pasos del repositorio https://github.com/ICESI/so-monitoring vistos en clase, los archivos de configuración usados tambien se encuentran en el repositorio mencionado. En mi caso por cuestiones de comodidad instalé el cliente y el servidor en la misma maquina. los pasos a seguir fueron los siguientes:

```
echo '[sensu]
name=sensu
baseurl=https://sensu.global.ssl.fastly.net/yum/$releasever/$basearch/
gpgcheck=0
enabled=1' | sudo tee /etc/yum.repos.d/sensu.repo

yum install sensu -y
sensu-install -p sensu-plugin
sensu-install -p sensu-plugins-slack
su -c 'rpm -Uvh http://download.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-9.noarch.rpm'
yum install erlang -y
yum install redis -y
service redis start
yum install socat -y
su -c 'rpm -Uvh http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.9/rabbitmq-server-3.6.9-1.el7.noarch.rpm'
service rabbitmq-server start
rabbitmqctl add_vhost /sensu
rabbitmqctl add_user sensu password
rabbitmqctl set_permissions -p /sensu sensu ".*" ".*" ".*"
rabbitmq-plugins enable rabbitmq_management
chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
yum install uchiwa -y
firewall-cmd --zone=public --add-port=5672/tcp --permanent
firewall-cmd --zone=public --add-port=15672/tcp --permanent
firewall-cmd --zone=public --add-port=3000/tcp --permanent
firewall-cmd --reload
gem install sensu-plugin
service sensu-server restart & service sensu-api restart & service uchiwa restart & service sensu-client start
```
Antes de realizar estos pasos se debe verificar que tanto ruby como las gemas de ruby se encuentren instaladas en la maquina.

4. Para este punto se creó un check de sensu del servicio de apache, el archivo de configuración del check (/etc/sensu/conf.d/check_apache.json) es el siguiente:

```
{
  "checks": {
    "apache_check": {
      "command": "/opt/sensu/embedded/bin/ruby /etc/sensu/plugins/check-apache.rb",
      "interval": 15,
      "handlers": ["slack"],
      "subscribers": ["webservers"]
    }
  }
}
```
El API para este check (/ect/sensu/plugins/check-apache.rb) es el siguiente:

```
#!/usr/bin/env ruby

procs = `ps aux`
running = false
procs.each_line do |proc|
  running = true if proc.include?('httpd')
end
if running
  puts 'OK - Apache daemon is running'
  exit 0
else
  puts 'WARNING - Apache daemon is NOT running'
  exit 1
end
```
Capturas de pantalla del dashboard de uchiwa:

![][1]

Captura de la caida del servicio, despues de ejecutar el comando service httpd stop se observa la alerta y el historial pasa de tener ceros a unos:

![][2]

5. El plugin de sensu instalado verifica la memoria RAM del equipo, el archivo de configuración del plugin (/etc/sensu/conf.d/check_ram.rb) es el siguiente:

```
{
  "checks": {
    "ram_usage": {
      "command": "/opt/sensu/embedded/bin/ruby /etc/sensu/plugins/check-ram.rb",
      "interval": 15,
      "handlers": ["slack"],
      "subscribers": [ "webservers" ]
    }
  }
}

```
Este check tiene las siguientes salidas:

- 0       CheckRAM OK: 65% free RAM left
- 1       CheckRAM WARNING: 40% free RAM left
- 2       CheckRAM CRITICAL: 13% free RAM left
- 3       CheckRAM UNKNOWN: invalid percentage

El API para este check (/etc/sensu/plugins/check-ram.rb) es el siguiente:

```
#!/usr/bin/env ruby
#
# Check free RAM Plugin
#

require 'sensu-plugin/check/cli'

class CheckRAM < Sensu::Plugin::Check::CLI

  option :warn,
    :short => '-w WARN',
    :proc => proc {|a| a.to_i },
    :default => 10

  option :crit,
    :short => '-c CRIT',
    :proc => proc {|a| a.to_i },
    :default => 5

  def run
    total_ram, free_ram = 0, 0

    `free -m`.split("\n").drop(1).each do |line|
      free_ram = line.split[3].to_i if line =~ /^-\/\+ buffers\/cache:/
      total_ram = line.split[1].to_i if line =~ /^Mem:/
    end

    unknown "invalid percentage" if config[:crit] > 100 or config[:warn] > 100

    percents_left = free_ram*100/total_ram
    message "#{percents_left}% free RAM left"

    critical if percents_left < config[:crit]
    warning if percents_left < config[:warn]
    ok
  end
end
```
Capturas de pantalla del check de ram (RAM_USAGE) en el dashboard de uchiwa:

![][3]

6. El handler de slack se configuró para los dos checks vistos anteriormente, el codigo de configuración del handler de slack (/etc/sensu/conf.d/handler_slack.json) es el siguiente:

```
{
    "handlers": {
        "slack": {
            "type": "pipe",
            "command": "/opt/sensu/embedded/bin/ruby /opt/sensu/embedded/lib/ruby/gems/2.4.0/gems/sensu-plugins-slack-1.0.0/bin/handler-slack.rb -j slack-integration",
            "severites": ["warning", "unknown"]
        }
    },
    "slack-integration": {
        "webhook_url": "https://hooks.slack.com/services/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",
        "template" : ""
    }
}

```
Para obtener la url webhook cree un canal propio en slack y una app. En el API de la misma se debe activar y generar los "incoming webhooks" como se muestra a continuación:

![][4]

Captura de pantalla de las notificaciones del check de ram:

![][5]

7. Para la instalación y configuración del stack de ELK se siguieron las instrucciones de https://gist.github.com/d4n13lbc/be1ad5039dff1c058b335482488d4965 que fueron vistas en clase, se siguieron las instrucciones tanto para el servidor como para el cliente.

Capturas verificando el estado del servicio:

![][6]

![][7]

![][8]

8. Se implementó el ejemplo https://github.com/elastic/examples/tree/master/ElasticStack_twitter, para este ejemplo se creo el siguiente archivo de configuración 

```
input {
  twitter {
    consumer_key       => "3NvlwpBFSjC2e5RzqW89pp0Sp"
    consumer_secret    => "d9gC2rOPMQNiVZbWxolkZ7qg6yc97wPg1I1FAqmt3PodsXKWyX"
    oauth_token        => "125781364-xyaQNE5ll6eK5Xre4ixrJtCrWBwCBYkOQqKbfc4O"
    oauth_token_secret => "OADnesSPd3tyPbp0nU2ue33HIXh35V8KRPBsVMPzaIxcH"
    keywords           => [ "thor", "spiderman", "wolverine", "ironman", "hulk"]
    full_tweet         => true
  }
}

filter { }

output {
  stdout {
    codec => dots
  }
  elasticsearch {
      hosts => "192.168.192.130:9200"
      index         => "twitter_elastic_example"
      document_type => "tweets"
      template      => "./twitter_template.json"
      template_name => "twitter_elastic_example"
      template_overwrite => true
  }
}

```
Para este archivo fue necesario crear una app de twitter para poder obtener los datos del input, los demás archivos de configuración se dejaron igual a los que se encuentran en el repositorio. Captura de la app de twitter: 

![][10]

Al correr el archivo de configuración podemos acceder al dashboard de kibana y observar los tweets de acuerdo con las keywords que se establecieron: 

![][9]


### Referencias

- http://www.programblings.com/2013/11/06/creating-a-sensu-check/
- https://github.com/elastic/examples/tree/master/ElasticStack_twitter


[1]: Images/uchiwa1.PNG
[2]: Images/uchiwa2.PNG
[3]: Images/uchiwa3.PNG
[4]: Images/hook.PNG
[5]: Images/slack.PNG
[6]: Images/status.PNG
[7]: Images/elastic.PNG
[8]: Images/kibana.PNG
[9]: Images/twitter.PNG
[10]: Images/twitterapp.PNG
