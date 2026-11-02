# 🖥️ Windows 11 VM en GitHub Actions con RDP

[![Video Tutorial](https://img.youtube.com/vi/6KDdo7-oPvY/maxresdefault.jpg)](https://youtu.be/6KDdo7-oPvY?si=DHSPfv8f2FH1PKIE)

Crea una máquina virtual Windows 11 gratis con 16GB RAM y 256GB de almacenamiento usando GitHub Actions. Acceso remoto seguro vía Tailscale VPN.


## 📋 Especificaciones

- **OS:** Windows Server (latest)
- **RAM:** 16 GB
- **Almacenamiento:** 256 GB SSD
- **Duración:** Hasta 6 horas

## 📄 Código Completo

```yaml
name: RDP Ultra Brasil (60FPS RDP + 90FPS Reemo + macOS Tahoe)

on:
  workflow_dispatch:

jobs:
  ultra-rdp-brasil:
    name: Windows RDP Brasil (Otimizado + Reemo 90FPS)
    runs-on: windows-latest
    timeout-minutes: 360

    steps:
      - name: 1. Corrigir Problemas do Azure & Desativar Servicos Pesados
        shell: pwsh
        run: |
          $ErrorActionPreference = "SilentlyContinue"
          Write-Host ">> Otimizando VM Azure e removendo gargalos de CPU/Rede..." -ForegroundColor Cyan

          # Fuso horario do Brasil (Brasilia UTC-3)
          Set-TimeZone -Id "E. South America Standard Time" -ErrorAction SilentlyContinue

          # Desativar Windows Defender Real-Time Monitoring e SmartScreen (evita bloqueio do Reemo e lag de CPU)
          Set-MpPreference -DisableRealtimeMonitoring $true -DisableBehaviorMonitoring $true -DisableIOAVProtection $true -DisableScriptScanning $true -ErrorAction SilentlyContinue
          New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender" -Force -ErrorAction SilentlyContinue | Out-Null
          Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender" -Name "DisableAntiSpyware" -Value 1 -Type DWord -Force -ErrorAction SilentlyContinue
          Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" -Name "SmartScreenEnabled" -Value "Off" -Force -ErrorAction SilentlyContinue

          # Desativar IE Enhanced Security Configuration (permite usar navegador sem bloqueios)
          Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" -Name "IsInstalled" -Value 0 -Force -ErrorAction SilentlyContinue
          Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" -Name "IsInstalled" -Value 0 -Force -ErrorAction SilentlyContinue

          # Parar servicos de telemetria, indexacao e Windows Update que causam picos de 100% CPU no Azure
          # (Mantemos RdAgent ativo para o Azure nunca derrubar a VM por falta de heartbeat)
          $heavyServices = @(
            "wuauserv", "UsoSvc", "DiagTrack", "SysMain", "WSearch",
            "Spooler", "MapsBroker", "lfsvc", "WerSvc", "PcaSvc"
          )
          foreach ($svc in $heavyServices) {
            if (Get-Service -Name $svc -ErrorAction SilentlyContinue) {
              Stop-Service -Name $svc -Force -NoWait -ErrorAction SilentlyContinue
              Set-Service -Name $svc -StartupType Disabled -ErrorAction SilentlyContinue
            }
          }

          Stop-Process -Name "ServerManager" -Force -ErrorAction SilentlyContinue
          New-Item -Path "HKLM:\SOFTWARE\Microsoft\ServerManager" -Force -ErrorAction SilentlyContinue | Out-Null
          Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\ServerManager" -Name "DoNotOpenServerManagerAtLogon" -Value 1 -Type DWord -Force -ErrorAction SilentlyContinue

          Write-Host ">> Problemas de interferencia do Azure corrigidos!" -ForegroundColor Green
          $global:LASTEXITCODE = 0

      - name: 2. Otimizar Rede para o Brasil (Menor Ping, Sem Nagle, Plano Desempenho Maximo)
        shell: pwsh
        run: |
          $ErrorActionPreference = "SilentlyContinue"
          Write-Host ">> Aplicando ajustes de baixa latencia para conexao Brasil <-> VM..." -ForegroundColor Cyan

          # Ativar Plano de Energia "Ultimate Performance" (Desempenho Maximo)
          powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61 | Out-Null
          powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c | Out-Null
          powercfg /change monitor-timeout-ac 0 | Out-Null
          powercfg /change standby-timeout-ac 0 | Out-Null

          # Prioridade Multimidia e Jogos (0% Throttling de Rede, GPU Priority 8)
          $mmProfile = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile"
          Set-ItemProperty -Path $mmProfile -Name "NetworkThrottlingIndex" -Value 0xffffffff -Type DWord -Force
          Set-ItemProperty -Path $mmProfile -Name "SystemResponsiveness" -Value 0 -Type DWord -Force
          $gamesTask = "$mmProfile\Tasks\Games"
          New-Item -Path $gamesTask -Force | Out-Null
          Set-ItemProperty -Path $gamesTask -Name "GPU Priority" -Value 8 -Type DWord -Force
          Set-ItemProperty -Path $gamesTask -Name "Priority" -Value 6 -Type DWord -Force
          Set-ItemProperty -Path $gamesTask -Name "Scheduling Category" -Value "High" -Force
          Set-ItemProperty -Path $gamesTask -Name "SFIO Priority" -Value "High" -Force

          # Win32PrioritySeparation = 38 (Foco total em janela ativa e resposta imediata de mouse/teclado)
          Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl" -Name "Win32PrioritySeparation" -Value 38 -Type DWord -Force

          # Otimizar TCP/UDP para rota Brasil (Desativar Algoritmo de Nagle: TcpAckFrequency=1 e TCPNoDelay=1)
          netsh int tcp set global autotuninglevel=normal | Out-Null
          netsh int tcp set global rss=enabled | Out-Null
          netsh int tcp set global ecncapability=enabled | Out-Null
          netsh int tcp set global timestamps=disabled | Out-Null

          $interfaces = Get-ChildItem "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces" -ErrorAction SilentlyContinue
          foreach ($iface in $interfaces) {
            Set-ItemProperty -Path $iface.PSPath -Name "TcpAckFrequency" -Value 1 -Type DWord -Force -ErrorAction SilentlyContinue
            Set-ItemProperty -Path $iface.PSPath -Name "TCPNoDelay" -Value 1 -Type DWord -Force -ErrorAction SilentlyContinue
            Set-ItemProperty -Path $iface.PSPath -Name "TcpDelAckTicks" -Value 0 -Type DWord -Force -ErrorAction SilentlyContinue
          }

          Write-Host ">> Rede e CPU otimizadas para baixa latencia no Brasil!" -ForegroundColor Green
          $global:LASTEXITCODE = 0

      - name: 3. Verificar Ferramentas (Node, Python, Git, 7-Zip, OpenCode)
        shell: pwsh
        run: |
          $ErrorActionPreference = "SilentlyContinue"
          $env:Path = [Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [Environment]::GetEnvironmentVariable("Path","User")
          foreach ($pkg in @("7zip", "git", "nodejs-lts", "python")) {
            $cmdName = switch ($pkg) { "7zip" { "7z" } "nodejs-lts" { "node" } default { $pkg } }
            if (-not (Get-Command $cmdName -ErrorAction SilentlyContinue)) {
              choco install $pkg -y --no-progress
            }
          }
          $env:Path = [Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [Environment]::GetEnvironmentVariable("Path","User")
          npm install -g opencode-ai@latest
          $npmBin = (npm config get prefix).Trim()
          $machinePath = [Environment]::GetEnvironmentVariable("Path","Machine")
          if ($machinePath -notlike "*$npmBin*") {
            [Environment]::SetEnvironmentVariable("Path", "$machinePath;$npmBin", "Machine")
          }
          $env:Path += ";$npmBin"
          Write-Host ">> Ferramentas prontas!" -ForegroundColor Green
          $global:LASTEXITCODE = 0

      - name: 4. Instalar Mesa3D + Aceleracao OpenGL (FPS Desbloqueado)
        shell: pwsh
        run: |
          $ErrorActionPreference = "SilentlyContinue"
          $ver = "24.3.4"
          $url = "https://github.com/pal1000/mesa-dist-win/releases/download/$ver/mesa3d-$ver-release-msvc.7z"
          Invoke-WebRequest -Uri $url -OutFile "$env:TEMP\mesa3d.7z" -UseBasicParsing

          $mesaDir = "C:\mesa3d"
          New-Item -ItemType Directory -Force -Path $mesaDir | Out-Null
          $7zPath = if (Test-Path "C:\Program Files\7-Zip\7z.exe") { "C:\Program Files\7-Zip\7z.exe" } else { "7z.exe" }
          & $7zPath x "$env:TEMP\mesa3d.7z" -o"$mesaDir" -y | Out-Null

          $vars = @{
            "GALLIUM_DRIVER"              = "llvmpipe"
            "LP_NUM_THREADS"              = "$env:NUMBER_OF_PROCESSORS"
            "MESA_GL_VERSION_OVERRIDE"    = "4.6COMPAT"
            "MESA_GLSL_VERSION_OVERRIDE"  = "460"
            "MESA_GLTHREAD"               = "true"
            "vblank_mode"                 = "0"
          }
          foreach ($k in $vars.Keys) {
            [Environment]::SetEnvironmentVariable($k, $vars[$k], "Machine")
          }

          $x64 = (Get-ChildItem $mesaDir -Directory -Recurse -Filter "x64" | Select-Object -First 1).FullName
          $batLines = @(
            "@echo off",
            "if `"%~1`"==`"`" (echo Arraste a pasta do app ou jogo sobre este arquivo & pause & exit /b)",
            "copy /Y `"$x64\opengl32.dll`" `"%~1\`" >nul",
            "copy /Y `"$x64\libgallium_wgl.dll`" `"%~1\`" >nul",
            "copy /Y `"$x64\libglapi.dll`" `"%~1\`" >nul",
            "echo Mesa3D (FPS Desbloqueado) ativado em: %~1",
            "pause"
          )
          Set-Content -Path "C:\Users\Public\Desktop\Ativar-3D-em-App.bat" -Value ($batLines -join "`r`n") -Encoding ASCII
          Write-Host ">> Mesa3D instalado com vblank_mode=0 (FPS desbloqueado)!" -ForegroundColor Green
          $global:LASTEXITCODE = 0

      - name: 5. Desbloquear 60 FPS no Windows App (RDP) + Configurar Reemo 90 FPS (Mesma Tela Principal)
        shell: pwsh
        run: |
          $ErrorActionPreference = "SilentlyContinue"
          Write-Host ">> Configurando RDP a 60 FPS, AVC444/H.264, UDP e Unificacao de Tela para Reemo..." -ForegroundColor Cyan

          # 1. Habilitar conexoes RDP e forcar sessao unica por usuario (impede criar 2a sessao separada)
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fSingleSessionPerUser" -Value 1 -Type DWord -Force

          $rdpTcp = 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp'
          Set-ItemProperty -Path $rdpTcp -Name "UserAuthentication" -Value 0 -Type DWord -Force
          Set-ItemProperty -Path $rdpTcp -Name "SecurityLayer" -Value 0 -Type DWord -Force
          Set-ItemProperty -Path $rdpTcp -Name "MaxMonitors" -Value 1 -Type DWord -Force

          # 2. DESBLOQUEAR 60 FPS NO RDP (DWMFRAMEINTERVAL = 15 -> 1000ms / 15ms = ~66 FPS)
          $winStations = 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations'
          Set-ItemProperty -Path $winStations -Name "DWMFRAMEINTERVAL" -Value 15 -Type DWord -Force
          Set-ItemProperty -Path $rdpTcp -Name "DWMFRAMEINTERVAL" -Value 15 -Type DWord -Force

          # 3. Ativar Codec H.264 / AVC444, Transporte UDP e Permitir Wallpaper no RDP
          $tsPolicies = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services'
          New-Item -Path $tsPolicies -Force | Out-Null
          Set-ItemProperty -Path $tsPolicies -Name "fSingleSessionPerUser" -Value 1 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "AVC444ModePreferred" -Value 1 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "AVCHardwareEncodePreferred" -Value 1 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "VisualExperiencePolicy" -Value 1 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "MaxCompressionLevel" -Value 1 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "SelectTransport" -Value 0 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "fClientDisableUDP" -Value 0 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "fNoRemoteDesktopWallpaper" -Value 0 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "EnforceRemovalOfRemoteDesktopWallpaper" -Value 0 -Type DWord -Force
          Set-ItemProperty -Path $tsPolicies -Name "fResetBroken" -Value 0 -Type DWord -Force

          # 4. Abrir portas TCP + UDP no Firewall (RDP 3389, Tailscale 41641 P2P Direto Brasil, Reemo WebRTC/UDP)
          netsh advfirewall firewall delete rule name="RDP-Tailscale" 2>$null | Out-Null
          netsh advfirewall firewall add rule name="RDP-TCP-3389" dir=in action=allow protocol=TCP localport=3389 | Out-Null
          netsh advfirewall firewall add rule name="RDP-UDP-3389" dir=in action=allow protocol=UDP localport=3389 | Out-Null
          netsh advfirewall firewall add rule name="Tailscale-Direct-Brazil-UDP" dir=in action=allow protocol=UDP localport=41641 | Out-Null
          netsh advfirewall firewall add rule name="Reemo-WebRTC-UDP" dir=in action=allow protocol=UDP localport=3478,49152-65535 | Out-Null
          netsh advfirewall firewall add rule name="Reemo-TCP" dir=in action=allow protocol=TCP localport=443,8888 | Out-Null

          # 5. Pre-baixar o instalador oficial do Reemo Desktop na Area de Trabalho
          $reemoInstaller = "C:\Users\Public\Desktop\1-Instalar-Reemo-Desktop.exe"
          try {
            Invoke-WebRequest -Uri "https://download.reemo.io/reemo.setup.x64.exe" -OutFile $reemoInstaller -UseBasicParsing
            Write-Host ">> Instalador do Reemo salvo na Area de Trabalho!" -ForegroundColor Green
          } catch {
            Write-Host ">> Aviso: falha ao pre-baixar Reemo, voce pode baixar pelo navegador." -ForegroundColor Yellow
          }

          # 6. Script para Unificar a Tela Principal ao entrar pelo Reemo (evita abrir 2a tela vazia)
          $fixReemoLines = @(
            "@echo off",
            "title Ativar Tela Principal Unica no Reemo (90 FPS)",
            "echo ==========================================================================",
            "echo   UNIFICANDO TELA PRINCIPAL PARA O REEMO DESKTOP (90 FPS)",
            "echo ==========================================================================",
            "echo [1/3] Desativando tela estendida (forcando Tela Unica / Espelhada)...",
            "DisplaySwitch.exe /clone 2>nul",
            "echo [2/3] Liberando Reemo no Firewall...",
            "if exist `"C:\Program Files\Reemo\reemo.exe`" (",
            "  netsh advfirewall firewall add rule name=`"Reemo-App`" dir=in action=allow program=`"C:\Program Files\Reemo\reemo.exe`" enable=yes >nul 2>&1",
            ")",
            "echo [3/3] Transferindo sua sessao atual do Windows App para a Tela Principal (Console)...",
            "echo       O Windows App vai fechar agora e sua mesma tela continuara aberta no Reemo!",
            "timeout /t 2 /nobreak >nul",
            "for /f `"tokens=3`" %%i in ('query session ^| findstr /C:`"Active`"') do (",
            "  tscon %%i /dest:console",
            ")",
            "DisplaySwitch.exe /clone 2>nul"
          )
          Set-Content -Path "C:\Users\Public\Desktop\2-Ativar-Tela-Principal-Reemo-90FPS.bat" -Value ($fixReemoLines -join "`r`n") -Encoding ASCII

          Restart-Service -Name TermService -Force
          Write-Host ">> RDP 60 FPS e Unificador de Tela Principal do Reemo configurados!" -ForegroundColor Green
          $global:LASTEXITCODE = 0

      - name: 6. Instalar Cursor macOS Tahoe + Wallpaper 4K Hinata Hyuga (Naruto) + Visual Rapido
        shell: pwsh
        run: |
          $ErrorActionPreference = "SilentlyContinue"
          Write-Host ">> Instalando Cursor macOS Tahoe e Wallpaper 4K da Hinata (Naruto)..." -ForegroundColor Cyan

          $customDir = "C:\CustomTheme"
          $cursorDest = "C:\Windows\Cursors\macOS-Tahoe"
          $wallDir = "C:\Wallpaper"
          New-Item -ItemType Directory -Force -Path $customDir, $cursorDest, $wallDir | Out-Null

          # --- A. BAIXAR E EXTRAIR CURSOR MACOS TAHOE ---
          git clone --depth 1 "https://github.com/dazham/MacOS-Cursors-on-Windows.git" "$customDir\MacOS-Cursors" 2>$null
          $tahoeFolder = Get-ChildItem -Path "$customDir\MacOS-Cursors" -Directory -Recurse | Where-Object { $_.Name -like "*Tahoe*" -and $_.Name -notlike "*Tail less*" } | Select-Object -First 1
          if (-not $tahoeFolder) {
            $tahoeFolder = Get-ChildItem -Path "$customDir\MacOS-Cursors" -Directory -Recurse | Where-Object { $_.Name -like "*Tahoe*" } | Select-Object -First 1
          }

          if ($tahoeFolder) {
            $curFiles = Get-ChildItem -Path $tahoeFolder.FullName -Include "*.cur","*.ani" -Recurse
            foreach ($f in $curFiles) {
              Copy-Item -Path $f.FullName -Destination "$cursorDest\$($f.Name)" -Force
            }
            $infFile = Get-ChildItem -Path $tahoeFolder.FullName -Filter "*.inf" -Recurse | Select-Object -First 1
            if ($infFile) {
              Copy-Item -Path $infFile.FullName -Destination "$cursorDest\$($infFile.Name)" -Force
              Start-Process "rundll32.exe" -ArgumentList "setupapi.dll,InstallHinfSection DefaultInstall 132 `"$cursorDest\$($infFile.Name)`"" -Wait
            }
            Write-Host ">> Cursores macOS Tahoe copiados para $cursorDest!" -ForegroundColor Green
          }

          function Find-TahoeCursor([string[]]$patterns) {
            foreach ($p in $patterns) {
              $match = Get-ChildItem -Path $cursorDest -File | Where-Object { $_.BaseName -match $p } | Select-Object -First 1
              if ($match) { return $match.FullName }
            }
            return ""
          }

          $cursorMap = @{
            "Arrow"       = Find-TahoeCursor @("^arrow$", "normal", "pointer", "default", "left_ptr")
            "Help"        = Find-TahoeCursor @("help", "question")
            "AppStarting" = Find-TahoeCursor @("work", "working", "appstarting", "progress", "busy")
            "Wait"        = Find-TahoeCursor @("^wait$", "busy")
            "Crosshair"   = Find-TahoeCursor @("cross", "precision")
            "IBeam"       = Find-TahoeCursor @("beam", "ibeam", "text", "xterm")
            "NWPen"       = Find-TahoeCursor @("pen", "handwriting", "nwpen")
            "No"          = Find-TahoeCursor @("^no$", "unavailable", "forbidden")
            "SizeNS"      = Find-TahoeCursor @("vsize", "sizens", "ns", "vertical")
            "SizeWE"      = Find-TahoeCursor @("hsize", "sizewe", "we", "horizontal")
            "SizeNWSE"    = Find-TahoeCursor @("dsize1", "sizenwse", "nwse", "diagonal1")
            "SizeNESW"    = Find-TahoeCursor @("dsize2", "sizenesw", "nesw", "diagonal2")
            "SizeAll"     = Find-TahoeCursor @("move", "sizeall", "fleur")
            "UpArrow"     = Find-TahoeCursor @("^up$", "uparrow", "alternate", "center")
            "Hand"        = Find-TahoeCursor @("link", "hand", "pointing")
            "Pin"         = Find-TahoeCursor @("location", "pin")
            "Person"      = Find-TahoeCursor @("person", "user")
          }
          $cursorJsonPath = "$customDir\cursors.json"
          $cursorMap | ConvertTo-Json | Set-Content -Path $cursorJsonPath -Encoding UTF8

          # --- B. BAIXAR E RENDERIZAR WALLPAPER HINATA HYUGA (NARUTO) EM 4K (3840x2160) ---
          $wallUrls = @(
            "https://images6.alphacoders.com/385/385172.jpg",
            "https://images6.alphacoders.com/385/thumb-1920-385172.jpg",
            "https://images4.alphacoders.com/117/thumb-1920-1174666.png",
            "https://images8.alphacoders.com/902/thumb-1920-902450.jpg",
            "https://images7.alphacoders.com/769/thumb-1920-769867.jpg"
          )
          $rawWall = "$wallDir\hinata_source.jpg"
          $finalWall4K = "$wallDir\Hinata-Hyuga-4K.bmp"
          $finalWallJpg = "$wallDir\Hinata-Hyuga-4K.jpg"

          $downloaded = $false
          [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
          foreach ($u in $wallUrls) {
            try {
              Invoke-WebRequest -Uri $u -OutFile $rawWall -Headers @{"User-Agent"="Mozilla/5.0 (Windows NT 10.0; Win64; x64)"} -UseBasicParsing -TimeoutSec 20
              if ((Test-Path $rawWall) -and ((Get-Item $rawWall).Length -gt 30000)) {
                $downloaded = $true
                Write-Host ">> Wallpaper da Hinata baixado de: $u" -ForegroundColor Green
                break
              }
            } catch {}
          }

          if ($downloaded) {
            try {
              Add-Type -AssemblyName System.Drawing
              $srcImg = [System.Drawing.Image]::FromFile($rawWall)
              $bmp4k = New-Object System.Drawing.Bitmap(3840, 2160)
              $bmp4k.SetResolution(96, 96)
              $g = [System.Drawing.Graphics]::FromImage($bmp4k)
              $g.InterpolationMode = [System.Drawing.Drawing2D.InterpolationMode]::HighQualityBicubic
              $g.SmoothingMode = [System.Drawing.Drawing2D.SmoothingMode]::HighQuality
              $g.PixelOffsetMode = [System.Drawing.Drawing2D.PixelOffsetMode]::HighQuality
              $g.CompositingQuality = [System.Drawing.Drawing2D.CompositingQuality]::HighQuality
              $g.DrawImage($srcImg, 0, 0, 3840, 2160)
              $g.Dispose()
              $srcImg.Dispose()
              $bmp4k.Save($finalWall4K, [System.Drawing.Imaging.ImageFormat]::Bmp)
              $bmp4k.Save($finalWallJpg, [System.Drawing.Imaging.ImageFormat]::Jpeg)
              $bmp4k.Dispose()
              Copy-Item -Path $finalWallJpg -Destination "C:\Users\Public\Desktop\Wallpaper-Hinata-4K.jpg" -Force
              Write-Host ">> Wallpaper Hinata renderizado em 4K UHD (3840x2160)!" -ForegroundColor Green
            } catch {
              Copy-Item -Path $rawWall -Destination $finalWall4K -Force
            }
          }

          # --- C. SCRIPT DE LOGIN PARA APLICAR CURSOR TAHOE + WALLPAPER 4K + MOUSE SEM ACELERACAO ---
          $applyScriptPath = "C:\CustomTheme\Apply-Tahoe-And-Hinata4K.ps1"
          $scriptLines = @(
            "`$ErrorActionPreference = 'SilentlyContinue'",
            "`$cPath = 'HKCU:\Control Panel\Cursors'",
            "Set-ItemProperty -Path `$cPath -Name '(Default)' -Value 'macOS Tahoe' -Force",
            "Set-ItemProperty -Path `$cPath -Name 'Scheme Source' -Value 2 -Type DWord -Force",
            "if (Test-Path 'C:\CustomTheme\cursors.json') {",
            "  `$map = Get-Content 'C:\CustomTheme\cursors.json' -Raw | ConvertFrom-Json",
            "  foreach (`$prop in `$map.PSObject.Properties) {",
            "    if (`$prop.Value -and (Test-Path `$prop.Value)) {",
            "      Set-ItemProperty -Path `$cPath -Name `$prop.Name -Value `$prop.Value -Force",
            "    }",
            "  }",
            "}",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Mouse' -Name 'MouseSpeed' -Value '0' -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Mouse' -Name 'MouseThreshold1' -Value '0' -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Mouse' -Name 'MouseThreshold2' -Value '0' -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'MenuShowDelay' -Value '0' -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'JPEGImportQuality' -Value 100 -Type DWord -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'WallpaperStyle' -Value '10' -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'TileWallpaper' -Value '0' -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'WallPaper' -Value 'C:\Wallpaper\Hinata-Hyuga-4K.bmp' -Force",
            "Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop\WindowMetrics' -Name 'MinAnimate' -Value '0' -Force",
            "Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\VisualEffects' -Name 'VisualFXSetting' -Value 2 -Type DWord -Force",
            "Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize' -Name 'EnableTransparency' -Value 0 -Type DWord -Force",
            "Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize' -Name 'AppsUseLightTheme' -Value 0 -Type DWord -Force",
            "Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize' -Name 'SystemUsesLightTheme' -Value 0 -Type DWord -Force",
            "Add-Type -MemberDefinition '[DllImport(`"user32.dll`", CharSet = CharSet.Auto)] public static extern int SystemParametersInfo(int uAction, int uParam, string lpvParam, int fuWinIni);' -Name 'Win32Theme' -Namespace 'Win32' -ErrorAction SilentlyContinue",
            "[Win32.Win32Theme]::SystemParametersInfo(0x0057, 0, `$null, 3) | Out-Null",
            "if (Test-Path 'C:\Wallpaper\Hinata-Hyuga-4K.bmp') {",
            "  [Win32.Win32Theme]::SystemParametersInfo(0x0014, 0, 'C:\Wallpaper\Hinata-Hyuga-4K.bmp', 3) | Out-Null",
            "}",
            "Start-Process 'DisplaySwitch.exe' -ArgumentList '/clone' -WindowStyle Hidden -ErrorAction SilentlyContinue"
          )
          Set-Content -Path $applyScriptPath -Value ($scriptLines -join "`r`n") -Encoding UTF8

          powershell.exe -NoProfile -ExecutionPolicy Bypass -File $applyScriptPath
          Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "ApplyMacOSTahoeAndHinata4K" -Value "powershell.exe -WindowStyle Hidden -NoProfile -ExecutionPolicy Bypass -File `"$applyScriptPath`"" -Force

          $reapplyLines = @(
            "@echo off",
            "powershell.exe -NoProfile -ExecutionPolicy Bypass -File `"$applyScriptPath`"",
            "echo Cursor macOS Tahoe e Wallpaper Hinata 4K aplicados!",
            "timeout /t 2 >nul"
          )
          Set-Content -Path "C:\Users\Public\Desktop\3-Reaplicar-Cursor-Tahoe-e-Wallpaper-4K.bat" -Value ($reapplyLines -join "`r`n") -Encoding ASCII
          Write-Host ">> Cursor macOS Tahoe e Wallpaper 4K configurados!" -ForegroundColor Green
          $global:LASTEXITCODE = 0

      - name: 7. Criar e Unificar Usuario Principal (Mesma Sessao para RDP e Reemo)
        shell: pwsh
        run: |
          $ErrorActionPreference = "SilentlyContinue"
          $upper = (65..90) | ForEach-Object { [char]$_ }
          $lower = (97..122) | ForEach-Object { [char]$_ }
          $digits = (48..57) | ForEach-Object { [char]$_ }
          $randPart = -join ((1..10) | ForEach-Object { ($upper + $lower + $digits) | Get-Random })
          $password = "BrRdp@" + $randPart + "90"

          $secure = ConvertTo-SecureString $password -AsPlainText -Force

          # Configura a mesma senha para 'runneradmin' (dono da Tela Principal Console Session 1) e para 'RDP'
          Set-LocalUser -Name "runneradmin" -Password $secure -PasswordNeverExpires $true -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Administrators" -Member "runneradmin" -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "runneradmin" -ErrorAction SilentlyContinue

          New-LocalUser -Name "RDP" -Password $secure -AccountNeverExpires -ErrorAction SilentlyContinue | Out-Null
          Set-LocalUser -Name "RDP" -Password $secure -PasswordNeverExpires $true -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Administrators" -Member "RDP" -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RDP" -ErrorAction SilentlyContinue

          $password | Out-File -FilePath "C:\Users\Public\Desktop\rdp_password.txt" -Encoding utf8
          "RDP_PASSWORD=$password" | Out-File -FilePath $env:GITHUB_ENV -Append -Encoding utf8
          Write-Host ">> Usuarios runneradmin e RDP configurados com sucesso!" -ForegroundColor Green
          $global:LASTEXITCODE = 0

      - name: 8. Instalar Tailscale
        shell: pwsh
        run: |
          $installer = "$env:TEMP\tailscale.msi"
          Invoke-WebRequest -Uri "https://pkgs.tailscale.com/stable/tailscale-setup-latest-amd64.msi" -OutFile $installer -UseBasicParsing
          Start-Process msiexec.exe -ArgumentList "/i","`"$installer`"","/quiet","/norestart" -Wait
          Remove-Item $installer -Force -ErrorAction SilentlyContinue
          $global:LASTEXITCODE = 0

      - name: 9. Conectar Tailscale
        shell: pwsh
        run: |
          $ts = "$env:ProgramFiles\Tailscale\tailscale.exe"
          & $ts up --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} --hostname=gh-runner-$env:GITHUB_RUN_ID

          $tsIP = $null; $i = 0
          while (-not $tsIP -and $i -lt 15) {
            Start-Sleep -Seconds 4
            $tsIP = (& $ts ip -4 2>$null | Select-Object -First 1)
            $i++
          }
          if (-not $tsIP) { Write-Error "Tailscale IP not assigned."; exit 1 }
          "TAILSCALE_IP=$tsIP" | Out-File -FilePath $env:GITHUB_ENV -Append -Encoding utf8
          Get-NetConnectionProfile | Where-Object { $_.InterfaceAlias -like "*Tailscale*" } | Set-NetConnectionProfile -NetworkCategory Private -ErrorAction SilentlyContinue
          $global:LASTEXITCODE = 0

      - name: 10. Manter RDP Online (Keep Alive)
        shell: pwsh
        run: |
          Write-Host "==================================================================" -ForegroundColor Green
          Write-Host "          RDP ULTRA BRASIL (60 FPS RDP + 90 FPS REEMO)            " -ForegroundColor Yellow
          Write-Host "==================================================================" -ForegroundColor Green
          Write-Host "IP Tailscale             : $env:TAILSCALE_IP"
          Write-Host "Usuario Principal (Reemo): runneradmin   <-- USE ESTE PARA MESMA TELA NO REEMO!"
          Write-Host "Usuario Alternativo      : RDP"
          Write-Host "Password                 : $env:RDP_PASSWORD"
          Write-Host "------------------------------------------------------------------"
          Write-Host "PASSO A PASSO PARA REEMO 90 FPS NA TELA PRINCIPAL:"
          Write-Host "1. Entre no Windows App usando User: runneradmin (ou RDP)"
          Write-Host "2. Abra '1-Instalar-Reemo-Desktop.exe' na Area de Trabalho"
          Write-Host "3. Clique 2x em '2-Ativar-Tela-Principal-Reemo-90FPS.bat' na Area de Trabalho"
          Write-Host "   para transferir a sessao para a Tela Principal antes de conectar pelo Reemo!"
          Write-Host "==================================================================" -ForegroundColor Green
          while ($true) { Start-Sleep -Seconds 60 }  
```

## ⚠️ Importante

- Solo para uso personal y educativo
- No minería de criptomonedas
- Los datos no persisten entre ejecuciones
- Requiere Tailscale instalado en tu PC

## 📝 Licencia

MIT License
