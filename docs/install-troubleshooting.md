# Installation troubleshooting

## ModemManager claims the modem

ModemManager can access USB serial ports and QMI devices while HiDeck is using them. Symptoms may include a busy device, failed modem initialization, or unreliable AT/QMI communication. An active service is a potential conflict, not proof that it owns every modem.

Check the service and its recent activity:

```sh
systemctl status ModemManager.service --no-pager
journalctl -u ModemManager.service -n 50 --no-pager
```

On a dedicated HiDeck VM whose internet connection does not depend on ModemManager, download and run the installer with the explicit option:

```sh
curl -fsSL https://raw.githubusercontent.com/0xayanami/hid/main/deploy.sh -o /tmp/hid-deploy.sh
sudo env HIDECK_DIR=/opt/hideck HIDECK_DISABLE_MODEMMANAGER=1 sh /tmp/hid-deploy.sh
```

This stops and masks ModemManager for all devices on this host. Masking also prevents automatic service activation. No MBN, IMS, USB mode, or database settings are changed by this check. The option defaults to off; the installer stops with instructions when the service is active or enabled.

For an existing installation, stopping the conflicting service does not require rebuilding HiDeck:

```sh
sudo systemctl mask --now ModemManager.service
```

Confirm the service is inactive and masked:

```sh
systemctl is-active ModemManager.service
systemctl is-enabled ModemManager.service
```

Expected output is `inactive` and `masked`; these inspection commands may return a nonzero status.

To restore ModemManager later:

```sh
sudo systemctl unmask ModemManager.service
sudo systemctl enable --now ModemManager.service
```

Restore the previous service enablement state if it was not originally enabled.

## A device remains busy

Check the actual device paths reported by HiDeck. Install the diagnostic tool and inspect owners (replace the sample paths with your devices):

```sh
sudo apt install -y psmisc
sudo fuser -v /dev/ttyUSB2 /dev/cdc-wdm0
```

The HiDeck process itself may legitimately own these devices. Do not blindly kill every reported process. Look for another modem service, a serial terminal, a QMI command still running, or another container using the same modem. Close the competing application and retry.

A stopped ModemManager does not fix missing USB passthrough: verify `lsusb` inside the VM. Only one VM should own the USB modem. For missing audio devices, check `aplay -l`, `arecord -l`, and the host `snd_usb_audio` driver.

## Docker build errors

The source Dockerfile requires Buildx/BuildKit. `COPY --chmod` or `RUN --mount` errors from the legacy builder mean the Buildx plugin is missing. Install `docker-buildx-plugin` and verify `sudo docker buildx version`.

The first build downloads base images, npm packages, and Go modules. It needs internet access; do not use `--network=none` for this source build. The earlier offline binary-overlay build was a different deployment method.

## Browser calling

Use trusted HTTPS for microphone access. Allow the microphone and add the actual HiDeck host to your privacy extension's whitelist. Disable competing global WebRTC blockers. WebRTC media also needs UDP connectivity; an accessible web page alone does not verify the audio path.
