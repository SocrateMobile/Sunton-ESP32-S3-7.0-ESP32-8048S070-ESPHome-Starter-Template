# Sunton ESP32-S3 7.0" (ESP32-8048S070) – ESPHome Starter Template

Ce dépôt propose une configuration ESPHome moderne (compatible ESPHome 2024.11+ / 2026.x et ESP-IDF 5.x) pour la carte de développement **Sunton ESP32-S3 7.0 pouces** (`ESP32-8048S070`).

Cette base résout les instabilités courantes rencontrées sur ce modèle :
- **Suppression des scintillements et du tearing** (effet de balayage de haut en bas) grâce au pilote `mipi_rgb` cadencé à 16 MHz et au rafraîchissement contrôlé (`auto_clear_enabled: true`).
- **Tactile GT911 stable** : fonctionnement en polling I2C direct sans conflit d'interruption ni de broche de reset bloquante.
- **Résolution des conflits matériels ESP32-S3** : désactivation de la console USB-Serial-JTAG qui entre en conflit avec les broches I2C (GPIO 19 et 20).
- **Navigation multi-onglets** réactive (Page 1, Page 2, Page 3) avec affichage dynamique de la version de Home Assistant.

---

## 🛠️ Caractéristiques techniques de la carte

| Composant | Spécification |
| :--- | :--- |
| **Modèle** | Sunton ESP32-8048S070C (Capacitif) / ESP32-8048S070N |
| **SoC** | ESP32-S3-WROOM-1-N16R8 |
| **Processeur** | Xtensa® 32-bit LX7 dual-core @ 240 MHz |
| **Mémoire Flash** | 16 Mo (Quad SPI / DIO) |
| **PSRAM** | 8 Mo Octal SPI (OPI) @ 80 MHz |
| **Dalle LCD** | TFT TN / IPS 7,0 pouces |
| **Résolution** | 800 × 480 pixels (format 16:9) |
| **Interface écran** | Bus parallèle RGB 16 bits (DPI / MIPI RGB standard) |
| **Dalle tactile** | Écran capacitif Goodix GT911 (Bus I2C @ `0x5D`) |
| **Rétroéclairage** | Contrôle PWM via transistor sur le **GPIO 02** |
| **Alimentation** | 5V via connecteur USB-C ou bornier JST |

---

## 📌 Brochage matériel (Pinout)

### Bus RGB (Écran)
* **Signaux de contrôle :** `DE: 41`, `HSYNC: 39`, `VSYNC: 40`, `PCLK: 42`
* **Canal Rouge (5 bits) :** `GPIO 14`, `GPIO 21`, `GPIO 47`, `GPIO 48`, `GPIO 45`
* **Canal Vert (6 bits) :** `GPIO 9`, `GPIO 46`, `GPIO 3`, `GPIO 8`, `GPIO 16`, `GPIO 1`
* **Canal Bleu (5 bits) :** `GPIO 15`, `GPIO 7`, `GPIO 6`, `GPIO 5`, `GPIO 4`

### Contrôleur tactile (GT911)
* **SDA :** `GPIO 19`
* **SCL :** `GPIO 20`
* **Adresse I2C :** `0x5D` (mode polling, broches INT et RST matérielles gérées hors ESPHome pour éviter tout gel du bus)

---

## 📦 Code ESPHome complet (`sunton-7inch-starter.yaml`)

Ce modèle affiche une page de démarrage synchronisée, puis bascule sur une interface à 3 onglets ("Page 1", "Page 2", "Page 3") affichant le texte « Hello World » et la version de votre serveur Home Assistant.

