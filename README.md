# Tutorial with Mycroft Mark II

This tutorial should work for almost any Raspberry Pi and USB microphone. Audio enhancements and local wake word detection may require a 64-bit operating system, however.
Use your Mycroft Mark II (or Dev Kit) as Wyoming Satellite for Home Assistant.

**This is a draft that might be merged into the [Wyoming Protocol repo](https://github.com/rhasspy/wyoming-satellite/tree/master/docs).**

---

Create a voice satellite using a Mycroft Mark II (or Neon OVOS Mycroft AI Mark II).

(You can still order a Mark II at the [Neon AI shop](https://neon.ai/PurchaseHardware)https://neon.ai/PurchaseHardware)

This tutorial should work for the [Mark II](https://mycroft-ai.gitbook.io/mark-ii/) and the [Dev Kit](https://mycroft-ai.gitbook.io/docs/using-mycroft-ai/get-mycroft/mark-ii/mark-ii-dev-kit).

## Install OS

TPD

## Install Software

After flashing and booting the satellite, connect to it over SSH using the username/password you configured during flashing.

**On the satellite**, make sure system dependencies are installed:

```sh
sudo apt-get update
sudo apt-get install --no-install-recommends  \
  git \
  python3-venv
```

Clone the `wyoming-satellite` repository:

```sh
git clone https://github.com/rhasspy/wyoming-satellite.git
```

Change to the `wyoming-satellite` folder and install the dependencies:

```sh
cd wyoming-satellite/
python3 -m venv .venv
.venv/bin/pip3 install --upgrade pip
.venv/bin/pip3 install --upgrade wheel setuptools
.venv/bin/pip3 install \
  -f 'https://synesthesiam.github.io/prebuilt-apps/' \
  -r requirements.txt \
  -r requirements_audio_enhancement.txt \
  -r requirements_vad.txt
```

If the installation was successful, you should be able to run:

```sh
script/run --help
```

## Determine Audio Devices

Picking the correct microphone/speaker devices is critical for the satellite to work. We'll do a test recording and playback in this section.

List your available microphones with:

```sh
arecord -L
```

If you have the ReSpeaker 2Mic HAT, you should see:

```
plughw:CARD=sndrpisimplecar,DEV=0
    snd_rpi_simple_card, simple-card_codec_link snd-soc-dummy-dai-0
    Hardware device with all software conversions
```

Just use the `default` settings for the Mark II.

Record a 5 second sample from your chosen microphone:

```sh
arecord -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav
```

Say something while `arecord` is running. If you get errors, try a different microphone device by changing `-D <device>`.

List your available speakers with:

```sh
aplay -L
```

You should see:

```
plughw:CARD=sndrpisimplecar,DEV=0
    snd_rpi_simple_card, simple-card_codec_link snd-soc-dummy-dai-0
    Hardware device with all software conversions
```

For other speakers, prefer ones that start with `plughw:` or just use `default` if you don't know what to use.

Play back your recorded sample WAV:

```sh
aplay test.wav
```

You should hear your recorded sample. If there are problems, try a different speaker device by changing `-D <device>`.

Make note of your microphone and speaker devices for the next step.

## Running the Satellite

In the `wyoming-satellite` directory, run:

```sh
script/run \
  --debug \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -r 22050 -c 1 -f S16_LE -t raw'
```

You can set `--name <NAME>` to whatever you want, but it should stay the same every time you run the satellite.

In Home Assistant, check the "Devices & services" section in Settings. After some time, you should see your satellite show up as "Discovered" (Wyoming Protocol). Click the "Configure" button and "Submit". Choose the area that your satellite is located, and click "Finish".

Your satellite should say "Streaming audio", and you can use the wake word of your preferred pipeline.

## Create Services

You can run wyoming-satellite as a systemd service by first creating a service file:

``` sh
sudo systemctl edit --force --full wyoming-satellite.service
```

Paste in the following template, and change both `/home/pi` and the `script/run` arguments to match your set up:

```text
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -r 22050 -c 1 -f S16_LE -t raw' \
WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Save the file and exit your editor. Next, enable the service to start at boot and run it:

``` sh
sudo systemctl enable --now wyoming-satellite.service
```

(you may need to hit CTRL+C to get back to a shell prompt)

With the service running, you can view logs in real-time with:

``` sh
journalctl -u wyoming-satellite.service -f
```

If needed, disable and stop the service with:

``` sh
sudo systemctl disable --now wyoming-satellite.service
