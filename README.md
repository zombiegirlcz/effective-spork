# ---------------------------------------------------------
# WINS EMERGENCY SYSTEM OVERRIDE V1.0
# Účel: Náhrada za blokovaný Task Manager a RegEdit
# Běží v kontextu PowerShellu, obchází GPO restrikce na .exe
# ---------------------------------------------------------

Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# --- Hlavní okno ---
$form = New-Object System.Windows.Forms.Form
$form.Text = "Wins Emergency Protocol - System Override"
$form.Size = New-Object System.Drawing.Size(600, 500)
$form.StartPosition = "CenterScreen"
$form.BackColor = [System.Drawing.Color]::FromArgb(30,30,30)
$form.ForeColor = [System.Drawing.Color]::White

# --- Tabs (Záložky) ---
$tabControl = New-Object System.Windows.Forms.TabControl
$tabControl.Dock = "Fill"
$form.Controls.Add($tabControl)

# --- TAB 1: Procesy (Náhrada Task Manageru) ---
$tabProc = New-Object System.Windows.Forms.TabPage
$tabProc.Text = "Správce Procesů"
$tabProc.BackColor = [System.Drawing.Color]::FromArgb(45,45,48)
$tabControl.Controls.Add($tabProc)

$listBox = New-Object System.Windows.Forms.ListBox
$listBox.Location = New-Object System.Drawing.Point(10, 10)
$listBox.Size = New-Object System.Drawing.Size(400, 350)
$listBox.BackColor = [System.Drawing.Color]::Black
$listBox.ForeColor = [System.Drawing.Color]::LimeGreen
$tabProc.Controls.Add($listBox)

$btnRefresh = New-Object System.Windows.Forms.Button
$btnRefresh.Text = "Obnovit seznam"
$btnRefresh.Location = New-Object System.Drawing.Point(420, 10)
$btnRefresh.Size = New-Object System.Drawing.Size(150, 40)
$btnRefresh.BackColor = [System.Drawing.Color]::DarkBlue
$btnRefresh.ForeColor = [System.Drawing.Color]::White
$tabProc.Controls.Add($btnRefresh)

$btnKill = New-Object System.Windows.Forms.Button
$btnKill.Text = "UKONČIT PROCES (Kill)"
$btnKill.Location = New-Object System.Drawing.Point(420, 60)
$btnKill.Size = New-Object System.Drawing.Size(150, 40)
$btnKill.BackColor = [System.Drawing.Color]::DarkRed
$btnKill.ForeColor = [System.Drawing.Color]::White
$tabProc.Controls.Add($btnKill)

# Logika pro procesy
$RefreshProcs = {
    $listBox.Items.Clear()
    $procs = Get-Process | Sort-Object ProcessName
    foreach ($p in $procs) {
        $listBox.Items.Add("$($p.Id) - $($p.ProcessName)")
    }
}
$btnRefresh.Add_Click($RefreshProcs)

$btnKill.Add_Click({
    if ($listBox.SelectedItem) {
        try {
            $pidToKill = $listBox.SelectedItem.Split("-")[0].Trim()
            Stop-Process -Id $pidToKill -Force -ErrorAction Stop
            [System.Windows.Forms.MessageBox]::Show("Proces $pidToKill byl násilně ukončen.", "Success")
            & $RefreshProcs
        } catch {
             [System.Windows.Forms.MessageBox]::Show("Chyba při ukončování: $_", "Error")
        }
    }
})

# --- TAB 2: Registry & Fixy (Odblokování) ---
$tabReg = New-Object System.Windows.Forms.TabPage
$tabReg.Text = "Registry & Opravy"
$tabReg.BackColor = [System.Drawing.Color]::FromArgb(45,45,48)
$tabControl.Controls.Add($tabReg)

$lblStatus = New-Object System.Windows.Forms.Label
$lblStatus.Location = New-Object System.Drawing.Point(10, 200)
$lblStatus.Size = New-Object System.Drawing.Size(500, 100)
$lblStatus.Text = "Stav: Připraven..."
$tabReg.Controls.Add($lblStatus)