```yaml
substitutions:
  devicename: "ecran-sunton-7"
  friendly_name: "Écran Sunton 7 Pouces"
  project_name: "Sunton.ESP32-S3-8048S070"
  project_version: "2.0"

esphome:
  name: ${devicename}
  min_version: 2024.11.0
  project:
    name: ${project_name}
    version: ${project_version}
  build_flags:
    - "-DBOARD_HAS_PSRAM"
  on_boot:
    priority: 600
    then:
      - light.turn_on: display_backlight

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  flash_size: 16MB
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240: y
      CONFIG_ESP32S3_DATA_CACHE_64KB: y
      CONFIG_SPIRAM_FETCH_INSTRUCTIONS: y
      CONFIG_SPIRAM_RODATA: y
      CONFIG_ESP_MAIN_TASK_STACK_SIZE: "32768"
      CONFIG_FREERTOS_HZ: "1000"
      # Libère les GPIO 19 et 20 (utilisés par l'I2C) du multiplexage USB-Serial-JTAG
      CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG: n

api:
  encryption:
    key: !secret api_encryption_key

captive_portal:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: false

logger:
  hardware_uart: UART0
  baud_rate: 0
  level: INFO

ota:
  - platform: esphome
    encryption:

psram:
  mode: octal
  speed: 80MHz

# ---------------------------------------------------------
# VARIABLES GLOBALES
# ---------------------------------------------------------
globals:
  - id: boot_done
    type: bool
    initial_value: 'false'
  - id: display_page
    type: int
    restore_value: false
    initial_value: '0'

# ---------------------------------------------------------
# HORLOGE DE RAFRAÎCHISSEMENT
# ---------------------------------------------------------
interval:
  - interval: 500ms
    then:
      - lambda: |-
          if (!id(boot_done)) {
            bool api_ok  = api_is_connected();
            bool time_ok = id(ha_time).now().is_valid();
            
            if (api_ok && time_ok) {
              id(boot_done) = true;
              id(display_page) = 0;
            }
            id(my_display).update();
          }
  - interval: 15s
    then:
      - lambda: |-
          if (id(boot_done)) {
            id(my_display).update();
          }

# ---------------------------------------------------------
# BUS I2C & TACTILE GT911
# ---------------------------------------------------------
i2c:
  - id: bus_a
    sda: 19
    scl: 20
    frequency: 400kHz
    scan: true

touchscreen:
  platform: gt911
  id: my_touchscreen
  i2c_id: bus_a
  address: 0x5D
  transform:
    swap_xy: false
    mirror_x: false
    mirror_y: false
  on_touch:
    - lambda: |-
        if (!id(boot_done)) return;
        static uint32_t last_touch_ms = 0;
        uint32_t now = millis();
        if (now - last_touch_ms < 250) return;

        int x = touch.x;
        int y = touch.y;
        
        // Détection de la barre de navigation inférieure (y >= 420)
        if (y >= 420) {
          int next_page = id(display_page);
          if (x < 266) next_page = 0;
          else if (x < 533) next_page = 1;
          else next_page = 2;

          if (next_page != id(display_page)) {
            last_touch_ms = now;
            id(display_page) = next_page;
            id(my_display).update();
          }
        }

# ---------------------------------------------------------
# POLICES DE CARACTÈRES
# ---------------------------------------------------------
font:
  - file:
      type: gfonts
      family: "Rubik"
      weight: 700
    id: font_24
    size: 24
    glyphsets: [GF_Latin_Kernel]
    ignore_missing_glyphs: true
    glyphs: ' abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!"%()+,-_.:/[]''?çéèàùâêîôûëïüÿœæÇÉÈÀÙÂÊÎÔÛËÏÜŸŒÆ€°=;|'

  - file:
      type: gfonts
      family: "Rubik"
      weight: 700
    id: font_32
    size: 32
    glyphsets: [GF_Latin_Kernel]
    ignore_missing_glyphs: true
    glyphs: ' abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!"%()+,-_.:/[]''?çéèàùâêîôûëïüÿœæÇÉÈÀÙÂÊÎÔÛËÏÜŸŒÆ€°=;|'

  - file:
      type: gfonts
      family: "Rubik"
      weight: 700
    id: font_48
    size: 48
    glyphsets: [GF_Latin_Kernel]
    ignore_missing_glyphs: true
    glyphs: ' abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!"%()+,-_.:/[]''?çéèàùâêîôûëïüÿœæÇÉÈÀÙÂÊÎÔÛËÏÜŸŒÆ€°=;|'

# ---------------------------------------------------------
# CONFIGURATION ÉCRAN MIPI RGB (800x480)
# ---------------------------------------------------------
display:
  - platform: mipi_rgb
    model: RPI
    id: my_display
    color_order: rgb
    invert_colors: true
    update_interval: never
    auto_clear_enabled: true
    dimensions:
      width: 800
      height: 480
    de_pin: 41
    hsync_pin: 39
    vsync_pin: 40
    pclk_pin: 42
    pclk_frequency: 16MHz
    pclk_inverted: true
    hsync_pulse_width: 4
    hsync_back_porch: 40
    hsync_front_porch: 48
    vsync_pulse_width: 4
    vsync_back_porch: 31
    vsync_front_porch: 13
    data_pins:
      red: [14, 21, 47, 48, 45]
      green: [9, 46, 3, 8, 16, 1]
      blue: [15, 7, 6, 5, 4]
    lambda: |-
      const auto COLOR_WHITE      = Color(255, 255, 255);
      const auto COLOR_BLACK      = Color(0, 0, 0);
      const auto COLOR_BLUE       = Color(0, 120, 230);
      const auto COLOR_GREEN      = Color(40, 167, 69);
      const auto COLOR_DARK_GRAY  = Color(120, 120, 120);

      // -------------------------------------------------------
      // ÉCRAN DE BOOT (ATTENTE CONNEXION)
      // -------------------------------------------------------
      if (!id(boot_done)) {
        bool api_ok  = api_is_connected();
        bool time_ok = id(ha_time).now().is_valid();

        it.printf(400, 100, id(font_48), COLOR_BLUE, TextAlign::TOP_CENTER, "DÉMARRAGE DU SYSTÈME");
        it.printf(400, 200, id(font_32), COLOR_BLACK, TextAlign::TOP_CENTER, "Initialisation des liaisons...");

        it.printf(400, 280, id(font_24), api_ok ? COLOR_GREEN : COLOR_DARK_GRAY, TextAlign::TOP_CENTER, 
                  api_ok ? "[OK] Home Assistant connecté" : "[..] Connexion à Home Assistant...");
        it.printf(400, 320, id(font_24), time_ok ? COLOR_GREEN : COLOR_DARK_GRAY, TextAlign::TOP_CENTER, 
                  time_ok ? "[OK] Horloge réseau synchronisée" : "[..] Synchronisation de l'heure...");
        return;
      }

      // -------------------------------------------------------
      // INTERFACE PRINCIPALE
      // -------------------------------------------------------
      int current_p = id(display_page);

      // Titre supérieur
      it.printf(20, 15, id(font_32), COLOR_DARK_GRAY, TextAlign::TOP_LEFT, "Sunton ESP32-S3 7.0\"");

      // Signal WiFi (en haut à droite)
      float rssi = id(wifi_rssi).state;
      int bars = 0;
      if (!std::isnan(rssi)) {
        if (rssi > -60) bars = 4; else if (rssi > -70) bars = 3; else if (rssi > -80) bars = 2; else if (rssi > -90) bars = 1;
      }
      for (int b = 0; b < 4; b++) {
        int bar_h = (b + 1) * 6;
        int bx = 745 + (b * 8);
        int by = 40 - bar_h;
        if (b < bars) it.filled_rectangle(bx, by, 5, bar_h, COLOR_BLUE);
        else it.rectangle(bx, by, 5, bar_h, COLOR_DARK_GRAY);
      }

      // Récupération de la version de Home Assistant
      std::string ha_ver = id(text_sensor_ha_version).state;
      if (ha_ver.empty() || ha_ver == "unavailable" || ha_ver == "unknown") {
        ha_ver = "Connexion en cours...";
      }

      // -------------------------------------------------------
      // CONTENU DES PAGES
      // -------------------------------------------------------
      if (current_p == 0) {
        it.printf(400, 140, id(font_48), COLOR_BLUE, TextAlign::TOP_CENTER, "Hello World - Page 1");
        it.printf(400, 240, id(font_32), COLOR_BLACK, TextAlign::TOP_CENTER, "Version Home Assistant :");
        it.printf(400, 290, id(font_32), COLOR_DARK_GRAY, TextAlign::TOP_CENTER, "%s", ha_ver.c_str());
      }
      else if (current_p == 1) {
        it.printf(400, 140, id(font_48), COLOR_GREEN, TextAlign::TOP_CENTER, "Hello World - Page 2");
        it.printf(400, 240, id(font_32), COLOR_BLACK, TextAlign::TOP_CENTER, "Version Home Assistant :");
        it.printf(400, 290, id(font_32), COLOR_DARK_GRAY, TextAlign::TOP_CENTER, "%s", ha_ver.c_str());
      }
      else if (current_p == 2) {
        it.printf(400, 140, id(font_48), COLOR_BLACK, TextAlign::TOP_CENTER, "Hello World - Page 3");
        it.printf(400, 240, id(font_32), COLOR_BLACK, TextAlign::TOP_CENTER, "Version Home Assistant :");
        it.printf(400, 290, id(font_32), COLOR_DARK_GRAY, TextAlign::TOP_CENTER, "%s", ha_ver.c_str());
      }

      // -------------------------------------------------------
      // BARRE DE NAVIGATION INFÉRIEURE
      // -------------------------------------------------------
      it.line(0, 420, 800, 420, COLOR_DARK_GRAY);
      it.line(266, 420, 266, 480, COLOR_DARK_GRAY);
      it.line(533, 420, 533, 480, COLOR_DARK_GRAY);

      // Bouton Onglet 0
      it.filled_rectangle(0, 421, 266, 59, current_p == 0 ? COLOR_BLUE : COLOR_WHITE);
      it.printf(133, 450, id(font_32), current_p == 0 ? COLOR_WHITE : COLOR_BLACK, TextAlign::CENTER, "Page 1");

      // Bouton Onglet 1
      it.filled_rectangle(267, 421, 266, 59, current_p == 1 ? COLOR_BLUE : COLOR_WHITE);
      it.printf(400, 450, id(font_32), current_p == 1 ? COLOR_WHITE : COLOR_BLACK, TextAlign::CENTER, "Page 2");

      // Bouton Onglet 2
      it.filled_rectangle(534, 421, 266, 59, current_p == 2 ? COLOR_BLUE : COLOR_WHITE);
      it.printf(666, 450, id(font_32), current_p == 2 ? COLOR_WHITE : COLOR_BLACK, TextAlign::CENTER, "Page 3");

# ---------------------------------------------------------
# RÉTROÉCLAIRAGE
# ---------------------------------------------------------
output:
  - platform: ledc
    id: gpio_backlight_pwm
    pin: GPIO02
    frequency: 1220

light:
  - platform: monochromatic
    output: gpio_backlight_pwm
    name: "${friendly_name} Rétroéclairage"
    icon: mdi:lightbulb-on
    id: display_backlight
    restore_mode: ALWAYS_ON

time:
  - platform: homeassistant
    id: ha_time

# ---------------------------------------------------------
# ENTITÉS ET CAPTEURS HOME ASSISTANT
# ---------------------------------------------------------
sensor:
  - platform: wifi_signal
    id: wifi_rssi
    internal: true
    update_interval: 15s

text_sensor:
  # Entité Home Assistant exposant la version actuelle du serveur
  - platform: homeassistant
    id: text_sensor_ha_version
    entity_id: sensor.current_version
    internal: true
