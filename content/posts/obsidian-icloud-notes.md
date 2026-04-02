---
title: "Shared folder from iphone to linux laptop"
date: "2026-03-31"
summary: "Sharing notes between iphone (Obsidian app) - icloud folder - mac mini sync - network share - MX Linux (Obsidian app)"
# description: ""
tags: ["Home Lab", "Obsidian notes", "Second brain", "Network mounted folder"]
author: "Liviu Iancu"
weight: 2
series: ["Network"]
# categories: [""]
# aliases: [""]
# cover:
#   image: images/img.png
#   caption: "Description [text](https://link.somewhere/)"
---
This was all done just for knowledge and fun, but it could have some useful applications, depending on your sharing needs, apps on each system (mac, linux, windows), folder share between different os and hardware limitations you have.  

My use case: Use a icloud shared folder for Obsidian notes. Share it on LAN network. Because i'm mostly working on a linux laptop and icloud notes app is not available.  

**Data flow overview:**  
iphone Obsidian app - icloud folder - mac mini sync - network share - mx linux - Obsidian app

Steps required to setup and blockers encountered are detailed below. 

**Network folder share from Mac mini to Mx Linux Laptop**

Macos - place obsidian folder in icloud folder.
- Disable "Optimize Storage" (requires GUI):
System Settings > iCloud > iCloud Drive > Options > Uncheck "Optimize Mac Storage"
This forces full local sync instead of on-demand.
- Share the folder via System Settings > General > Sharing > File Sharing
Verify shared folder:
```bash
sharing -l
# Example output
List of Share Points
name:           Obsidian
path:           /Users/username/Library/Mobile Documents/iCloud~md~obsidian/Documents
        smb:    {
                name:   Obsidian
...
```

Use SSHFS instead of SMB to mount the shared folder on MX Linux.
On Mx Linux:
```bash
sudo apt install sshfs
```
Mount folder:
```bash
mkdir -p ~/icloud-obsidian
sshfs username@mini-ip:/Users/username/Library/Mobile\ Documents/iCloud~md~obsidian/Documents ~/icloud-obsidian
```

**Installing Obsidian on MX Linux using Appimage**  
*Did not use snap package because it's not recommended with MX Linux*

Download .AppImage file from obsidian
Then run command:
```bash
mkdir -p ~/.local/bin 
mv ~/Downloads/Obsidian-*.AppImage ~/.local/bin/obsidian 
chmod u+x ~/.local/bin/obsidian
```

Create a `.desktop` file for menu integration (replace username):
```bash
cat > ~/.local/share/applications/obsidian.desktop << 'EOF'
[Desktop Entry]
Name=Obsidian
Exec=/home/username/.local/bin/obsidian --no-sandbox
Icon=obsidian
Type=Application
Categories=Office;
EOF
```

Refresh menu:
```bash
update-desktop-database ~/.local/share/applications/
```
Icon for obsidian should be listed.

**Import from inotes to obsidian.**  
Currently there's no way to export in bulk from icloud notes.Requires plugin to install and activate inside Obsidian, but is very easy and intuitive to use.

**Problems encountered during testing and setup**

- mac folder syncs to linux, Obsidian shows old content.  
fix: ctrl + p  >> reload app without saving

- iphone change is not downloaded to macmini.  
fix: always keep finder open whith the Obsidian folder
*disabled "Optimize Storage" earlier, but macOS iCloud still uses **lazy sync**—files don't fully download until accessed. Opening the folder in Finder triggers the download.

- folder is not synced to mx linux  
fix: mac mini stays screen always on setting. Lock screen. Phisycal screen turned off button.

- after laptop restart folder is not mounted.  
fix: use autofs

- some imports failed.  
fix: manual transfer of each failed note.

**Launchd macos job**

Add a launchd (macos) job. It forces the entire tree to sync every hour (replace username):
```xml
cat > ~/Library/LaunchAgents/com.icloud.obsidian.sync.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.icloud.obsidian.sync</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>find /Users/username/Library/Mobile\ Documents/iCloud~md~obsidian/Documents -type f -exec cat {} \; > /dev/null 2>&1</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>3600</integer>
</dict>
</plist>
EOF
```
Load the job.
```bash
# If there is a old job, unload it.
launchctl unload ~/Library/LaunchAgents/com.icloud.obsidian.sync.plist

launchctl load ~/Library/LaunchAgents/com.icloud.obsidian.sync.plist
```

Something similar could probably also work with google drive. 

Thanks for the visit, have fun building ideas!