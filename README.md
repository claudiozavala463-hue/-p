Clear-Host

Write-Host ""
Write-Host "🛡️ AUDITORIA AVANZADA DE RED" -ForegroundColor Cyan
Write-Host "========================================================"
Write-Host ""

$fecha = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"

$resultados = @()

$conexiones = Get-NetTCPConnection -State Established -ErrorAction SilentlyContinue

foreach ($con in $conexiones) {

    try {
        $proceso = Get-Process -Id $con.OwningProcess -ErrorAction Stop
        $nombreProceso = $proceso.ProcessName
    }
    catch {
        $nombreProceso = "Desconocido"
    }

    try {
        $dns = ([System.Net.Dns]::GetHostEntry($con.RemoteAddress)).HostName
    }
    catch {
        $dns = "Sin resolver"
    }

    if (
        $con.RemoteAddress -match '^10\.' -or
        $con.RemoteAddress -match '^192\.168\.' -or
        $con.RemoteAddress -match '^172\.(1[6-9]|2[0-9]|3[0-1])\.'
    ) {
        $tipoIP = "Privada"
    }
    else {
        $tipoIP = "Publica"
    }

    $resultado = [PSCustomObject]@{
        Fecha          = Get-Date
        Proceso        = $nombreProceso
        PID            = $con.OwningProcess
        IP_Local       = $con.LocalAddress
        Puerto_Local   = $con.LocalPort
        IP_Remota      = $con.RemoteAddress
        Puerto_Remoto  = $con.RemotePort
        DNS            = $dns
        Tipo_IP        = $tipoIP
        Estado         = $con.State
    }

    $resultados += $resultado
}

Write-Host ""
Write-Host "📡 CONEXIONES ACTIVAS"
Write-Host "--------------------------------------------------------"

$resultados | Sort-Object Proceso | Format-Table -AutoSize

Write-Host ""
Write-Host "⚠️ POSIBLES CONEXIONES SOSPECHOSAS"
Write-Host "--------------------------------------------------------"

$sospechosas = $resultados | Where-Object {
    $_.Tipo_IP -eq "Publica" -and $_.DNS -eq "Sin resolver"
}

if ($sospechosas) {
    $sospechosas | Format-Table -AutoSize
}
else {
    Write-Host "No se detectaron conexiones sospechosas." -ForegroundColor Green
}

Write-Host ""
Write-Host "🌐 TABLA ARP (IP ↔ MAC)"
Write-Host "--------------------------------------------------------"

try {
    $vecinos = Get-NetNeighbor |
    Where-Object {
        $_.IPAddress -notmatch "^ff" -and
        $_.LinkLayerAddress -ne ""
    } |
    Select-Object IPAddress, LinkLayerAddress, State

    $vecinos | Format-Table -AutoSize
}
catch {
    Write-Host "No fue posible obtener vecinos ARP."
}

Write-Host ""
Write-Host "💾 EXPORTANDO REPORTES..."
Write-Host "--------------------------------------------------------"

$csvConexiones = "$env:USERPROFILE\Desktop\Conexiones_$fecha.csv"
$csvARP = "$env:USERPROFILE\Desktop\ARP_$fecha.csv"

$resultados | Export-Csv $csvConexiones -NoTypeInformation -Encoding UTF8

try {
    $vecinos | Export-Csv $csvARP -NoTypeInformation -Encoding UTF8
}
catch {}

Write-Host ""
Write-Host "✅ AUDITORIA FINALIZADA" -ForegroundColor Green
Write-Host ""
Write-Host "Conexiones guardadas en:"
Write-Host $csvConexiones -ForegroundColor Yellow
Write-Host ""
Write-Host "ARP/MAC guardadas en:"
Write-Host $csvARP -ForegroundColor Yellow
Write-Host ""
Write-Host "========================================================"
