# Home Assistant - Integracja z Energa S.A [PL]
Skrypt pozwala pobrać dane z licznika dostawcy Energa S.A oraz wysłać je na serwer MQTT. Dotyczy to zarówno licznika jednokierunkowego jak i dwukierunkowego (w taryfie G11 i G12W).

![MojLicznik Screenshot](https://github.com/wkawecki/Energa-HomeAssistant-Integration/blob/master/licznik_G12W.png?raw=true)

![HA Screenshot](https://github.com/wkawecki/Energa-HomeAssistant-Integration/blob/master/ha.png?raw=true)
## Ograniczenia.
Skrypt bazuje na pozyskiwaniu danych bezpośrednio ze stron internetowych (ang. Web scrapping), dlatego przy jakichkolwiek zmianach na stronie przez Energa S.A program może ulec awarii. Postaram się na bieżąco aktualizować to repetytorium.
## Instalacja
1. Tworzenie konta na [Mój Licznik](https://mojlicznik.energa-operator.pl)
2. Aktualizacja listy dostępnych pakietów:
    * `sudo apt update`
3. Instalacja wymaganych pakietów:
    * `sudo apt install python3 python3-pip git`
    * `sudo pip3 install selenium`
    * `sudo pip3 install pyvirtualdisplay`
    * `sudo pip3 install paho-mqtt`
    * `sudo apt install xvfb`
        * Dla RPI: `sudo apt install chromium-chromedriver`
        * Dla Debiana: `sudo apt install chromium-driver`
4. Instalacja skryptu:
    * `git clone https://github.com/wkawecki/Energa-HomeAssistant-Integration.git Energa-script `
    * `cd Energa-script && sudo chmod +x main.py`
## Użycie
```
usage: main.py [-h] -tm {1,2} -ms MQTT_SERVER [-mu MQTT_USERNAME]
               [-mp MQTT_PASSWORD] -mt MQTT_TOPIC -eu [-mo MQTT_PORT] ENERGA_USERNAME -ep
               ENERGA_PASSWORD 

required arguments:
  -tm {1,2,3,4}, --type_of_meter {1,2,3,4}
                        Typ licznika: 1 dla jednokierunkowego, 2 dla dwukierunkowego, 3 dla jednokierunkowego (taryfa G12W), 4 dla dwukierunkowego (taryfa G12W)
  -ms MQTT_SERVER, --mqtt_server MQTT_SERVER
                        Serwer MQTT
                        Haslo MQTT
  -mt MQTT_TOPIC, --mqtt_topic MQTT_TOPIC
                        Temat MQTT
  -eu ENERGA_USERNAME, --energa_username ENERGA_USERNAME
                        Login Energa S.A
  -ep ENERGA_PASSWORD, --energa_password ENERGA_PASSWORD
                        Haslo Energa S.A

optional arguments:
  -h, --help            show this help message and exit
  -mu MQTT_USERNAME, --mqtt_username MQTT_USERNAME
                        Uzytkownik MQTT
  -mp MQTT_PASSWORD, --mqtt_password MQTT_PASSWORD
                        Haslo MQTT
  -mo MQTT_PORT, --mqtt_port MQTT_PORT
                        Port MQTT

przykład: ./main.py -tm 1 -ms 192.168.19.129 -mt home/energa -eu email@gmail.com -ep haslo_energa -mu login_mqtt -mp hasło_mqtt -mo 1883
```
#### Wiadomość na serwerze MQTT po uruchomieniu skryptu:
![MQTT Screenshot](https://papuutekapt.github.io/assets/Energa-HomeAssistant-Integration/mqtt.png)
## Konfiguracja Home Assistant'a
W pliku configuration.yaml dodajemy (dla taryfy G11 - typ licznika 1 lub 2):
```
sensor energa:
  - platform: mqtt
    name: "Energia Pobrana" #Nazwa encji
    state_topic: "home/energa" #Nasz temat MQTT
    unit_of_measurement: "kWh"
    value_template: "{{ value_json.used }}"
  - platform: mqtt
    name: "Energia Oddana" #Nazwa encji
    state_topic: "home/energa" #Nasz temat MQTT
    unit_of_measurement: "kWh" 
    value_template: "{{ value_json.produced }}"
```
W pliku configuration.yaml dodajemy (dla taryfy G12W - typ licznika 3 lub 4):
```
sensor energa:
  - platform: mqtt
    name: "Energia pobrana dzienna"
    state_topic: "home/energa" 
    unit_of_measurement: "kWh"
    value_template: "{{ value_json.used_1 }}"
  - platform: mqtt
    name: "Energia pobrana nocna"
    state_topic: "home/energa" 
    unit_of_measurement: "kWh"
    value_template: "{{ value_json.used_2 }}"
  - platform: mqtt
    name: "Energia oddana dzienna" 
    state_topic: "home/energa"
    unit_of_measurement: "kWh" 
    value_template: "{{ value_json.produced_1 }}"    
  - platform: mqtt
    name: "Energia oddana nocna" 
    state_topic: "home/energa"
    unit_of_measurement: "kWh" 
    value_template: "{{ value_json.produced_2 }}"
  - platform: template
    sensors:
      energia_pobrana:
        friendly_name: Energia pobrana
        unit_of_measurement: kWh
        value_template: "{{ states('sensor.energia_pobrana_dzienna')|float + states('sensor.energia_pobrana_nocna')|float }}"
  - platform: template
    sensors:
      energia_oddana:
        friendly_name: Energia oddana
        unit_of_measurement: kWh
        value_template: "{{ states('sensor.energia_oddana_dzienna')|float + states('sensor.energia_oddana_nocna')|float }}"
```
## Automatyczne uruchanianie skryptu
Energa S.A pobiera dane z liczników w nocy. Niestety z doświadczenia wiem, że nie jest to stała godzina. Proponuję uruchaniać skrypt codziennie o 5 rano.
W tym celu dodajemy wpis do cron'a:
* `crontab -e`
```
0 5 * * * python3 /home/pi/Energa-script/main.py -tm 1 -ms 192.168.19.129 -mt home/energa -eu email@gmail.com -ep haslo_energa -mu login_mqtt -mp hasło_mqtt
```

lub co 2 godziny od 6 do 22
```
0 6-22/2 * * * python3 /home/pi/Energa-script/main.py -tm 4 -ms 192.168.1.134 -mt home/energa -eu email@gmail.com -ep haslo_energa -mu login_mqtt -mp hasło_mqtt -mo 1883
```
Jeśli nie masz zainstalowanego crona, możesz to zrobić to komendą: `sudo apt install cron`
## Testowane na:
* Raspbian 10 Buster na Raspberry PI 3B+,
* Raspbian 10 Buster na Raspberry PI 4B,
* Debian 10.
* Powinno działać na każdej dystrybucji bazującej na debianie.
# Home Assistant - Energa S.A Integration [EN]
Script allows to get data from polish energy distributor: Energa S.A 

![HA Screenshot](https://github.com/wkawecki/Energa-HomeAssistant-Integration/blob/master/ha.png?raw=true)
