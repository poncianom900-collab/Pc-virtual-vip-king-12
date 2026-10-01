name: RDP

on:
  workflow_dispatch:

jobs:
  secure-rdp:
    runs-on: windows-latest
    timeout-minutes: 3600

    steps:
      - name: Configure RDP
        shell: powershell
        run: |
          $rdpPath = 'HKLM:\System\CurrentControlSet\Control\Terminal Server'
          $tcpPath = 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp'

          Set-ItemProperty -Path $rdpPath -Name 'fDenyTSConnections' -Value 0 -Force
          Set-ItemProperty -Path $tcpPath -Name 'UserAuthentication' -Value 0 -Force
          Set-ItemProperty -Path $tcpPath -Name 'SecurityLayer' -Value 0 -Force

          netsh advfirewall firewall delete rule name="RDP-Tailscale" 2>$null
          netsh advfirewall firewall add rule name="RDP-Tailscale" dir=in action=allow protocol=TCP localport=3389

          if (Get-Service -Name TermService -ErrorAction SilentlyContinue) {
              Restart-Service -Name TermService -Force
          }

      - name: Create RDP User
        shell: powershell
        run: |
          $username = "Skyro"

          $upper = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.ToCharArray()
          $lower = 'abcdefghijklmnopqrstuvwxyz'.ToCharArray()
          $numbers = '0123456789'.ToCharArray()
          $special = '!@#$%^&*()-_=+[]{}:,.?'.ToCharArray()

          $passwordChars = @()
          $passwordChars += $upper | Get-Random -Count 4
          $passwordChars += $lower | Get-Random -Count 4
          $passwordChars += $numbers | Get-Random -Count 4
          $passwordChars += $special | Get-Random -Count 4

          $password = -join ($passwordChars | Sort-Object { Get-Random })
          $securePassword = ConvertTo-SecureString $password -AsPlainText -Force

          if (Get-LocalUser -Name $username -ErrorAction SilentlyContinue) {
              Remove-LocalUser -Name $username
          }

          New-LocalUser `
            -Name $username `
            -Password $securePassword `
            -AccountNeverExpires `
            -PasswordNeverExpires `
            -UserMayNotChangePassword

          Add-LocalGroupMember -Group "Administrators" -Member $username -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member $username -ErrorAction SilentlyContinue

          if (-not (Get-LocalUser -Name $username -ErrorAction SilentlyContinue)) {
              throw "Failed to create RDP user."
          }

          "RDP_CREDS=User: $username | Password: $password" |
            Out-File $env:GITHUB_ENV -Append -Encoding utf8

      - name: Install Tailscale
        shell: powershell
        run: |
          $tsUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi"
          $installerPath = "$env:TEMP\tailscale.msi"

          Invoke-WebRequest -Uri $tsUrl -OutFile $installerPath

          Start-Process msiexec.exe `
            -ArgumentList "/i", "`"$installerPath`"", "/quiet", "/norestart" `
            -Wait

          Remove-Item $installerPath -Force

      - name: Establish Tailscale Connection
        shell: powershell
        run: |
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up `
            --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} `
            --hostname=gh-runner-$env:GITHUB_RUN_ID

          $tsIP = $null
          $retries = 0

          while (-not $tsIP -and $retries -lt 10) {
              $tsIP = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4

              if (-not $tsIP) {
                  Start-Sleep -Seconds 5
              }

              $retries++
          }

          if (-not $tsIP) {
              Write-Error "Tailscale IP not assigned. Exiting."
              exit 1
          }

          "TAILSCALE_IP=$tsIP" | Out-File $env:GITHUB_ENV -Append -Encoding utf8

      - name: Verify RDP Accessibility
        shell: powershell
        run: |
          $connected = $false

          for ($i = 1; $i -le 6; $i++) {
              $result = Test-NetConnection `
                -ComputerName $env:TAILSCALE_IP `
                -Port 3389 `
                -WarningAction SilentlyContinue

              if ($result.TcpTestSucceeded) {
                  $connected = $true
                  break
              }

              Start-Sleep -Seconds 5
          }

          if (-not $connected) {
              throw "RDP port 3389 is not reachable."
          }

      - name: Maintain Connection
        shell: powershell
        run: |
          Write-Host ""
          Write-Host "======================================"
          Write-Host "RDP ACCESS"
          Write-Host "======================================"
          Write-Host "Address : $env:TAILSCALE_IP"
          Write-Host "Username: Skyro"
          Write-Host "Credentials: $env:RDP_CREDS"
          Write-Host "======================================"
          Write-Host ""

          while ($true) {
              Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] RDP Active"
              Start-Sleep -Seconds 300
          }
