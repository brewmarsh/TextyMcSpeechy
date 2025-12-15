# Setting up Audio in WSL2

The error `arecord: device_list:277: no soundcards found` occurs because WSL2 is a Virtual Machine. By default, it is isolated from your host hardware and does not "see" your physical sound card or microphone.

Here are the two ways to solve this. **Method 1** is generally preferred for standard audio, while **Method 2** is required if you specifically need the device to show up as a hardware device (e.g., for raw hardware access).

-----

### Method 1: The PulseAudio Network Bridge (Recommended)

Instead of trying to access the hardware directly, you stream the audio from Windows into WSL2 over a virtual network connection.

**Note:** With this method, `arecord -l` (list hardware) will **still be empty**. This is expected. You will record via the `pulse` virtual device instead.

#### 1. Prepare Windows

1.  Download **PulseAudio for Windows** (e.g., from the [PulseAudio on Windows project](https://www.freedesktop.org/wiki/Software/PulseAudio/Ports/Windows/Support/)).
2.  Unzip it to a folder (e.g., `C:\PulseAudio`).
3.  Edit `etc\pulse\default.pa` inside that folder:
      * Find the line `load-module module-native-protocol-tcp`
      * Change it to:
        ```text
        load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1;172.16.0.0/12
        ```
        *(This allows connections from localhost and the WSL2 subnet range).*
4.  Run `bin\pulseaudio.exe` (keep this window open to test).

#### 2. Configure WSL2

1.  Install the ALSA-to-Pulse plugins:
    ```bash
    sudo apt update && sudo apt install pulseaudio-utils libasound2-plugins
    ```
2.  Tell WSL where the Windows audio server is. Run this command (and add it to your `~/.bashrc` to make it permanent):
    ```bash
    export PULSE_SERVER=tcp:$(grep nameserver /etc/resolv.conf | awk '{print $2}')
    ```

#### 3. Test Recording

Do not use `arecord -l`. Instead, try recording by explicitly specifying the pulse device:

```bash
arecord -D pulse -f cd test.wav
```

If this creates a file and you can play it back (or copy it to Windows to play), the bridge is working.

-----

### Method 2: USB Passthrough (usbipd-win)

If you are using a **USB Microphone** and your software *requires* a hardware device handle (i.e., you *must* see it in `arecord -l`), you can attach the USB device directly to the WSL2 kernel.

1.  **On Windows:** Install `usbipd-win` using PowerShell (Admin):
    ```powershell
    winget install usbipd
    ```
2.  **On Windows:** List your devices to find the Bus ID of your microphone:
    ```powershell
    usbipd list
    ```
3.  **On Windows:** Bind the device (replace `<BUSID>` with your device's ID, e.g., `1-1`):
    ```powershell
    usbipd bind --busid <BUSID>
    ```
4.  **On Windows:** Attach the device to WSL:
    ```powershell
    usbipd attach --wsl --busid <BUSID>
    ```
5.  **On WSL2:** Check for the device. It should now appear:
    ```bash
    lsusb
    arecord -l
    ```

### Troubleshooting

  * **Firewall:** If Method 1 fails, ensure the Windows Firewall is not blocking `pulseaudio.exe` on "Public" networks (WSL2 network adapters sometimes classify as Public).
  * **WSLg:** If you are on Windows 11, WSLg attempts to handle this automatically, but often defaults to *output only*. Method 1 usually overrides WSLg quirks reliably for input.
