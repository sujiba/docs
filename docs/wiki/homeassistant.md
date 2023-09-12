---
title: Home Assistant
---

# Home Assistant
## Einführung
----------------
Home Assistant ist eine kostenlose und quelloffene Software zur Hausautomation, die als zentrales Steuerungssystem in einem Smart Home oder Smart House konzipiert ist. Geschrieben in Python liegt ihr Hauptaugenmerk auf lokaler Steuerung und Privatsphäre. 

## Vorbereitung
----------------
Es wird empfohlen, anstelle der SD-Karte eine SSD einzusetzen, da Home Assistant sehr schreibintensiv ist und SD-Karten durch viele Schreibvorgänge frühzeitig einen Defekt erleiden können. Damit der Raspberry Pi beim Systemstart von der SSD bootet, muss die Boot-Reihenfolge angepasst werden.

### Boot from USB
Raspberry Pi Imager öffnen. Unter "OS Wählen/Misc utility images/Bootloader/USB Boot" auswählen. Dann unter "SD-Karte wählen" die SD-Karte auswählen und mit "Schreiben" den Vorgang starten. Sobald die SD-Karte beschrieben und ausgeworfen wurde, kann diese in den Raspberry Pi gesteckt werden. Den Raspberry Pi an den Strom anschließen. Eine Minute warten, den Raspberry Pi wieder vom Strom trennen und die SD-Karte entfernen.

### HassOS auf SSD laden
Als nächstes können wir die SSD vorbereiten. Diese über SATA zu USB Adapter am Rechner anschließen. Erneut den Raspberry Pi Imager öffnen. In diesem Fall wählen wir unter "OS wählen/Other specific-purpose OS/Home assistants and home automation/Home Assistant/Home Assistant OS 9.3 (Pi Version)" aus. Unter "SD-Karte wählen" die SSD auswählen und auf "Schreiben" klicken. Nach Abschluss des Schreibvorgangs kann die SSD vom Rechenr getrennt und an den Raspberry Pi angeschlossen werden. Abschließen den Raspberry Pi wieder mit Strom versorgen.

## Einrichtung
--------------
Home Assistant ist jetzt unter [http://homeassistant.local:8123](http://homeassistant.local:8123) erreichbar und kann eingerichtet werden. Siehe hierfür [Onboarding](https://www.home-assistant.io/getting-started/onboarding/).

## Add-Ons
-----------
Unter "Einstellungen/Add-Ons" kann man über den "Add-On Store" (Schalter unten rechts) Erweiterungen hinzufügen. Diese Add-Ons sind weitere Docker Container, die über Home Assistant konfiguriert und gestartet werden. Die Konfigurationen können auch in YAML bearbeitet werden. Hierfür bei dem jeweiligen Add-On unter dem Reiter "Konfiguration" auf das drei Punkte Menü klicken und "Als YAML bearbeiten" auswählen.

### Let's Encrypt
[Dokumentation](https://github.com/home-assistant/addons/blob/master/letsencrypt/DOCS.md)
</br >

Folgend ein Beispiel zur Erstellung eines Let's Encrypt Zertifkats mit DNS-01 und der Netcup API:
<details>
<summary>Konfiguration</summary>
```yaml
domains:
  - home.example.com
  - second.example.com
email: MAIL
keyfile: privkey.pem
certfile: fullchain.pem
challenge: dns
dns:
  provider: dns-netcup
  propagation_seconds: 900
  netcup_customer_id: "123456"
  netcup_api_key: KEY
  netcup_api_password: PASSWORD
```
</details>

Das Zertifikat sowie der Key werden unter /ssl abgelegt und können so auch von anderen Add-Ons genutzt werde.

### NGINX Home Assistant SSL proxy
[Dokumentation](https://github.com/home-assistant/addons/blob/master/nginx_proxy/DOCS.md)
</br>

Folgend ein Beispiel zur Einrichtung des Reverse Proxys:

<details>
<summary>Konfiguration</summary>
```yaml
domain: home.example.com
hsts: max-age=31536000; includeSubDomains
certfile: fullchain.pem
keyfile: privkey.pem
cloudflare: false
customize:
  active: true
  default: nginx_proxy_default*.conf
  servers: nginx_proxy/*.conf
```
</details>

Durch das Setzen der Option "active: true" haben wir jetzt die Möglichkeit weitere vHosts für andere Add-Ons (zB Uptime Kuma) unter "nginx_proxy/*.conf" anzulegen. Der Ordner "nginx_proxy" muss unter "/share" angelegt werden. Dies kann mit dem Add-On "Terminal & SSH" gemacht werden.

### SSH & Terminal
[Dokumentation](https://github.com/home-assistant/addons/blob/master/ssh/DOCS.md)

### Mosquitto broker
[Dokumentation](https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md)

### Zigbee2MQTT
[Dokumentation](https://github.com/zigbee2mqtt/hassio-zigbee2mqtt/blob/master/README.md)
</br>

Folgend ein Beispiel zur Einrichtung von Zigbee2MQTT mit dem ConBee II:
<details>
<summary>Konfiguration</summary>
```yaml
data_path: /config/zigbee2mqtt
socat:
  enabled: false
  master: pty,raw,echo=0,link=/tmp/ttyZ2M,mode=777
  slave: tcp-listen:8485,keepalive,nodelay,reuseaddr,keepidle=1,keepintvl=1,keepcnt=5
  options: "-d -d"
  log: false
mqtt: {}
serial:
  adapter: deconz
  port: /dev/ttyACM0
```
</details>

### Uptime Kuma
[Dokumentation](https://github.com/hassio-addons/addon-uptime-kuma/blob/main/uptime-kuma/DOCS.md)
</br>

Damit Uptime Kuma auch über https erreichbar ist, muss jetzt unter "/share/nginx_proxy" eine neue vHost Konfiguration angelegt werden:
<details>
<summary>uptime-kuma.conf</summary>
```bash
server {
  listen 443 ssl http2;
  server_name sub.domain.com;
  ssl_certificate     /ssl/fullchain.pem;
  ssl_certificate_key /ssl/privkey.pem;

  location / {
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   Host $host;
    proxy_pass         http://HOSTNAME:3001/;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection "upgrade";
  }
}
```
</details>

"HOSTNAME" muss ersetzt werden. Hier muss der korrekte Name des Docker Containers eingetragen werden, ansonsten kann der Traffic nicht weitergeleitet werden. Der Hostname des Containers ist bei dem jeweiligen Add-On unter dem Punkt "Informationen" zu finden. 

## Integrationen
----------------
Über [Integrationen](https://www.home-assistant.io/integrations/) können die verschiedensten Hersteller sowie Produkte in Home Assistant eingebunden werden. 