# Funkce pro odblokování
$btnUnblock = New-Object System.Windows.Forms.Button
$btnUnblock.Text = "ODBLOKOVAT TaskMgr/RegEdit"
$btnUnblock.Location = New-Object System.Drawing.Point(20, 20)
$btnUnblock.Size = New-Object System.Drawing.Size(250, 50)
$btnUnblock.BackColor = [System.Drawing.Color]::DarkOrange
$btnUnblock.ForeColor = [System.Drawing.Color]::Black
$tabReg.Controls.Add($btnUnblock)

$btnUnblock.Add_Click({
    $paths = @(
        "HKCU:\Software\Microsoft\Windows\CurrentVersion\Policies\System",
        "HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System"
    )
    
    $log = ""
    
    foreach ($path in $paths) {
        if (Test-Path $path) {
            try {
                # Pokus o nastavení hodnot na 0 (Vypnutí zákazu)
                Set-ItemProperty -Path $path -Name "DisableTaskMgr" -Value 0 -ErrorAction SilentlyContinue
                Set-ItemProperty -Path $path -Name "DisableRegistryTools" -Value 0 -ErrorAction SilentlyContinue
                Set-ItemProperty -Path $path -Name "DisableCMD" -Value 0 -ErrorAction SilentlyContinue
                $log += "Pokus o opravu v $path dokončen.`n"
            } catch {
                $log += "Chyba přístupu k $path (možná potřeba Admin).`n"
            }
        }
    }
    
    # Zkusit smazat IFEO (častý trik malwaru na blokování exe)
    $ifeoPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options"
    $targets = @("taskmgr.exe", "regedit.exe", "cmd.exe")
    foreach ($t in $targets) {
        if (Test-Path "$ifeoPath\$t") {
             $log += "Nalezen blok v IFEO pro $t - Zkouším odstranit...`n"
             try { Remove-Item "$ifeoPath\$t" -Recurse -Force; $log += "Odstraněno.`n" } catch { $log += "Selhalo odstranění.`n" }
        }
    }

    $lblStatus.Text = $log
})

# --- TAB 3: Shell (Konzole) ---
$tabShell = New-Object System.Windows.Forms.TabPage
$tabShell.Text = "Mini Konzole"
$tabShell.BackColor = [System.Drawing.Color]::FromArgb(45,45,48)
$tabControl.Controls.Add($tabShell)

$txtCommand = New-Object System.Windows.Forms.TextBox
$txtCommand.Location = New-Object System.Drawing.Point(10, 10)
$txtCommand.Size = New-Object System.Drawing.Size(450, 30)
$tabShell.Controls.Add($txtCommand)

$btnRun = New-Object System.Windows.Forms.Button
$btnRun.Text = "Spustit"
$btnRun.Location = New-Object System.Drawing.Point(470, 8)
$btnRun.Size = New-Object System.Drawing.Size(80, 32)
$tabShell.Controls.Add($btnRun)

$txtOutput = New-Object System.Windows.Forms.TextBox
$txtOutput.Location = New-Object System.Drawing.Point(10, 50)
$txtOutput.Size = New-Object System.Drawing.Size(540, 350)
$txtOutput.Multiline = $true
$txtOutput.ScrollBars = "Vertical"
$txtOutput.BackColor = [System.Drawing.Color]::Black
$txtOutput.ForeColor = [System.Drawing.Color]::Yellow
$tabShell.Controls.Add($txtOutput)

$btnRun.Add_Click({
    try {
        $out = Invoke-Expression $txtCommand.Text | Out-String
        $txtOutput.Text = $out
    } catch {
        $txtOutput.Text = "Chyba: $_"
    }
})

# Inicializace
& $RefreshProcs
$form.Add_Shown({$form.Activate()})
[void]$form.ShowDialog()
