# FFmpeg Screen Recorder

Lightweight screen recorder using FFmpeg with high-quality H.264 encoding, system audio capture, and optional microphone input.

## Features

- **High quality** — CRF 18 (near-lossless) with `libx264`
- **System audio** — records audio playing from your computer
- **Microphone** — optional mic input mixed with system audio
- **Volume control** — adjust system and mic levels per recording
- **No bloat** — single bash script, no GUI dependencies

## Requirements

- `ffmpeg` (with `libx264` and `pulse` support)
- `pulseaudio` or `pipewire-pulse`
- X11 display

Install ffmpeg:

```bash
sudo apt install ffmpeg
```

## Installation

```bash
# Clone or download, then:
chmod +x record
sudo cp record /usr/local/bin/   # system-wide
# OR
cp record ~/.local/bin/          # user-only (ensure it's in PATH)
```

## Usage

```bash
record                          # system audio + mic (default volumes)
record --no-mic                 # system audio only
record --sys-vol 2              # system audio at 2x, mic at 1.5x
record --sys-vol 2 --mic-vol 1  # system at 2x, mic at 1x
record --help                   # show options
```

**Stop recording:** press `q` in the terminal.

**Output:** saved to `~/Videos/screen-YYYYMMDD-HHMMSS.mkv`

## Volume Reference

| Flag | Default | Range | Description |
|------|---------|-------|-------------|
| `--sys-vol` | 3.5 | 0.0–10.0 | System audio volume multiplier |
| `--mic-vol` | 1.5 | 0.0–10.0 | Microphone volume multiplier |

- `1.0` = original volume
- `2.0` = 2x louder
- `0.5` = half volume
- `0.0` = muted

## Live Adjustments (during recording)

Use PulseAudio controls while recording:

```bash
pavucontrol                 # GUI — adjust per-source sliders
pactl set-source-volume 57 0.5   # system audio to 50%
pactl set-source-volume 58 0.5   # mic to 50%
```

Find your source indices:

```bash
pactl list short sources
```

## Customize

Edit the script to change defaults:

```bash
# In record, change these lines:
sys_vol="3.5"
mic_vol="1.5"
```

For **hardware encoding** (Intel QSV), replace `libx264` with `h264_qsv`:

```bash
-c:v h264_qsv -global_quality 18
```

## Script (`record`)

```bash
#!/bin/bash

sys_audio="alsa_output.pci-0000_00_1b.0.analog-stereo.monitor"
mic_audio="alsa_input.pci-0000_00_1b.0.analog-stereo"

sys_vol="3.5"
mic_vol="1.5"

while [[ $# -gt 0 ]]; do
  case "$1" in
    --no-mic) no_mic=1 ;;
    --sys-vol) sys_vol="$2"; shift ;;
    --mic-vol) mic_vol="$2"; shift ;;
    --help|-h) echo "Usage: record [--no-mic] [--sys-vol N] [--mic-vol N]"; exit 0 ;;
    *) echo "Unknown: $1 (use --help)"; exit 1 ;;
  esac
  shift
done

out="$HOME/Videos/screen-$(date +%Y%m%d-%H%M%S).mkv"

if [ "$no_mic" = "1" ]; then
  ffmpeg -video_size 1366x768 -framerate 30 -f x11grab -i :0.0 \
         -f pulse -i "$sys_audio" \
         -af "volume=$sys_vol" \
         -c:v libx264 -crf 18 -preset fast \
         -c:a aac -b:a 192k \
         "$out"
else
  ffmpeg -video_size 1366x768 -framerate 30 -f x11grab -i :0.0 \
         -f pulse -i "$sys_audio" \
         -f pulse -i "$mic_audio" \
         -filter_complex "[1:a]volume=$sys_vol[sys];[2:a]volume=$mic_vol[mic];[sys][mic]amix=inputs=2:duration=first:dropout_transition=2[a]" \
         -map 0:v -map "[a]" \
         -c:v libx264 -crf 18 -preset fast \
         -c:a aac -b:a 192k \
         "$out"
fi

echo "Saved: $out"
```
