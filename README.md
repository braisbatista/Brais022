
name: RDP Skyro

on:
  workflow_dispatch:

jobs:
  rdp:
    runs-on: windows-latest
    timeout-minutes: 350

    steps:
      - name: Configurar RDP com segurança
        shell: pwsh
        run: |
          Set-ItemProperty `
            -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
            -Name 'fDenyTSConnections' -Value 0

          Set-ItemProperty `
            -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
            -Name 'UserAuthentication' -Value 1

          Enable-NetFirewallRule -Name 'RemoteDesktop-UserMode-In-TCP'

          Set-Service -Name TermService -StartupType Automatic
          Start-Service -Name TermService

      - name: Criar usuário RDP
        shell: pwsh
        env:
          RDP_PASSWORD: ${{ secrets.RDP_PASSWORD }}
        run: |
          if ([string]::IsNullOrWhiteSpace($env:RDP_PASSWORD)) {
            throw "Configure o segredo RDP_PASSWORD no GitHub."
          }

          $securePassword = ConvertTo-SecureString `
            $env:RDP_PASSWORD -AsPlainText -Force

          if (Get-LocalUser -Name "Skyro" -ErrorAction SilentlyContinue) {
            Set-LocalUser -Name "Skyro" -Password $securePassword
          } else {
            New-LocalUser `
              -Name "Skyro" `
              -Password $securePassword `
              -PasswordNeverExpires
          }

          Add-LocalGroupMember `
            -Group "Remote Desktop Users" `
            -Member "Skyro"

          if (-not (Get-LocalUser -Name "Skyro")) {
            throw "Falha ao criar usuário RDP."
          }

      - name: Conectar ao Tailscale
        uses: tailscale/github-action@v4
        with:
          authkey: ${{ secrets.TAILSCALE_AUTH_KEY }}
          hostname: skyro-rdp-${{ github.run_id }}

      - name: Verificar conexão
        shell: pwsh
        run: |
          $tsIP = $null

          for ($i = 0; $i -lt 20; $i++) {
            $tsIP = (tailscale ip -4 | Select-Object -First 1).Trim()

            if ($tsIP) {
              break
            }

            Start-Sleep -Seconds 5
          }

          if (-not $tsIP) {
            throw "Não foi possível obter o IP Tailscale."
          }

          "TAILSCALE_IP=$tsIP" >> $env:GITHUB_ENV

          Write-Host "IP Tailscale: $tsIP"

          $result = Test-NetConnection `
            -ComputerName $tsIP `
            -Port 3389 `
            -WarningAction SilentlyContinue

          if (-not $result.TcpTestSucceeded) {
            throw "Porta RDP não está acessível."
          }

          Write-Host "RDP acessível!"

      - name: Informações de acesso
        shell: pwsh
        run: |
          "## RDP SKYRO" >> $env:GITHUB_STEP_SUMMARY
          "IP: $env:TAILSCALE_IP" >> $env:GITHUB_STEP_SUMMARY
          "Usuário: Skyro" >> $env:GITHUB_STEP_SUMMARY
          "Porta: 3389" >> $env:GITHUB_STEP_SUMMARY
          "Senha: consulte seu segredo RDP_PASSWORD." >> $env:GITHUB_STEP_SUMMARY

      - name: Manter sessão ativa
        shell: pwsh
        run: |
          Write-Host "RDP SKYRO iniciado."

          while ($true) {
            Write-Host "RDP ativo: $(Get-Date)"
            Start-Sleep -Seconds 60
          }
          
