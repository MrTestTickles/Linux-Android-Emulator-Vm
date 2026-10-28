# Accelerated Alpine Linux VM on Apple Silicon using only the Android Emulator

Run a fast, hardware-accelerated **Alpine Linux (aarch64)** VM on an Apple Silicon Mac
without Homebrew, UTM, Lima, or any installer. The only thing needed is the Android
emulator that ships with the Android SDK, which is already Google-signed and allowed
to use Apple's Hypervisor.framework.

| Setup | sysbench CPU (events/s) | Cores | Kernel boot to root |
|---|---|---|---|
| Plain `-sysdir` QEMU launch (software emulation, Cortex-A57) | 161 (2 threads) | 2 | ~10 s |
| Booted through a real AVD (HVF, Apple core) | **26,361** (4 threads) | 4 | ~1.4 s |

## Why this exists

The Android emulator wraps its own `qemu-system-aarch64`. When you launch it with a bare
`-sysdir ... -qemu -M virt` command line, the launcher decides not to enable HVF and
prints `hvf is not enabled on this aarch64 host`. Then `-cpu host` fails with
`unable to find CPU model 'host'`, so everything runs in slow software emulation.

When the emulator is started with a **real AVD**, the launcher enables HVF and passes
the right CPU/machine options itself. So the trick is:

1. Create a real (tiny) Android AVD.
2. Launch that AVD, but override the kernel, initramfs, and disk so that it boots
   Alpine instead of Android.

`emulator -accel-check` confirms the Mac allows the hypervisor:

```
accel:
0
Hypervisor.Framework OS X Version 26.6
accel
```

This is an experiment/workaround, not a supported configuration.

## Requirements

- Apple Silicon Mac (tested: macOS 26.x, 8 GB RAM)
- Android SDK `emulator` (e.g. `~/Library/Android/sdk/emulator`)
- Android command-line tools (`sdkmanager`, `avdmanager`) and a Java 17+ runtime
- A prepared Alpine disk image and matching kernel/initramfs (see below)
- Files in `~/alpine`: `disk.img` (raw, ext4 root on `vda1`), `Image-disk`, `initramfs-disk`

## 1. Get the Android command-line tools

The zip filename contains a changing version number. Read it from Google's repository index:

```sh
mkdir -p ~/Library/Android/sdk/cmdline-tools && cd ~/Library/Android/sdk/cmdline-tools
FILE=$(curl -s https://dl.google.com/android/repository/repository2-3.xml | grep -o 'commandlinetools-mac-[0-9]*_latest.zip' | head -1)
echo "FILE=$FILE"
curl -LO "https://dl.google.com/android/repository/$FILE"
unzip -q "$FILE" && rm -rf latest && mv cmdline-tools latest && rm "$FILE"
ls latest/bin     # should list sdkmanager and avdmanager
```

`sdkmanager` needs Java. If `java -version` fails, point `JAVA_HOME` at a JDK you have:

```sh
export JAVA_HOME="/path/to/jdk/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
```

## 2. Create the throwaway AVD

```sh
export ANDROID_HOME=~/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/emulator
yes | sdkmanager --licenses
sdkmanager "system-images;android-34;default;arm64-v8a"
avdmanager create avd -n shell -k "system-images;android-34;default;arm64-v8a"
emulator -list-avds     # should print: shell
```

Check that the real AVD gets acceleration: run `emulator -avd shell -no-window` and confirm
the `hvf is not enabled` warning does **not** appear. Stop it with Ctrl-C.

## 3. Prepare the Alpine disk (one time)

The disk must boot by itself from the kernel/initramfs given on the command line. Things
that had to be true on the guest root filesystem:

- `/etc/fstab` root entry: `/dev/vda1 / ext4 defaults 0 1`
- Serial console getty in `/etc/inittab`: `ttyAMA0::respawn:/sbin/getty -L 115200 ttyAMA0 vt100`
- Networking on `eth0` via DHCP in `/etc/network/interfaces`
- OpenRC services enabled: `devfs dmesg mdev hwdrivers` (sysinit), `modules sysctl hostname bootmisc syslog networking` (boot), `sshd` (default)
- `PermitRootLogin yes` and `AllowTcpForwarding yes` in `/etc/ssh/sshd_config`
- Kernel modules `virtio_pci,virtio_blk,ext4` loaded by the initramfs (`modules=` on the kernel command line)

Find your root filesystem UUID inside the VM with `blkid /dev/vda1` and use it in step 4.
(The UUID is used instead of `/dev/vda1` because the AVD's own drives also appear as virtio
disks and can shift device names.)

## 4. Launch script: `~/bin/alpine-run-avd`

