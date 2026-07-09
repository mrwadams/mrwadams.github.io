---
layout: post
title: "Building a fully local voice assistant on an ESP32-S3 audio board"
date: 2026-07-09
categories: [ai, esp32]
tags: [llm, voice-assistant, xiaozhi, lm-studio, self-hosting, esp32-s3, wake-word]
---

![The Waveshare ESP32-S3-AUDIO-Board](/assets/images/esp32-s3-audio-board.jpg)
*The Waveshare ESP32-S3-AUDIO-Board. Photo: [Waveshare](https://www.waveshare.com/esp32-s3-audio-board.htm).*

I picked up a [Waveshare ESP32-S3-AUDIO-Board](https://docs.waveshare.com/ESP32-S3-AUDIO-Board) to have a play with. It is built for voice work: an ES8311 codec, an ES7210 ADC feeding a dual microphone array with hardware echo cancellation, a speaker amplifier, and a ring of addressable RGB LEDs, all driven by an ESP32-S3 with 8MB of PSRAM.

I wanted to see if I could turn it into a voice assistant that runs entirely on my own hardware. No cloud service, no API keys, nothing leaving the house. A microphone in a room that only ever talks to a box on my own network is a much easier thing to be comfortable with than the alternative. This is a write-up of how I got there, including the bits that caught me out, so I can repeat it later.

## What I was aiming for

A board that wakes on a wake word, listens, and answers out loud, with every stage of the pipeline running locally: speech-to-text, the language model, and text-to-speech. The board should only ever talk to a server on my own LAN.

The assistant firmware is [Xiaozhi](https://github.com/78/xiaozhi-esp32), an open-source ESP32 voice assistant with a large following. Waveshare maintain a prebuilt build of it for this exact board. It captures audio, runs wake-word detection and echo cancellation on the device itself, and streams the rest to a server. The server is where you decide how private the setup is, and I ran the whole chain locally.

<pre class="mermaid">
graph LR
    A["ESP32-S3 board\nmic array + AEC + wake word"] -->|Wi-Fi / WebSocket| B
    B["xiaozhi-esp32-server\n(Docker, on my Mac)"] --> C["SenseVoice\nspeech-to-text"]
    B --> D["LM Studio\nlocal LLM (Gemma)"]
    B --> E["Kokoro\ntext-to-speech"]
    style A fill:#2d333b,stroke:#768390,color:#adbac7
    style B fill:#2d333b,stroke:#768390,color:#adbac7
    style C fill:#2d333b,stroke:#768390,color:#adbac7
    style D fill:#2d333b,stroke:#768390,color:#adbac7
    style E fill:#2d333b,stroke:#768390,color:#adbac7
</pre>

Four pieces, all on my own machine: [`xiaozhi-esp32-server`](https://github.com/xinnan-tech/xiaozhi-esp32-server) running in Docker to tie it together, a Gemma model served by [LM Studio](https://lmstudio.ai/) over its OpenAI-compatible API, SenseVoice doing speech-to-text inside the server, and [Kokoro](https://github.com/remsky/Kokoro-FastAPI) for a small, local text-to-speech voice.

## Standing up the server

The server has a simplified "server only" deployment: one Docker container plus a config file, with no database or web UI. I added a second container for Kokoro so the whole thing comes up together, and pointed the server at LM Studio for the language model.

The config override (`data/.config.yaml`) is where you pick each module and wire it to the local services:

```yaml
selected_module:
  VAD: SileroVAD
  ASR: FunASR          # local speech-to-text (SenseVoice)
  LLM: LMStudioLLM     # -> LM Studio on the host
  TTS: OpenAITTS       # -> Kokoro, repointed below
  Intent: function_call
  Memory: nomem

LLM:
  LMStudioLLM:
    type: openai
    base_url: http://192.168.1.50:1234/v1   # your Mac's LAN IP
    model_name: your-local-model
    api_key: lm-studio

TTS:
  OpenAITTS:
    type: openai
    api_url: http://kokoro-tts:8880/v1/audio/speech
    model: kokoro
    voice: af_bella
    language: English
```

Speech-to-text runs locally with SenseVoice, so there is a one-off model download of around 900MB, mounted into the container.

## Flashing and pointing it at my own server

The board arrived running a factory demo. Flashing the Xiaozhi firmware is a single merged binary written to offset `0x0` with [esptool](https://github.com/espressif/esptool):

```bash
esptool.py --chip esp32s3 -p /dev/cu.usbmodem2101 erase_flash
esptool.py --chip esp32s3 -p /dev/cu.usbmodem2101 -b 921600 \
  write_flash 0x0 merged-binary.bin
```

By default the firmware talks to the Xiaozhi project's public cloud. The step that keeps everything local is the **Custom OTA URL** field in the setup portal, which redirects the board to your own server without rebuilding anything. On first boot the board comes up as its own Wi-Fi access point:

1. Join the board's `Xiaozhi-XXXX` hotspot from a phone.
2. Open `http://192.168.4.1`.
3. On the **Advanced** tab, set the Custom OTA URL to your server: `http://192.168.1.50:8003/xiaozhi/ota/`.
4. On the main tab, enter your home Wi-Fi and save.

The board reboots, joins the network, fetches its WebSocket address from your server, and from then on talks only to your machine.

## The things that caught me out

Most of my time went on three problems, none of them obvious.

The first was that Docker could not reach LM Studio. LM Studio binds to `127.0.0.1` by default, so the server container could not see it, even using `host.docker.internal`. On macOS, Docker Desktop runs inside a VM, so "localhost" from a container is not the host machine. The fix was to bind LM Studio to the LAN and point the config at the Mac's real address. That does expose the unauthenticated model server to the whole network, so it is only sensible on a network you trust.

The second was stranger: my first model choice returned nothing out loud. It was a reasoning model, and it burned its whole token budget thinking before it ever reached an answer. For a voice assistant you want something that replies directly, or a much bigger budget to think in. A plain instruction-tuned model was quicker and actually spoke.

The last one had me puzzled for a while. It kept replying in Chinese. Xiaozhi's prompt template carries a firm "reply in this language" instruction, and the language is read from the TTS config, which defaults to Chinese. My English speech voice then tried to pronounce Chinese characters and produced pure noise. The fix was one line, `language: English` on the TTS block. Setting the persona prompt on its own did nothing.

With those out of the way the round trip worked: say the wake word, ask a question in English, and hear a spoken answer a couple of seconds later, all from my own hardware.

## Changing the wake word

The default wake word is a Chinese phrase, and I wanted "Jarvis". This is where it gets a little more involved. The wake word is detected on the board by Espressif's WakeNet, so you cannot use an arbitrary phrase; you choose one of Espressif's pre-trained models and it is compiled into the firmware. The available English options include Jarvis, Alexa, Computer and Hi ESP.

The board's source is in the upstream Xiaozhi repository, so this meant building the firmware rather than flashing the prebuilt binary. Two things worth knowing:

- The wake word is a build option. I swapped `CONFIG_SR_WN_WN9_NIHAOXIAOZHI_TTS` for `CONFIG_SR_WN_WN9_JARVIS_TTS` in the ESP32-S3 defaults.
- The current firmware needs ESP-IDF 5.5.2 or newer to compile. The 5.4 I already had was fine for flashing but would not build it.

To avoid redoing the Wi-Fi setup, I flashed the individual partitions rather than doing a full-chip erase. That leaves the NVS partition, which holds the Wi-Fi and server settings, untouched:

```bash
esptool.py --chip esp32s3 -p /dev/cu.usbmodem2101 -b 921600 write_flash \
  0x0      build/bootloader/bootloader.bin \
  0x8000   build/partition_table/partition-table.bin \
  0xd000   build/ota_data_initial.bin \
  0x20000  build/xiaozhi.bin \
  0x800000 build/generated_assets.bin
```

The board rebooted, reconnected to my server on its own, and now answers to "Jarvis".

## Where it's at

The result is a small, self-contained voice assistant: a board on the shelf that wakes on "Jarvis", understands English, and answers out loud, with every part of the pipeline running on a machine I control. All the pieces are open source, and the only thing on the network is a server I can see.

There is plenty left to tinker with. The obvious next steps are swapping the voice, giving it memory between conversations, and wiring up the on-board screen and camera. But as a first outing, having a private assistant answer back in its own voice was a good place to pause.
