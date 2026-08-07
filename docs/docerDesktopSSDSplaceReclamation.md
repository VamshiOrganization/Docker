## Guide: Docker Desktop SSD Space Reclamation & Fixes

### Phase 1: Standard Docker Cleanup (Inside Docker)

First, clean up stopped containers, unused images, build caches, and volumes inside Docker.

Run in Command Prompt or PowerShell:

DOS

```
docker system prune -a --volumes -f
docker builder prune -a -f

```

> **Note:** This marks data as free _inside_ Linux, but Windows WSL virtual disks (`.vhdx`) do **not** automatically shrink on your physical C: drive.

### Phase 2: Finding & Compacting the Virtual Disk

#### Step 1: Locate the Large `.vhdx` File

Find out which file is actually consuming space by running this in PowerShell:

PowerShell

```
Get-ChildItem -Path "$env:LOCALAPPDATA\Docker\wsl" -Recurse -Filter "*.vhdx" | Select-Object FullName, @{Name="Size (GB)";Expression={"{0:N2}" -f ($_.Length / 1GB)}}

```

_(Look for the large file, typically located at `...\Docker\wsl\disk\docker_data.vhdx`)_.

#### Step 2: Attempt Standard Disk Compaction (`diskpart`)

1.  Quit Docker Desktop.
    
2.  Open **Command Prompt as Administrator** and run:
    
    DOS
    
    ```
    wsl --shutdown
    diskpart
    
    ```
    
3.  Inside `diskpart`:
    
    DOS
    
    ```
    select vdisk file="C:\Users\<Your-Username>\AppData\Local\Docker\wsl\disk\docker_data.vhdx"
    compact vdisk
    exit
    
    ```
    

### Phase 3: Direct File Reset (Guaranteed Space Recovery)

If `diskpart` doesn't shrink the file because Linux filesystem locks are preventing space release, reset the VHDX file manually.

1.  **Quit Docker Desktop** completely.
    
2.  Open **PowerShell as Administrator** and shut down WSL:
    
    PowerShell
    
    ```
    wsl --shutdown
    
    ```
    
3.  Delete the bloated virtual disk file directly:
    
    PowerShell
    
    ```
    Remove-Item -Path "$env:LOCALAPPDATA\Docker\wsl\disk\docker_data.vhdx" -Force
    
    ```
    
    _This immediately frees up all the disk space on your C: drive._
    

### Phase 4: Fixing Docker Desktop Startup (If It Won't Open)

If Docker Desktop fails to launch after manually deleting a `.vhdx` file, Windows WSL still has old file pointers cached in memory. Reset them using these steps:

#### Step 1: Force-kill all Docker processes

Open **Command Prompt as Administrator**:

DOS

```
taskkill /F /IM "Docker*" /T

```

#### Step 2: Unregister the cached WSL distribution

DOS

```
wsl --shutdown
wsl --unregister docker-desktop

```

#### Step 3: Restart Windows Docker Services

DOS

```
net stop com.docker.service
net start com.docker.service
wsl --shutdown

```

#### Step 4: Launch Docker Desktop

1.  Search for **Docker Desktop** in your Start menu.
    
2.  Right-click and choose **Run as Administrator**.
    
3.  Docker Desktop will re-initialize fresh WSL containers automatically in 15–30 seconds.
    

### Phase 5: Preventive Setup (Enable Auto-Shrink for Future)

To prevent WSL 2 virtual disks from holding onto deleted space in the future:

1.  Press `Win + R`, type `%USERPROFILE%`, and hit Enter.
    
2.  Create or open a file named `.wslconfig`.
    
3.  Add the following config:
    
    Ini, TOML
    
    ```
    [wsl2]
    sparseVhd=true
    
    ```
    
4.  Save the file and restart WSL in Command Prompt:
    
    DOS
    
    ```
    wsl --shutdown
    
    ```
