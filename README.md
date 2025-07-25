# 🧠 Local AI Voice Assistant Stack (GPU-Accelerated)

> ⚠️ **Note:** This stack is intended to be run in a **containerized Docker environment**.  
> If you are installing Home Assistant or its components outside Docker (e.g., via Home Assistant OS, Supervised, or Python virtualenv),  
> please refer to the official documentation instead: [Getting Started with Home Assistant](https://www.home-assistant.io/getting-started/)

This repo contains a self-hosted voice assistant stack that runs entirely on your machine — no cloud required, no subscriptions, and no surveillance.

It uses Docker Compose to orchestrate services like Home Assistant, Whisper for speech recognition, Piper for text-to-speech, and OpenWakeWord for wake word detection. Each service is GPU-accelerated for fast, low-latency voice interaction.

---

## ⚙️ What's Inside

| Service         | Purpose                             | GPU Support | Notes |
|-----------------|-------------------------------------|-------------|-------|
| `homeassistant` | Smart home brain                    | ❌          | Controls devices and automations |
| `whisper`       | Speech-to-text                      | ✅          | Whisper model w/ CUDA support |
| `piper`         | Text-to-speech                      | ✅          | CUDA-accelerated voice synthesis |
| `openwakeword`  | Wake word detection                 | ✅          | Useful for ESP32/Atom devices |

---


## 🧰 Folder Structure

```text
.
├── docker-compose.yml         # Core stack config
├── homeassistant_config/      # HA configuration volume
├── whisper-data/              # Whisper model data
├── piper-data/                # Piper voice data
└── wakeword-data/             # Custom wake word files
```

Note: 

> ℹ️ Docker will auto-create the volume folders (like `./whisper-data`, `./piper-data`, etc.) if they don’t exist.  
> You don’t need to manually create them — they’ll show up after your first `docker compose up`.

---

## 🚀 Quick Start

> Make sure you have Docker and NVIDIA GPU support set up (via `nvidia-container-toolkit`).

1. Clone this repo:
   ```bash
   git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
   cd YOUR_REPO
   ```

2. Start everything:
   ```bash
   docker compose up -d
   ```

3. Access Home Assistant:
   - Navigate to [http://localhost:8123](http://localhost:8123) in your browser.

---


## 🖥️ GPU Acceleration

Each AI service uses NVIDIA GPU acceleration via the following Compose directive:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: ["gpu", "utility", "compute"]
```

This speeds up Whisper, Piper, and OpenWakeWord drastically compared to CPU-based inference. This ensures the entire stack hits your GPU.

> 💡 Make sure your NVIDIA drivers and `nvidia-docker` runtime are properly installed.

---

## 🔵 Bluetooth Support (Linux Only)

```yaml
devices:
  - "/dev/hci0"
```

This line in the `homeassistant` container enables Bluetooth support — useful for presence detection, BLE sensors, or Matter-over-Bluetooth. It only works on Linux hosts.

> 🪟 If you're on **Windows** or **macOS**, you can safely remove this line.

---

## 🌐 Network Mode: Host

```yaml
network_mode: host
```

Home Assistant uses host networking to enable local discovery (e.g., mDNS, SSDP, uPnP) for devices like Google Home, Chromecast, and other smart devices.

> ⚠️ **This setting is only relevant if you're running Home Assistant inside Docker.**  
> If you're running Home Assistant natively (e.g., via Home Assistant OS, Supervised, or in a VM), this is handled automatically — you don’t need to configure network mode manually.

> ⚠️ **Docker limitation on Windows/macOS:**  
> `network_mode: host` only works properly on **Linux**.  
> On other platforms, local device discovery may not work unless you use bridged networking, WSL2, or run HA in a Linux-based VM.

---

## 🔔 Wake Word Support (OpenWakeWord vs Voice PE)

This stack includes [`openwakeword`](https://github.com/davidchatting/openWakeWord) to support **custom wake words**, using `.tflite` models trained or downloaded locally.

### 🧠 When to Use It

- ✅ You're using **ESP32‑S3 devices** like:
  - M5Stack ATOM Echo
  - ESP32‑S3‑BOX‑3
- ✅ You want to stream audio to HA and use **custom wake words**

### 🚫 When NOT to Use It

- ❌ You're using a **Home Assistant Voice Preview Edition (Voice PE)** unit  
  > Voice PE uses the **`microWakeWord` engine**, which runs entirely on-device and **only supports built-in wake words** like:
  > - "Okay Nabu"
  > - "Hey Jarvis"
  > - "Hey Mycroft"

Including `openwakeword` future-proofs your setup for advanced use cases, but it's not currently used by Voice PE. Hopefully official support for openwakeword comes at some point in the future to HA Voice Preview.

---

## 🧼 Customization & Notes

- You can change the Whisper model size (`tiny`, `small`, `medium`) via the `MODEL=` environment variable. The default model I have set up is perfectly capable and ensures minimal VRAM hit but feel free to try different ones.
- Piper supports multiple voice presets — I set a default one for the `--voice` argument you can change this within the HA GUI itself.
- Feel free to remove services you don’t need (e.g., `openwakeword`) by commenting them out in `docker-compose.yml`.

---

## 📄 License

MIT License for code. Any media/design content is CC-BY 4.0 unless noted otherwise.

---

## 🗣️ Credits / Acknowledgements

- [Home Assistant](https://www.home-assistant.io/)
- Special thanks to the Home Assistant Voice team for making local voice a reality.
