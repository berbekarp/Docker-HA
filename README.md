# 🏡 Home Assistant Okosotthon Szerver

Ez a repozitórium a Home Assistant alapú okosotthon rendszerem teljes dokumentációját és konfigurációs fájljait tartalmazza. A rendszer egy nagy teljesítményű, szünetmentesített Raspberry Pi 5 hardverre és Docker konténeres virtualizációra épül.

## 🛠️ Hardver Specifikáció
* **SBC:** Raspberry Pi 5 (8GB RAM)
* **Tárhely:** Geekworm X1001 Raspberry Pi 5 PCIe to M.2 NVMe SSD board (+ NVMe SSD)
* **Tápellátás/UPS:** Geekworm x1200 UPS board
* **Ház:** Geekworm X1200-C1 Metal Case
* **Zigbee / Z-Wave Koordinátor:** SLZB-MRW10 by SMLIGHT (Hálózati/PoE koordinátor)

## 🐧 Operációs Rendszer
* **OS:** Ubuntu 24.04.2 LTS (Noble Numbat) 64-bit

---

## ⚙️ 1. Ubuntu Telepítés és Hardveres Finomhangolás

Az Ubuntu telepítése után a Geekworm kiegészítők (NVMe és UPS) maximális teljesítményéhez és megfelelő működéséhez be kell állítani a Raspberry Pi boot konfigurációját.

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

## 🐳 2. Docker és Portainer Telepítése

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

## 🚀 3. Konténerek Indítása (Docker Compose)

A rendszer összes szolgáltatását egyetlen `compose.yaml` fájl fogja össze, amely megtalálható a repó `content/` mappájában.

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

## 🔐 4. Mosquitto MQTT Alapbeállítás és Jelszóadás

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

## 📡 5. Z-Wave JS UI (zwavejs2mqtt) Alapbeállítások

A Z-Wave kezelőfelület a `http://<RASPBERRY_IP>:8091` címen érhető el. Mivel a `compose.yaml`-ben a fizikai USB portok ki vannak kommentelve, a Z-Wave hálózat az **SLZB-MRW10** eszközön keresztül, IP alapon kommunikál.

1. Lépj be a webes felületre, és menj a **Settings (Beállítások)** menübe.
2. **Z-Wave szekció:** * A *Serial Port* résznél válaszd a TCP/IP csatlakozást.
* Formátum: `tcp://<SLZB_MRW10_IP_CIME>:<PORT>` (A port általában a hálózati koordinátor webes felületén van megadva).


3. **MQTT szekció:**
* Host: `mosquitto` (vagy a szerver IP címe)
* Port: `1883`
* Hitelesítés: Add meg az előző lépésben létrehozott MQTT felhasználónevet és jelszót.


4. Mentsd el, majd engedélyezd a Z-Wave JS integrációt a Home Assistantban is (a `ws://<RASPBERRY_IP>:3000` címet használva).

```
