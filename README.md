# ESPHome Native Component for NSPanel (TheMysticle Fork)

This is a **fork** of the excellent [olicooper/esphome-nspanel-lovelace-native](https://github.com/olicooper/esphome-nspanel-lovelace-native) project.

This project is designed to utilise the UI provided by [NSPanel Lovelace UI](https://github.com/joBr99/nspanel-lovelace-ui), but use a native [ESPHome](https://github.com/esphome/esphome) component for the backend. My core goals for the project were to:
- **Reduce bandwidth consumption** - by utilising the efficient TCP communication implemented by ESPHome.
- **Remove the need for an MQTT broker** and make the device able to function on a minimal level during network outages.
- **Make the screen more responsive** by having the ESP32 do some of the heavy lifting (the ESP32 is actually quite powerful!)

While this fork maintains all of the original core card logic (Grid, Entities, Thermo, Media, etc.) developed by Oli Cooper, it introduces a complete **visual and functional overhaul of the Screensaver/Main Dashboard**.

### Key Modifications & Enhancements:
- **Image-Based Weather:** Replaced low-resolution font characters with high-quality Nextion Picture components (`p1.pic`).
- **Direct Protocol Sync:** Implemented a binary protocol to push temperature and button states directly to the screen. This bypasses the overhead of standard Lovelace parsing for the most common dashboard elements.
- **Synced Dashboard Buttons:** The screensaver features three dedicated toggle buttons (`bt0`, `bt1`, `bt2`) that stay in perfect sync with Home Assistant Relays, Lights, and Heating status (`hvac_action`).
- **Persistent Data:** Added a `requestUpdate` handler. When you wake the screen or navigate back to the screensaver, the Nextion "pulls" the current data from the ESP32, ensuring weather and button states are visible instantly without waiting for a state change.

A special thanks goes to these great projects for making this possible:
- [NSPanel Lovelace UI](https://github.com/joBr99/nspanel-lovelace-ui)
- [ESPHome NSPanel Lovelace UI](https://github.com/sairon/esphome-nspanel-lovelace-ui)
- [ESPHome](https://github.com/esphome/esphome)

---

## 📸 UI Gallery

| Custom Screensaver | Grid Card (Original Logic) | Climate Card |
| :---: | :---: | :---: |
| ![Screensaver Dashboard](https://github.com/TheMysticle/esphome-nspanel-lovelace-native/blob/dev/images/screensaver.png?raw=true) | ![Grid Card](https://github.com/TheMysticle/esphome-nspanel-lovelace-native/blob/dev/images/card%20page.png?raw=true) | ![Climate Page](https://github.com/TheMysticle/esphome-nspanel-lovelace-native/blob/dev/images/climate.png?raw=true) |

---

## 🛠 Installation & HMI Update

The installation of ESPHome on the ESP32 follows the standard ESPHome build method e.g. `esphome run --device COM6 ns-panel.yaml`. It is possible to do OTA updates with the ESPHome CLI after the initial upload.

## PSRAM

The NSPanel has on-board PSRAM which this project makes use of automatically, which means that the `psram` component is unavailable because the PSRAM is configured during code generation by adding specific `sdkconfig_options`. Additionally, when using Arduino (which is deprecated) the PSRAM cannot be used as it is not possible to customise the PSRAM pins for the Arduino framework build.

### 1. ESPHome Configuration
To use this backend, point your ESPHome YAML to this repository:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/TheMysticle/esphome-nspanel-lovelace-native
      ref: main
    refresh: 1h
    components: [nspanel_lovelace]
```

### 2. Update the Display (TFT/HMI)
This project **requires** the modified HMI firmware provided in this repo to display the overhauled screensaver. The files are located in the `/HMI` folder.
- **TFT File:** Upload the `.tft` file to your panel using the `upload_tft` service exposed by the device in Home Assistant.
- **HMI File:** Use the `.hmi` source file in the Nextion Editor if you wish to further customize the layout or icons.

The firmware version reported by the screen needs to be `52` or `53` for it to be compatible with this project.

---

## ⚙️ Configuration & Backend Setup

A basic configuration can be found in the `basic-example.yaml`, but you'll probably also want to look at the `advanced-example.yaml`.

### Adding Entities to C++ Setup
To ensure your dashboard buttons and weather update correctly, you must tell the ESP32 to track those specific entities. Open `components/nspanel_lovelace/nspanel_lovelace.cpp` and update the `setup()` function:

```cpp
void NSPanelLovelace::setup() {
  // Track these specifically for the Screensaver Dashboard sync
  this->create_entity("switch.ns_panel_left_relay");
  this->create_entity("switch.ns_panel_right_relay");
  this->create_entity("climate.living_room_thermostat");
  
  // Standard subscriptions...
}
```

### YAML Sync Triggers
Add triggers to your ESPHome YAML to keep the Nextion UI updated in real-time when states change in Home Assistant:

```yaml
binary_sensor:
  - platform: homeassistant
    entity_id: switch.ns_panel_left_relay
    id: relay_sync
    on_state:
      then:
        - lambda: 'id(nspanel_comp).send_display_command(id(relay_sync).state ? "bt0Val~1" : "bt0Val~0");'
```

### Icons & Colours

- **Icons:** Icon values can be an icon name or hex value (e.g. `hex:E549`). A list of icons can be found here: https://docs.nspanel.pky.eu/icon-cheatsheet.html
- **Colours:** Colours need to be a 16-bit number (0-65535) representing an `rgb565` colour code. You can use [this tool](https://chrishewett.com/blog/true-rgb565-colour-picker/) to select your colour.

---

## 🎨 Customizing the HMI

The dashboard is designed to be easily modifiable via the **Nextion Editor**:

1.  **Modify Labels:** Click the text boxes above the buttons (e.g., `tLR`, `tHeating`) and change the `txt` attribute to name your controls.
2.  **Swap Icons:** Select `bt0`, `bt1`, or `bt2` and change the `pic0` (Off) and `pic1` (On) attributes to use different images from the picture library.
3.  **Read-Only Mode:** To make a button display a state (like Heating active) without allowing manual toggles, add `tsw bt1,0` to the **Postinitialize** event of the screensaver page.
4.  **Weather Images:** The mapping between HA conditions and image IDs is located in `types.h`. You can replace the images in the Nextion library; simply ensure the ID numbers match your C++ code.

---

## Credits & License

This project is a fork based on the work of:
- [olicooper](https://github.com/olicooper/esphome-nspanel-lovelace-native) - Original native ESPHome backend logic.
- [joBr99](https://github.com/joBr99/nspanel-lovelace-ui) - Original Lovelace UI concepts.

Code in this repository is licensed under the GPLv3 license. Third-party code used in this project have their own license terms.