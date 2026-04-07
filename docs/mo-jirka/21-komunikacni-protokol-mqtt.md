Zde je přepracovaná maturitní otázka na téma MQTT, upravená do stejného stylu, v jakém jsme připravili pasivní součástky. Text je sjednocen, zjednodušen pro výuku a doplněn o praktické poznámky.

---

# Otázka 21 - Komunikační protokol MQTT

## Slovo úvodem
**MQTT** (*Message Queuing Telemetry Transport*) je extrémně lehký komunikační protokol navržený pro situace, kde je omezená propustnost sítě, nízký výpočetní výkon nebo požadavek na minimální spotřebu energie. Proto se stal de-facto standardem pro **IoT** (Internet věcí).

Na rozdíl od klasického modelu "klient-server" (např. HTTP), kde se klient přímo ptá serveru, využívá MQTT návrhový vzor **Publisher/Subscriber**.

---

## Princip fungování

### Broker (Centrální bod)
Broker je "mozkem" celé sítě. Žádná zpráva nejde přímo ze zařízení do zařízení, vše jde přes brokera. Jeho úkolem je přijímat zprávy, třídit je a rozesílat je těm, kteří o ně projevili zájem.

> Existuje spousta veřejných brokerů (např. `broker.hivemq.com`, `broker.emqx.io`), ale pro seriózní projekty se doporučuje vlastní broker (např. **Mosquitto**), který můžete provozovat na Raspberry Pi nebo v cloudu, routeru s OpenWRT, ESP32 atd.

### Publisher a Subscriber
- **Publisher (Vydavatel):** Zařízení (např. teploměr), které odesílá data na brokera pod určitým tématem.
- **Subscriber (Odběratel):** Zařízení (např. mobilní aplikace nebo displej), které řekne brokeru: „Chci dostávat všechny zprávy z tohoto tématu.“
- **Topic (Téma):** Adresa nebo "schránka", kam zpráva putuje. Má hierarchickou strukturu oddělenou lomítky (např. `dum/obyvak/teplota`).

> **Analogie s časopisem:** Broker je pošta. Publisher je redakce, která napíše článek do rubriky "Zahrada". Subscriber je čtenář, který si na poště předplatil rubriku "Zahrada". Jakmile redakce článek pošle, pošta (broker) jej automaticky doručí všem předplatitelům. Redakce neví, kdo čte, a čtenář nemusí vědět, kdo článek napsal.

---

## Vlastnosti témat (Topics)
- Témata jsou řetězce v **UTF-8** (podporují diakritiku, i když se v praxi raději nepoužívá).
- **Case-sensitive:** `Teplota` a `teplota` jsou dvě různá témata.
- **Dynamika:** Téma nemusíte předem zakládat. Pokud Publisher pošle zprávu do nového tématu, broker ho automaticky vytvoří.
- **Wildcards (Zástupné znaky):** Používají se při odběru pro sledování více témat najednou.
    1. **Single-level (+):** Nahrazuje právě jednu úroveň.
       - `myhome/+/temperature` → sleduje teplotu v přízemí i v patře, ale ne v celém domě najednou.