```sh
#!/bin/zsh
cd ~/alpine
exec ~/Library/Android/sdk/emulator/emulator -avd shell \
  -no-window -no-audio -no-snapshot -no-boot-anim -gpu swangle_indirect -show-kernel \
  -kernel Image-disk \
  -qemu -initrd initramfs-disk \
  -append "console=ttyAMA0 root=UUID=<YOUR-ROOT-UUID> rootfstype=ext4 modules=virtio_pci,virtio_blk,ext4" \
  -netdev user,id=n0,hostfwd=tcp::2222-:22 -device virtio-net-pci,netdev=n0 \
  -drive file=$HOME/alpine/disk.img,if=virtio,format=raw
```

```sh
chmod +x ~/bin/alpine-run-avd
alpine-run-avd
```

Notes:

- No `-cpu` or `-M` flags: the emulator sets those itself once HVF is on.
- Success looks like `Booting Linux on physical CPU 0x0 [0x610f0000]` (an Apple core), 4 CPUs,
  and `localhost login:` within a few seconds.
- The extra `vdb`–`vdf` drives and the `cat: can't open '/sys/class/virtio-ports/...'` lines
  belong to the Android AVD's virtual hardware and are harmless. Never mount them.
- Shut down cleanly with `poweroff` inside the VM.

SSH from the Mac:

```sh
ssh -p 2222 -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@localhost
```

## 5. Optional graphical desktop (XFCE over VNC)

Install once, inside the VM (as root):

```sh
apk add tigervnc xfce4 xfce4-terminal dbus dbus-x11 mousepad faenza-icon-theme font-dejavu
adduser -D me && echo 'me:changeme' | chpasswd
su - me -c 'mkdir -p ~/.vnc && echo changeme | vncpasswd -f > ~/.vnc/passwd && chmod 600 ~/.vnc/passwd'
rc-update add dbus default
```

Start script on the Mac, `~/bin/alpine-gui`:

```sh
#!/bin/zsh
SSH_OPTS=(-p 2222 -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no)

ssh $SSH_OPTS root@localhost 'sh -s' <<'REMOTE'
pkill xfce4-session; pkill xfwm4; pkill Xvnc
sleep 2
rm -f /tmp/.X1-lock /tmp/.X11-unix/X1
su - me -c 'setsid nohup Xvnc :1 -geometry 1280x720 -depth 24 -SecurityTypes VncAuth -PasswordFile /home/me/.vnc/passwd -localhost yes >/tmp/xvnc.log 2>&1 </dev/null &'
for i in 1 2 3 4 5 6 7 8 9 10; do [ -S /tmp/.X11-unix/X1 ] && break; sleep 1; done
sleep 1
su - me -c 'DISPLAY=:1 setsid nohup dbus-run-session startxfce4 >/tmp/xfce.log 2>&1 </dev/null &'
REMOTE

if ! lsof -i :5901 >/dev/null 2>&1; then
  ssh -f -N -L 5901:localhost:5901 $SSH_OPTS root@localhost
fi
open vnc://localhost:5901
```

Run it **once** and wait about 15 seconds. VNC password: `changeme`. Xvnc and XFCE do not
survive a VM reboot, so run `alpine-gui` after each boot. Do not run it twice in quick
succession: each run starts with `pkill`, which kills the previous attempt mid-startup.

If the screen is black, check `tail -20 /tmp/xfce.log` in the VM. Make sure no stale tunnel
holds port 5901 (`lsof -i :5901`, then `kill <PID>`).

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `hvf is not enabled on this aarch64 host` | Launched without a real AVD. Use `-avd shell`. |
| `unable to find CPU model 'host'` | Same: HVF not active. Do not pass `-cpu host` yourself. |
| `unknown option: -accel-check` | Run the `emulator` launcher, not the `qemu-android` binary. |
| `sdkmanager: command not found` | Command-line tools missing, or PATH not set (section 1/2). |
| Kernel panic, can't find root | Wrong UUID. Check `blkid`, and use `root=UUID=...`. |
| `administratively prohibited` on the SSH tunnel | Add `AllowTcpForwarding yes` to `sshd_config` in the VM, restart sshd. |
| `Address already in use` on 5901 | An old tunnel is running. It may be usable; otherwise `kill` it. |
| `Fatal server error: Server is already active for display 1` | Old Xvnc still running. `pkill Xvnc; rm -f /tmp/.X1-lock /tmp/.X11-unix/X1`. |
| Android Studio / Santa blocks Lima, Miniforge, etc. | Not a problem here: only Google's signed emulator is used. |

## Security notes

- The default credentials (`root` / `alpine`, `me` / `changeme`, VNC `changeme`) are for a
  local, forward-only-to-localhost lab VM. Change them for anything else.
- SSH root login is enabled and host key checking is disabled in the helper commands.
  That is acceptable for a throwaway VM on `localhost`; do not copy it to real servers.
- Xvnc binds to the VM's localhost only and is reached through an SSH tunnel.

## Status and limits

- Verified: accelerated boot, 4 cores, sysbench result above, SSH on port 2222.
- Unsupported hack: a future emulator update may change how it handles overridden kernels.
