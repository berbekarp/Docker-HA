# 🏡 Home Assistant Okosotthon Szerver

Ez a repozitórium a Home Assistant alapú okosotthon rendszerem teljes dokumentációját és konfigurációs fájljait tartalmazza. A rendszer egy nagy teljesítményű, szünetmentesített Raspberry Pi 5 hardverre és Docker konténeres virtualizációra épül.

## 🛠️ Hardver Specifikáció
* **SBC:** Raspberry Pi 5 (8GB RAM)
* **Tárhely:** Geekworm X1001 Raspberry Pi 5 PCIe to M.2 NVMe SSD board (+ NVMe SSD) - Samsung 970 EVOPlus 128GB
* **Tápellátás/UPS:** Geekworm x1200 UPS board
* **Ház:** Geekworm X1200-C1 Metal Case
* **Zigbee / Z-Wave Koordinátor:** SLZB-MRW10 by SMLIGHT (Hálózati/PoE koordinátor)

### 🛒 Hol szerezhetőek be az eszközök?
Ha te is hasonló rendszert építenél, itt találod a hivatalos és ajánlott beszerzési forrásokat:
* **Raspberry Pi 5 (8GB):** [Málna PC (Hivatalos magyar forgalmazó)](https://www.malnapc.hu/) vagy [Raspberry Pi Global](https://www.raspberrypi.com/products/raspberry-pi-5/)
* **Geekworm X1200 UPS Shield:** [Geekworm Hivatalos Webshop](https://geekworm.com/products/x1200)
* **Geekworm X1001 NVMe SSD Shield:** [Geekworm Hivatalos Webshop](https://geekworm.com/products/x1001)
* **Geekworm X1200-C1 Fémház:** [Geekworm Hivatalos Webshop](https://geekworm.com/products/x1200-c1) *(Kifejezetten az X1200-hoz és a Pi 5-höz tervezve)*
* **SLZB-MRW10 Koordinátor:** [SMLIGHT Hivatalos Oldal](https://smlight.tech/) vagy hazai forgalmazótól az [Okosotthon.bolt.hu](https://okosotthon.bolt.hu/webaruhaz/termek/smlight-slzb-mrw10-multiradio-poe-adapter/)-ról.

---

## 💾 1. Váltás SD kártyáról NVMe SSD-re és Ubuntu Telepítés

A maximális sebesség és megbízhatóság érdekében a rendszert SD kártya helyett a Geekworm X1001-es kártyán lévő NVMe SSD-ről fogjuk futtatni. Mivel a gépedben jelenleg bent van a régi SD kártya és az új NVMe SSD is, ezt a cserét közvetlenül a Raspberry Pi-n el tudjuk végezni.

### Ubuntu telepítése a Raspberry Pi Imagerrel (az NVMe SSD-re)
Először felírjuk az új operációs rendszert az SSD-re, miközben a régi rendszer (Raspberry Pi OS) fut az SD kártyáról.
1. Indítsd el a Pi-t az SD kártyával, és nyisd meg a [Raspberry Pi Imager](https://www.raspberrypi.com/software/) szoftvert (ha nincs telepítve, telepítsd a terminálból: `sudo apt install rpi-imager`).
2. **Készülék kiválasztása:** Válaszd a *Raspberry Pi 5*-öt.
3. **OS kiválasztása:** Menj az *Other general-purpose OS* -> *Ubuntu* menübe, és válaszd az **Ubuntu 24.04 LTS (64-bit)** verziót.
4. **Tároló kiválasztása:** Válaszd ki az **NVMe SSD-det** a listából.
5. *(Opcionális)* A fogaskerék ikonra kattintva már előre beállíthatod a Wi-Fi adatokat, a hosztnevet és az SSH hozzáférést a kényelmesebb első induláshoz.
6. Kattints a **Write (Írás)** gombra, és várd meg, amíg a folyamat befejeződik.

### A Boot sorrend átállítása (hogy az SSD-ről induljon)
Most, hogy az SSD-n már ott a friss Ubuntu, meg kell mondanunk a hardvernek, hogy mostantól ne az SD kártyáról próbáljon elindulni.
1. Nyiss egy terminált a továbbra is futó Raspberry Pi OS-ben.
2. Lépj be a konfigurációs menübe:
   ```bash
   sudo raspi-config
   ```
3. Navigálj ide: **Advanced Options** -> **Boot Order**.
4. Válaszd ki az **NVMe/USB Boot** opciót (hogy elsődlegesen az SSD-t keresse indításkor).
5. Mentsd el és lépj ki.
6. Állítsd le teljesen a gépet (`sudo shutdown now`), **távolítsd el az SD kártyát**, majd kapcsold be újra a Pi-t. A rendszer mostantól a villámgyors NVMe SSD-ről fog elindulni!

---

## ⚙️ 2. Ubuntu Hardveres Finomhangolás

Az Ubuntu sikeres SSD-s indulása után a Geekworm kiegészítők (NVMe és UPS) maximális teljesítményéhez és megfelelő működéséhez be kell állítani a Raspberry Pi boot konfigurációját.

1. Nyisd meg a boot konfigurációs fájlt:
```bash
sudo nano /boot/firmware/config.txt

```


2. Add hozzá (vagy engedélyezd) az alábbi sorokat a fájl végén:
```ini
# PCIe Gen 3 engedélyezése a Geekworm X1001 NVMe SSD maximális sebességéhez
dtparam=pciex1
dtparam=pciex1_gen=3

# I2C busz engedélyezése a Geekworm X1200 UPS kommunikációjához
dtparam=i2c_arm=on

```


3. Mentsd el a fájlt (Ctrl+O, Enter, Ctrl+X), majd indítsd újra a rendszert:
```bash
sudo reboot

```



---

## 🐳 3. Docker és Portainer Telepítése

A Home Assistant és a kiegészítők futtatásához a Docker Engine és a Portainer (grafikus konténerkezelő) telepítése szükséges.

**Docker telepítése:**

```bash
curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

```

*(A parancsok kiadása után jelentkezz ki, majd be, hogy a jogosultságok érvénybe lépjenek).*

**Portainer CE telepítése:**

```bash
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

```

A Portainer webes felülete a `https://<RASPBERRY_IP>:9443` címen érhető el, ahol az első belépéskor létre kell hozni az adminisztrátori fiókot.

---

## 🚀 4. Konténerek Indítása (Docker Compose)

A rendszer összes szolgáltatását egyetlen **[`compose.yaml`](https://www.google.com/search?q=content/compose.yaml)** fájl fogja össze, amely megtalálható a repó `content/` mappájában.

**Szolgáltatások:**

* **Home Assistant:** A fő okosotthon központ (Host hálózaton).
* **Mosquitto:** MQTT bróker az eszközök közötti üzenetküldéshez.
* **Z-Wave JS (zwavejs2mqtt):** Z-Wave eszközök kezelése és híd az MQTT felé.
* **Cloudflared:** Biztonságos távoli elérés (Cloudflare Tunnel).

**Indítás:**
Lépj abba a mappába, ahol a `compose.yaml` fájlod van, és add ki a következő parancsot:

```bash
docker compose up -d

```

---

## 🔐 5. Mosquitto MQTT Alapbeállítás és Jelszóadás

Alapértelmezetten a Mosquitto nem enged be jelszó nélkül. Létre kell hoznunk a konfigurációs fájlt és egy felhasználót a parancssorból.

**1. Konfigurációs fájl létrehozása:**
A host gépen hozz létre egy `mosquitto.conf` fájlt a `/mosquitto` mappában:

```bash
sudo nano /mosquitto/mosquitto.conf

```

Tartalma legyen ez:

```text
listener 1883
allow_anonymous false
password_file /mosquitto/pwfile

```

**2. Jelszó generálása a futó konténeren belül:**
Miután a konténer elindult, a következő paranccsal hozhatsz létre egy MQTT felhasználót (pl. `mqtt_user` néven):

```bash
docker exec -it mosquitto mosquitto_passwd -c /mosquitto/pwfile mqtt_user

```

A rendszer bekéri az új jelszót. Ha megadtad, indítsd újra a konténert, hogy a beállítások érvénybe lépjenek:

```bash
docker restart mosquitto

```

---

## 📡 6. Z-Wave JS UI (zwavejs2mqtt) Részletes Beállítások

A Z-Wave kezelőfelület a `http://<RASPBERRY_IP>:8091` címen érhető el. Mivel az **SLZB-MRW10** egy hálózati (PoE/LAN) koordinátor, a fizikai USB portok helyett TCP protokollon keresztül kapcsolódunk hozzá.

1. Lépj be a webes felületre, és menj a **Settings (Beállítások) -> Z-Wave** menüpontba.
2. **Soros port (Serial Port):** * Válaszd ki a TCP csatlakozást.
* Cím (Port): Írd be az SLZB-MRW10 IP címét és portját a következő formátumban: `tcp://<SLZB_MRW10_IP_CIME>:6638` (a 6638 a SMLIGHT eszközök alapértelmezett Z-Wave portja, ellenőrizd az eszköz webes felületén).


3. **Biztonsági Kulcsok (Security Keys):**
* A modern Z-Wave eszközök biztonságos (S2) kommunikációjához kulcsokra van szükség.
* Ugyanitt a beállításokban keresd meg a kulcsokat (S0 Legacy, S2 Access Control, S2 Authenticated, S2 Unauthenticated).
* Ha még üresek, kattints az ikonra a **generáláshoz (Generate)**. *Ezeket a kulcsokat mentsd el egy biztonságos helyre, mert ha elvesznek, újra kell párosítani az összes S2-es eszközt!*


4. **MQTT szekció:**
* Host: `mosquitto` (mivel egy docker hálózaton vannak, elég a konténer nevét megadni)
* Port: `1883`
* Auth: Kapcsold be, és add meg az előző lépésben létrehozott MQTT felhasználónevet és jelszót.


5. Mentsd el a beállításokat. A Home Assistantban a Z-Wave integráció hozzáadásakor a `ws://<RASPBERRY_IP>:3000` címet kell majd megadni.

---

## 🌐 7. Cloudflare Tunnel Beállítása (Távoli Elérés)

A biztonságos, portnyitás (port forwarding) nélküli távoli elérést a Cloudflare biztosítja. A `compose.yaml`-ben található `tunnel` konténerhez szükség van egy egyedi azonosítóra (Token).

1. Lépj be a [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) felületére.
2. Navigálj a **Networks -> Tunnels** menüpontba.
3. Kattints a **Create a tunnel** gombra, majd válaszd a **Cloudflared** opciót.
4. Adj nevet a tunnelnek (pl. `HomeAssistant-Pi5`).
5. A megjelenő telepítési opcióknál válaszd a **Docker** környezetet. Itt látni fogsz egy hosszú parancsot, amiben szerepel a token:
`--token eyJhbGciOiJIUzI1NiIsInR...`
6. **Csak a token kódsorát másold ki!**
7. Ezt a tokent illeszd be a `compose.yaml` fájlodba a `TUNNEL_TOKEN=<tunnel-token>` sorhoz (a kacsacsőrök helyére).
*(⚠️ Fontos: A tokent kezeld jelszóként! Ha a compose.yaml fájlt feltöltöd a nyilvános GitLab repódba, a tokent cseréld ki egy dummy szövegre, és csak a saját szervereden lévő fájlba írd bele az igazit!)*
8. A Cloudflare felületén a "Public Hostnames" fülnél állítsd be a saját domainedet, a "Service" típusnál válaszd a `HTTP`-t, a cél URL pedig legyen a Home Assistant belső címe: `http://<RASPBERRY_IP>:8123`.

---

## 🔄 8. Docker Konténerek Frissítése (Egyesével)

A Docker Compose használatának egyik legnagyobb előnye, hogy a szolgáltatásokat (konténereket) egyesével is tudod frissíteni, így a rendszer többi része zavartalanul futhat tovább. 

Ha például csak a Home Assistantot vagy a Z-Wave JS-t szeretnéd a legújabb verzióra frissíteni, kövesd az alábbi lépéseket:

**1. Lépj be abba a mappába, ahol a `compose.yaml` fájlod található:**
```bash
cd /eleresi/utvonal/a/compose/fajlhoz

```

**2. Töltsd le a legújabb frissítést (Image pull):**
Add ki a `pull` parancsot, mögé írva a frissíteni kívánt szolgáltatás pontos nevét (ahogy a compose fájlban szerepel, pl. `homeassistant`, `zwavejs2mqtt`, `mosquitto` vagy `tunnel`). Ez csak letölti a frissítést, de még nem állítja le a futó rendszert.

```bash
docker compose pull homeassistant

```

**3. Indítsd újra a konténert az új verzióval:**
Az alábbi parancs leállítja a régi konténert, törli, és azonnal létrehozza az újat a frissített fájlokkal, megtartva az összes beállításodat.

```bash
docker compose up -d homeassistant

```

**4. (Opcionális) Takarítsd ki a feleslegessé vált régi fájlokat:**
Minden frissítés után a régi, már nem használt image-ek (telepítőfájlok) a tárhelyen maradnak. Helyfelszabadítás céljából érdemes ezeket törölni:

```bash
docker image prune -f

```

*(A `-f` kapcsoló miatt nem fog rákérdezni a törlésre, hanem automatikusan eltávolítja a már nem használt image-eket).*
