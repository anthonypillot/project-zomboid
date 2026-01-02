# Install and run a Project Zomboid Dedicated Server

This document summarizes how to download the dedicated server files, run the server on Linux, and add it as a systemd service (with a control FIFO socket) so you can safely start/stop it.

## Prerequisites (Debian/Ubuntu)
- Install SteamCMD and required architectures:

```bash
sudo add-apt-repository multiverse
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install steamcmd -y
```

- Create a user to run the server (do not run as root):

```bash
sudo adduser --disabled-login --gecos "" pzuser
sudo mkdir -p /opt/pzserver
sudo chown pzuser:pzuser /opt/pzserver
```

## Downloading server files (SteamCMD)
Log in as the `pzuser` account and use a SteamCMD script to install/update the server into `/opt/pzserver`.

Create a script file, for example `$HOME/update_zomboid.txt` with the following contents:

```
@ShutdownOnFailedCommand 1
@NoPromptForPassword 1
force_install_dir /opt/pzserver/
login anonymous
app_update 380870 validate
quit
```

Notes:
- To install the unstable/beta build, append `-beta unstable` to the `app_update` line: `app_update 380870 -beta unstable validate`.
- To install a specific version, append `-beta 42.13.1` to the `app_update` line: `app_update 380870 -beta 42.13.1 validate`.

Run the script with SteamCMD (still as `pzuser`):

```bash
export PATH=$PATH:/usr/games
steamcmd +runscript $HOME/update_zomboid.txt
```

## Open required ports
Project Zomboid uses UDP ports; open them with UFW (example):

```bash
sudo ufw allow 16261/udp
sudo ufw allow 16262/udp
sudo ufw reload
```

If running multiple servers, each instance needs its own pair of UDP ports.

## Run the server (manual)
- After installing the server, run it at least once manually (it will ask you to set an admin password on first run):

```bash
cd /opt/pzserver
bash start-server.sh
# for non-Steam (e.g., GOG) use:
# bash start-server.sh -nosteam
# to select a server name (uses different .ini / save folder):
# bash start-server.sh -servername YOURSERVERNAME
```

The `start-server.sh` script launches the Java server. If you need to pass memory options or other JVM flags, edit the server start script accordingly.

## Systemd service (safe shutdown via control FIFO)
You can configure a systemd unit + socket so the server runs as `pzuser` and accepts control commands via a FIFO. The approach lets systemd send a safe save/quit sequence on stop.

1. Start the server manually once to create admin password and initial files, then exit the `pzuser` session.
2. Create a systemd service unit `/etc/systemd/system/zomboid.service` (example):

```ini
[Unit]
Description=Project Zomboid Server
After=network.target

[Service]
PrivateTmp=true
Type=simple
User=pzuser
WorkingDirectory=/opt/pzserver/
ExecStart=/bin/sh -c "exec /opt/pzserver/start-server.sh </opt/pzserver/zomboid.control"
ExecStop=/bin/sh -c "echo save > /opt/pzserver/zomboid.control; sleep 15; echo quit > /opt/pzserver/zomboid.control"
Sockets=zomboid.socket
KillSignal=SIGCONT

[Install]
WantedBy=multi-user.target
```

3. Create the matching socket unit `/etc/systemd/system/zomboid.socket` (example):

```ini
[Unit]
BindsTo=zomboid.service

[Socket]
ListenFIFO=/opt/pzserver/zomboid.control
FileDescriptorName=control
RemoveOnStop=true
SocketMode=0660
SocketUser=pzuser

[Install]
WantedBy=sockets.target
```

4. Reload systemd and enable the socket (run as root):

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now zomboid.socket
```

5. Control the server via systemctl (the socket listens and systemd will spawn the service on demand):

```bash
# start
sudo systemctl start zomboid.socket
# stop
sudo systemctl stop zomboid
# restart
sudo systemctl restart zomboid
# show journal logs
journalctl -u zomboid -f
```

To send a command to the running server (for example `help`), write to the FIFO:

```bash
echo "help" | sudo tee /opt/pzserver/zomboid.control
```

## Notes & troubleshooting
- Do not run the server as root.
- If you installed using Steam (GUI) rather than SteamCMD, the default install folder is typically under `~/.steam/steam/steamapps/common/Project Zomboid Dedicated Server` or `"/Program Files (x86)/Steam/..."` on Windows.
- If the server fails to start due to memory, edit the `start-server` script to adjust `-Xms`/`-Xmx` JVM flags.
- The server generates `servertest.ini`, `servertest_SandboxVars.lua`, and save folders under `~/Zomboid` (for `pzuser`); edit those files to change game settings.

## Useful commands recap

```bash
# update server (as pzuser)
steamcmd +runscript $HOME/update_zomboid.txt

# start server manually
cd /opt/pzserver && bash start-server.sh

# systemd control
sudo systemctl start zomboid.socket
sudo systemctl stop zomboid
sudo systemctl restart zomboid
sudo journalctl -u zomboid -f

# send command to server control FIFO
echo "save" | sudo tee /opt/pzserver/zomboid.control
```

---
This file is a concise summary adapted from the Project Zomboid wiki (Dedicated_server). For more advanced configuration (multiple servers on one host, workshop mods, custom server names), see the official wiki: https://pzwiki.net/wiki/Dedicated_server
