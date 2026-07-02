# Building and Flashing Firmware

## Option A: Build Locally

### Prerequisites
- Python 3.14+
- PlatformIO Core (`pip install platformio`)
- USB-C cable

### Build
```bash
# Clone the repo
git clone https://github.com/uxjulia/CrossInk.git
cd CrossInk

# Build the tiny variant (recommended for most users)
pio run -e tiny
```

The firmware binary is at `.pio/build/tiny/firmware-tiny.bin`.

### Flash
1. Connect your Xteink X4/X3 via USB-C
2. Run: `pio run -e tiny --target upload`
3. Wait for the upload to complete

## Option B: GitHub Actions (CI Build)

1. Push your changes to the `main` branch
2. Go to Actions tab → select the latest CI run
3. Download the `firmware-tiny` artifact
4. Flash using the [web installer](https://crosspoint-reader.github.io/crosspoint-reader/flash) or `esptool.py`

## Option C: Release Build (workflow_dispatch)

1. Go to Actions → "Compile Release"
2. Click "Run workflow" → enter version (e.g. `1.4.0-rc1`)
3. Wait for build to complete
4. Download the firmware binary from the draft release
