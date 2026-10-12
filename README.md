$ErrorActionPreference = 'Stop'
Set-Location $PSScriptRoot
$zip = Join-Path $PSScriptRoot 'portable.zip'
if (!(Test-Path $zip)) { throw 'Falta portable.zip' }
$payload = Join-Path $PSScriptRoot 'payload'
if (Test-Path $payload) { Remove-Item $payload -Recurse -Force }
New-Item -ItemType Directory -Path $payload | Out-Null
Add-Type -AssemblyName System.IO.Compression.FileSystem
# ZIP creado con rutas de Windows: normalizar separadores para extraer correctamente.
$archive = [IO.Compression.ZipFile]::OpenRead($zip)
try {
 foreach ($entry in $archive.Entries) {
  $name = $entry.FullName.Replace('\','/')
  if (!$name -or $name.EndsWith('/')) { continue }
  $dest = [IO.Path]::GetFullPath((Join-Path $payload $name))
  if (!$dest.StartsWith(([IO.Path]::GetFullPath($payload) + [IO.Path]::DirectorySeparatorChar),[StringComparison]::OrdinalIgnoreCase)) { throw "Ruta insegura: $name" }
  New-Item -ItemType Directory -Force -Path ([IO.Path]::GetDirectoryName($dest)) | Out-Null
  [IO.Compression.ZipFileExtensions]::ExtractToFile($entry,$dest,$true)
 }
} finally { $archive.Dispose() }
if (!(Test-Path (Join-Path $payload 'Matematicas vs. Zombis.exe'))) { throw 'No se encontro el ejecutable principal' }
if (!(Test-Path (Join-Path $payload 'resources\app.asar'))) { throw 'No se encontraron recursos Electron' }
$nsis = "${env:ProgramFiles(x86)}\NSIS\makensis.exe"
if (!(Test-Path $nsis)) { $nsis = "$env:ProgramFiles\NSIS\makensis.exe" }
if (!(Test-Path $nsis)) { throw 'Instale NSIS desde https://nsis.sourceforge.io/Download y vuelva a ejecutar este script.' }
& $nsis (Join-Path $PSScriptRoot 'instalador.nsi')
if ($LASTEXITCODE -ne 0) { throw "NSIS fallo: $LASTEXITCODE" }
Write-Host 'INSTALADOR CREADO:' (Join-Path $PSScriptRoot 'Matematicas_vs_Zombis_4.4.4_Instalador.exe')