![](images/21-single-level-wild-card.png)
    1. **Multi-level (#):** Nahrazuje všechny zbývající úrovně (musí být na konci).
       - `myhome/#` → odebírá úplně vše, co se v domě "šustne".
![](images/21-multilevel-wild-card.png)

---

## Datová zpráva (Payload)
MQTT je tzv. **payload agnostic**. To znamená, že protokolu je úplně jedno, co posíláte. Může to být:
- Čistý text (např. `"ON"`, `"OFF"`).
- Číslo (např. `23.5`).
- Formát **JSON** (nejčastější v moderním IoT pro přenos strukturovaných dat).
- Binární data (např. obrázek).
- Maximální velikost zprávy je teoreticky **256 MB**, ale v IoT se pohybujeme v řádu bytů.

---

## QoS - Kvalita služeb
QoS definuje, jak moc si chce být odesílatel jistý, že zpráva dorazila k brokerovi.
- **QoS 0 (At most once):** "Pošli a zapomeň". Nejrychlejší, ale zpráva se může ztratit.
- **QoS 1 (At least once):** Zpráva je doručena **alespoň jednou**. Příjemce potvrzuje přijetí, pokud potvrzení nedojde, odesílatel posílá znovu (může dojít k duplicitě).
- **QoS 2 (Exactly once):** Zpráva je doručena **právě jednou**. Nejpomalejší, vyžaduje čtyřcestný "handshake", ale zaručuje stoprocentní přesnost.

---

## Bezpečnost
MQTT sám o sobě nemá vestavěné zabezpečení, ale běžně máme 3 možnosti:

1. **Nešifrované spojení (port 1883) s anonymbími klienty:** Nejjednodušší, ale nejméně bezpečné. Vhodné pro testování nebo izolované sítě.
2. **Nešifrované spojení s autentizací:** Broker vyžaduje uživatelské jméno a heslo, ale data jsou stále přenášena v otevřeném textu.
3. **Šifrované spojení (port 8883) přes TLS/SSL:** Nejbezpečnější, data jsou šifrována, ale vyžaduje více výpočetního výkonu a správu certifikátů. Šifrování také zvyšuje zátěž na CPU, což může být problém pro některé mikrokontroléry nebo low power aplikace.



---

## Praktická ukázka: MicroPython (ESP32/RPi Pico W)

Při práci s MQTT na mikrokontrolérech používáme knihovnu `umqtt.simple`. Důležitým parametrem je **keepalive**, což je čas (v sekundách), po který broker drží spojení otevřené, i když se nic neposílá. Klient musí poslat kontrolní "ping", aby nebyl odpojen.

```python
import network
import time
from machine import Pin
from umqtt.simple import MQTTClient

# 1. Připojení k Wi-Fi
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect("SSID_NAZEV", "HESLO")

# 2. Definice Callbacku (funkce, co se spustí, když dorazí zpráva)
def callback_funkce(topic, msg):
    print(f"Prijata zprava: {msg} v tematu: {topic}")

# 3. Nastavení klienta
mqtt_server = 'broker.hivemq.com' # Veřejný testovací broker
client_id = 'moje_esp32_001'
topic_pub = b'skola/ucebna1/senzor1'
topic_sub = b'skola/ucebna1/ovladani'

client = MQTTClient(client_id, mqtt_server, keepalive=60)
client.set_callback(callback_funkce)

# 4. Hlavní smyčka
try:
    client.connect()
    client.subscribe(topic_sub)
    print("Pripojeno a odebira se...")
    
    while True:
        # Publikování dat (např. simulace teploty)
        client.publish(topic_pub, b"22.5")
        
        # Kontrola příchozích zpráv
        client.check_msg()
        time.sleep(5)
except:
    print("Chyba spojeni, restartuji...")
    time.sleep(5)
    machine.reset()
```

> Proč používáme u témat a zpráv předponu `b` (např. `b'data'`)? V MicroPythonu to znamená **bytes**. MQTT protokol nepracuje přímo s objekty typu *string*, ale s poli bajtů, což šetří RAM MCU.


> Pro instalaci knihovny `umqtt.simple` na ESP32/RPi Pico W použijte jeho vREPL:

```python
import mip
mip.install("umqtt.simple")
```
> Nezapomeňte desku pred tím připojit k WiFi:

```python
import network
wifi = network.WLAN(network.STA_IF)
wifi.active(True)
wifi.connect("SSID_NAZEV", "HESLO")
```

---

### Shrnutí MQTT
- **Výhody:** Nízká režie, funguje i na nestabilních sítích, obrovská škálovatelnost.
- **Nevýhody:** Vyžaduje centrální prvek (pokud spadne broker, celá síť "ztichne").
- **Zabezpečení:** Standardně běží na portu **1883** (nešifrovaně) nebo **8883** (šifrovaně přes TLS/SSL).