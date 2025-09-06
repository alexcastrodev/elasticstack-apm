# APM Elastic Stack


## Install in VM (Ubuntu 24.04.3 LTS)

```
sudo su
```

1. Update and install prerequisites

```bash
apt update && apt-get install apt-transport-https wget curl gpg ca-certificates -y
```

2. Download and install the public signing key:
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
```

3. Save the repository definition to `/etc/apt/sources.list.d/elastic-8.x.list`:
```bash
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-8.x.list
```

4. Update and install the Elastic Stack components:

```bash
apt update && apt install elasticsearch kibana apm-server -y
```

5. It will show the password for `elastic` user, save it.

i.e
```bash
Setting up elasticsearch (8.19.3) ...
--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is: ShBVOqbb_tftlmGy=ZKY

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

Reset the password of the elastic built-in superuser with 
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with 
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with 
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

-------------------------------------------------------------------------------------------------
```

6. Update the `elasticsearch.yml` file to set some parameters:

We will set the cluster.name, node name, network host and discovery type.

```bash
vim /etc/elasticsearch/elasticsearch.yml
```

```
cluster.name: elastic-vm
http.port: 9200
```

7. Set (or limit) the JVM heap size in `/etc/elasticsearch/jvm.options` file (You can set half of your RAM, here I have 8GB RAM so I set 4GB):

```bash
-Xms4g
-Xmx4g
```

8. Enable and start the elasticsearch service:

```bash
systemctl enable elasticsearch
systemctl start elasticsearch
systemctl status elasticsearch
```

You gonna see something like this:

```bash
root@elastic:/home/ubuntu# systemctl start elasticsearch
root@elastic:/home/ubuntu# systemctl status elasticsearch
● elasticsearch.service - Elasticsearch
     Loaded: loaded (/usr/lib/systemd/system/elasticsearch.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-09-06 10:41:17 UTC; 3s ago
       Docs: https://www.elastic.co
   Main PID: 12858 (java)
      Tasks: 105 (limit: 4599)
     Memory: 2.5G (peak: 2.5G)
        CPU: 17.318s
     CGroup: /system.slice/elasticsearch.service
             ├─12858 /usr/share/elasticsearch/jdk/bin/java -Xms4m -Xmx64m -XX:+UseSerialGC -Dcli.name=server -Dcli.script=/usr/share/elasticsearch/bin/elasticsearch -Dcli.libs=lib/tools/server-cli -Des.path.home=/usr/share/e>
             ├─12921 /usr/share/elasticsearch/jdk/bin/java -Des.networkaddress.cache.ttl=60 -Des.networkaddress.cache.negative.ttl=10 -XX:+AlwaysPreTouch -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true>
             └─12943 /usr/share/elasticsearch/modules/x-pack-ml/platform/linux-aarch64/bin/controller

Sep 06 10:41:02 elastic systemd[1]: Starting elasticsearch.service - Elasticsearch...
Sep 06 10:41:17 elastic systemd[1]: Started elasticsearch.service - Elasticsearch.
```

9. Create credentials for Kibana to connect to Elasticsearch:

```bash
root@elastic-n2:/home/ubuntu# curl -u elastic:'ShBVOqbb_tftlmGy=ZKY' --cacert /etc/elasticsearch/certs/http_ca.crt -X POST "https://localhost:9200/_security/service/elastic/kibana/credential/token/kibana-token"
{"created":true,"token":{"name":"kibana-token","value":"AAEAAWVsYXN0aWMva2liYW5hL2tpYmFuYS10b2tlbjpUR1ZrMHVPeFEyZXZnTVZCUDN3TXRR"}
```

10. Copy the certificates from elasticsearch to kibana:

```bash
sudo mkdir -p /etc/kibana/certs
scp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/certs/http_ca.crt
sudo chown root:kibana /etc/kibana/certs/http_ca.crt
sudo chmod 640 /etc/kibana/certs/http_ca.crt
```

11. Update the `kibana.yml` file to set some parameters:


We will set the server.host and elasticsearch.hosts.

```bash
vim /etc/kibana/kibana.yml
```

```bash
server.port: 5601
server.host: "192.168.x.x"  # to be reached from others
elasticsearch.hosts: ["https://192.168.x.x:9200"]
server.publicBaseUrl: http://192.168.x.x:5601
elasticsearch.serviceAccountToken: "AAEAAWVsYXN0aWMva2liYW5hL2tpYmFuYS10b2tlbjpUR1ZrMHVPeFEyZXZnTVZCUDN3TXRR"
elasticsearch.ssl.verificationMode: certificate
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/http_ca.crt"]
```

12. Enable and start the kibana service:

```bash
systemctl enable kibana
systemctl start kibana
systemctl status kibana
```

You gonna see something like this:

```bash
root@elastic:/home/ubuntu# systemctl enable kibana
Created symlink /etc/systemd/system/multi-user.target.wants/kibana.service → /usr/lib/systemd/system/kibana.service.
root@elastic:/home/ubuntu# systemctl start kibana
root@elastic:/home/ubuntu# systemctl status kibana
● kibana.service - Kibana
     Loaded: loaded (/usr/lib/systemd/system/kibana.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-09-06 10:44:03 UTC; 2s ago
       Docs: https://www.elastic.co
   Main PID: 16581 (node)
      Tasks: 11 (limit: 4599)
     Memory: 363.3M (peak: 363.7M)
        CPU: 2.364s
     CGroup: /system.slice/kibana.service
             └─16581 /usr/share/kibana/bin/../node/glibc-217/bin/node /usr/share/kibana/bin/../src/cli/dist

Sep 06 10:44:03 elastic systemd[1]: Started kibana.service - Kibana.
Sep 06 10:44:03 elastic kibana[16581]: Kibana is currently running with legacy OpenSSL providers enabled! For details and instructions on how to disable see https://www.elastic.co/guide/en/kibana/8.19/production.html#openssl>
Sep 06 10:44:03 elastic kibana[16581]: {"log.level":"info","@timestamp":"2025-09-06T10:44:03.899Z","log.logger":"elastic-apm-node","ecs.version":"8.10.0","agentVersion":"4.13.0","env":{"pid":16581,"proctitle":"/usr/share/kib>
Sep 06 10:44:03 elastic kibana[16581]: Native global console methods have been overridden in production environment.
Sep 06 10:44:04 elastic kibana[16581]: [2025-09-06T10:44:04.764+00:00][INFO ][root] Kibana is starting
Sep 06 10:44:04 elastic kibana[16581]: [2025-09-06T10:44:04.785+00:00][INFO ][node] Kibana process configured with roles: [background_tasks, ui]
````

13. Access Kibana web console: (MAYBE)

![Kibana Login Screen](./github/1.png)

It will (maybe) ask you Enrollment token. You can generate it by running the following command:

```bash
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

output:

```bash
root@elasticapm:/home/ubuntu# /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
eyJ2ZXIiOiI4LjE0LjAiLCJhZHIiOlsiMTkyLjE2OC4xLjIzOTo5MjAwIl0sImZnciI6ImYwMzJmOWIwZDM1ZWFmYzYxODQxYTU0NDdhMDlhNjZhNDY2ZDg5MzBiNzc2MjM5MTkzNTFhZDNhY2I0MzQxYmYiLCJrZXkiOiJmaXFFSDVrQlFaSnhBczhwS2RzbDpPM3NWRHE3LU5qLUxpRDNWQ2R2TGpBIn0=
```

You will see a modal: 

```
Verification required
Copy the code from the Kibana server or run bin/kibana-verification-code to retrieve it.
```

Run the following command to get the code:

```bash
root@minha-vm:/home/ubuntu# /usr/share/kibana/bin/kibana-verification-code
Your verification code is:  707 514 
```

![Kibana Login Screen](./github/2.png)

After that, configure Elastic button.

![Install](./github/3.png)

it will ask for username and password, use `elastic` and the password you saved during elasticsearch installation (step 5).

![Kibana Login Screen](./github/4.png)

Open your browser and go to `http://<your-vm-ip>:5601` (e.g., `http://192.168.x.x:5601`)

14. Copy the certificates from elasticsearch to apm-server:

```bash
root@elasticapm:/home/ubuntu# sudo mkdir -p /etc/apm-server/certs
root@elasticapm:/home/ubuntu# scp /etc/elasticsearch/certs/http_ca.crt /etc/apm-server/certs/http_ca.crt
root@elasticapm:/home/ubuntu# sudo chown root:apm-server /etc/apm-server/certs/http_ca.crt
root@elasticapm:/home/ubuntu# sudo chmod 640 /etc/apm-server/certs/http_ca.crt
```


15. Now, we will install and configure apm-server.

```bash
curl -s -u elastic:'ShBVOqbb_tftlmGy=ZKY' \
  --cacert /etc/apm-server/certs/http_ca.crt \
  https://localhost:9200/_security/api_key \
  -H 'Content-Type: application/json' -d '{
  "name": "apm-elastic-superuser",
  "role_descriptors": {
    "all_access_role": {
      "cluster": ["all"],
      "index": [
        { "names": ["*"], "privileges": ["all"] }
      ],
      "applications": [
        { "application": "*", "privileges": ["*"], "resources": ["*"] }
      ]
    }
  }
}'
{"id":"9QK0H5kB5VLQM_8jZfNO","name":"apm-elastic-superuser","api_key":"PhWlJEywiVMV-Ybp_lReLQ""encoded":"OVFLMEg1a0I1VkxRTV84alpmTk86UGhXbEpFeXdpVk1WLVlicF9sUmVMUQ=="}
```


16. Update the `apm-server.yml` file to set some parameters:

```bash
vim /etc/apm-server/apm-server.yml
```

get the id and api_key to set in the file.

9QK0H5kB5VLQM_8jZfNO:PhWlJEywiVMV-Ybp_lReLQ

use this file: [apm-server.yml](./apm-server.yml)

13. Enable and start the apm-server service:

Copy certificates from elasticsearch to apm-server:

Then

```bash
systemctl enable apm-server
systemctl start apm-server
systemctl status apm-server
```

You gonna see something like this:

```bash
root@minha-vm:/home/ubuntu# systemctl enable apm-server
Created symlink /etc/systemd/system/multi-user.target.wants/apm-server.service → /usr/lib/systemd/system/apm-server.service.
root@minha-vm:/home/ubuntu# systemctl start apm-server
root@minha-vm:/home/ubuntu# systemctl status apm-server
● apm-server.service - Elastic APM Server
     Loaded: loaded (/usr/lib/systemd/system/apm-server.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-09-06 12:33:02 WEST; 3s ago
       Docs: https://www.elastic.co/apm
   Main PID: 22344 (apm-server)
      Tasks: 6 (limit: 9421)
     Memory: 39.4M (peak: 39.7M)
        CPU: 52ms
     CGroup: /system.slice/apm-server.service
             └─22344 /usr/share/apm-server/bin/apm-server --path.home /usr/share/apm-server --path.config /etc/apm-server --path.data /var/lib/apm-server --path.logs /var/log/apm-server --environment systemd

Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.logger":"beater.handler","log.origin":{"function":"github.com/elastic/apm-server/internal/beater/api.NewMux","file.name":"api/mux.go","file.>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.logger":"beater.handler","log.origin":{"function":"github.com/elastic/apm-server/internal/beater/api.NewMux","file.name":"api/mux.go","file.>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.logger":"beater.handler","log.origin":{"function":"github.com/elastic/apm-server/internal/beater/api.NewMux","file.name":"api/mux.go","file.>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.logger":"beater.handler","log.origin":{"function":"github.com/elastic/apm-server/internal/beater/api.NewMux","file.name":"api/mux.go","file.>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.logger":"beater.handler","log.origin":{"function":"github.com/elastic/apm-server/internal/beater/api.NewMux","file.name":"api/mux.go","file.>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.origin":{"function":"github.com/elastic/apm-server/internal/beater/jaeger.RegisterGRPCServices","file.name":"jaeger/grpc.go","file.line":72}>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.logger":"beater","log.origin":{"function":"github.com/elastic/apm-server/internal/beater.server.run","file.name":"beater/server.go","file.li>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.964+0100","log.logger":"beater","log.origin":{"function":"github.com/elastic/apm-server/internal/beater.waitReady","file.name":"beater/waitready.go","file.>
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.967+0100","log.logger":"beater","log.origin":{"function":"github.com/elastic/apm-server/internal/beater.(*httpServer).start","file.name":"beater/http.go",">
Sep 06 12:33:02 minha-vm apm-server[22344]: {"log.level":"info","@timestamp":"2025-09-06T12:33:02.967+0100","log.logger":"beater","log.origin":{"function":"github.com/elastic/apm-server/internal/beater.(*httpServer).start","file.name":"beater/http.go"
```

Save it. (it will change the kibana.yml file to add the apm settings)

Now, you can use APM in you application.

Example using in Rails application:

1. Add the `elastic-apm` gem to your Gemfile:

```ruby
gem 'elastic-apm'
```

2. Run `bundle install` to install the gem.


3. Create the configuration file for Elastic APM:

```bash
/app/config/elastic_apm.yml
```

I am using the same API key created in step 11. But you should create a separated one for your application.

```yaml
server_url: http://192.168.1.240:8200
secret_token: 'OVFLMEg1a0I1VkxRTV84alpmTk86UGhXbEpFeXdpVk1WLVlicF9sUmVMUQ=='
service_name: 'application-name'
environment: 'development'
```

4. Start Rails application server

Do some navigations and use this in Rails console to see the transactions:

```ruby
ElasticAPM.start

raise "Testing APM"

ElasticAPM.stop
```

It will show in Observability > Applications:

![Kibana Login Screen](./github/6.png)
