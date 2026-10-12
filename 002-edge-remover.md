# Edge Remover

```powershell
$tempPath = Join-Path $env:TEMP "Edge.bat"
Invoke-WebRequest -Uri "https://gh-proxy.org/https://github.com/ShadowWhisperer/Remove-MS-Edge/blob/main/Batch/Edge.bat?raw=true" -OutFile $tempPath
# Both.bat
# Invoke-WebRequest -Uri "https://gh-proxy.org/https://github.com/ShadowWhisperer/Remove-MS-Edge/blob/main/Batch/Both.bat?raw=true" -OutFile $tempPath
# Edge-Appx.bat
# Invoke-WebRequest -Uri "https://gh-proxy.org/https://github.com/ShadowWhisperer/Remove-MS-Edge/blob/main/Batch/Edge-Appx.bat?raw=true" -OutFile $tempPath

Start-Process -FilePath "cmd.exe" -ArgumentList "/c `"$tempPath`"" -Verb RunAs
```