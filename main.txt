@echo off
chcp 65001 >nul
setlocal enabledelayedexpansion

set scriptDir=%~dp0

REM Define o diretório de dados no AppData
set "appDataDir=%APPDATA%\TazzSoftwares\TazzVideoDownloader"
powershell -Command "if(-not(Test-Path '!appDataDir!')) { New-Item -ItemType Directory -Path '!appDataDir!' -Force | Out-Null }"

set "logFile=!appDataDir!\downloads.log"
if not exist "!logFile!" (
    break> "!logFile!"
)

set "configFile=!appDataDir!\config.ini"

if exist "%configFile%" (
    call :readConfig
) else (
    set downloadDir=%USERPROFILE%\Downloads
    set fileType=mp4
    set downloadPlaylist=0
    set historyEnabled=1
    set limitHistory=1

    call :saveConfig
)

for /f "delims=" %%A in ('echo prompt $E^| cmd') do set "ESC=%%A"
set "BLUE=%ESC%[94m"
set "GREEN=%ESC%[92m"
set "YELLOW=%ESC%[93m"
set "GRAY=%ESC%[90m"
set "RED=%ESC%[91m"
set "RESET=%ESC%[0m"
set "separator= |TVD> "

:initMenu
cls
echo !BLUE!===========================!RESET!
echo !BLUE!   TAZZ VIDEO DOWNLOADER   !RESET!
echo !BLUE!===========================!RESET!
echo.
echo !BLUE!------- MENU -------!RESET!
echo !BLUE![1]!RESET! Local de download: !YELLOW!!downloadDir!!RESET!
if "!fileType!"=="mp4" (
    echo !BLUE![2]!RESET! Tipo de arquivo: !YELLOW!Vídeo mp4!GRAY! / !GRAY!Áudio mp3!RESET!
) else (
    echo !BLUE![2]!RESET! Tipo de arquivo: !GRAY!Vídeo mp4!GRAY! / !YELLOW!Áudio mp3!RESET!
)
if "!downloadPlaylist!"=="1" (
    echo !BLUE![3]!RESET! Download playlist no YouTube: !GREEN!Ativado!GRAY! / !GRAY!Desativado!RESET!
) else (
    echo !BLUE![3]!RESET! Download playlist no YouTube: !GRAY!Ativado!GRAY! / !RED!Desativado!RESET!
)
echo.
echo !BLUE![00]!RESET! Acessar histórico
echo !BLUE![0]!RESET! Sair
echo !BLUE!--------------------!RESET!
echo.
echo !RED!^^!ATENÇÃO^^!:!RESET! Ao ativar download de playlist no YouTube, qualquer link de vídeo com playlist (PÚBLICA) baixará a playlist inteira em vez do vídeo individual.
echo !YELLOW!INFO:!RESET! Cole os links dos vídeos (separados por espaço) ou digite uma opção para editar o !BLUE!MENU!RESET!.
echo.
set "initMenuInput="
set /p initMenuInput="» !YELLOW!"

echo !RESET!

if "!initMenuInput!"=="1" goto mudarLocal
if "!initMenuInput!"=="2" goto mudarTipo
if "!initMenuInput!"=="3" goto mudarPlaylist
if "!initMenuInput!"=="00" goto history
if "!initMenuInput!"=="0" goto :eof

set links=!initMenuInput!

set links=!initMenuInput!
call :processDownload
goto initMenu

:processDownload
if "!links!"=="" (
    goto initMenu
)

if not exist "%downloadDir%" (
    cls
    echo !RED!========================================!RESET!
    echo !RED!ERRO: Caminho de download não encontrado!RESET!
    echo !RED!========================================!RESET!
    echo.
    echo Caminho: !YELLOW!%downloadDir%!RESET!
    echo.
    echo !GREEN!Pressione qualquer tecla para voltar...!RESET!
    pause >nul
    goto initMenu
)

cd /d "%downloadDir%"

setlocal enabledelayedexpansion
set counter=0
set "initMenuInput=!links!"

:processLinks
if "!initMenuInput:~0,1!"==" " (
    set "initMenuInput=!initMenuInput:~1!"
    goto processLinks
)

if "!initMenuInput!"=="" goto acabou

set "url="
set "pos=0"
:findSpaces
if "!initMenuInput:~%pos%,1!"=="" (
    set "url=!initMenuInput!"
    set "initMenuInput="
    goto processURL
)
if "!initMenuInput:~%pos%,1!"==" " (
    set "url=!initMenuInput:~0,%pos%!"
    set "initMenuInput=!initMenuInput:~%pos%!"
    goto processURL
)
set /a pos+=1
goto findSpaces

:processURL
if "!downloadPlaylist!"=="0" (
    if "!url:*youtube.com=!" neq "!url!" (
        for /f "tokens=1 delims=&" %%i in ("!url!") do set "url=%%i"
    )
)

:executeDownload
set /a counter+=1

echo.
echo !BLUE![!counter!]!RESET! Iniciando download: !url!
echo.

REM Lista os arquivos antes do download
rem powershell -NoProfile -Command "Get-ChildItem '%downloadDir%' -File | Select-Object -ExpandProperty Name | Out-File '%TEMP%\files_before.txt' -Encoding UTF8" 2>nul

if "!fileType!"=="mp3" (
    "!scriptDir!yt-dlp.exe" -x --audio-format mp3 "!url!"
)
if "!fileType!"=="mp4" (
    "!scriptDir!yt-dlp.exe" -f "bestvideo+bestaudio/best" --merge-output-format mp4 "!url!"
)

echo.
if errorlevel 1 (
    echo !BLUE![!counter!]!RESET! !RED!Erro ao baixar!RESET!
    if "!historyEnabled!"=="1" (
        for /f "delims=" %%i in ('powershell -NoProfile -Command "Get-Date -Format \"dd/MM/yyyy HH:mm:ss\""') do set "timestamp=%%i"
        echo !timestamp!!separator!ERRO!separator!!fileType!!separator!Nenhum!separator!!url!!separator!!downloadDir!>>"!logFile!"
    )
    echo.
    echo !BLUE!Deseja tentar novamente?!RESET!
    echo.
    echo !BLUE![1]!RESET! Sim
    echo !BLUE![2]!RESET! Não
    echo.
    :askToRetry
    set "retry="
    set /p retry="%ESC%[Digite uma opção: "
    if "!retry!"=="1" (
        set /a counter-=1
        goto executeDownload
    ) else if "!retry!"=="2" (
        goto processLinks
    ) else (
        REM Apaga 3 linhas
        echo !ESC![A!ESC![K!ESC![A!ESC![K!ESC!
        goto askToRetry
    )
) else (
    echo !BLUE![!counter!]!RESET! !GREEN!Download concluído com sucesso!RESET!
    if "!historyEnabled!"=="1" (
        for /f "delims=" %%i in ('powershell -NoProfile -Command "Get-Date -Format \"dd/MM/yyyy HH:mm:ss\""') do set "timestamp=%%i"
        
        for /f "delims=" %%j in ('powershell -NoProfile -Command "$f=Get-ChildItem '%downloadDir%' -File | Sort-Object LastWriteTime -Descending | Select-Object -First 1; if($f){$f.Name} else {Write-Host 'Nenhum'}"') do set "fileName=%%j"

        echo !timestamp!!separator!SUCESSO!separator!!fileType!!separator!!fileName!!separator!!url!!separator!!downloadDir!>>"!logFile!"
    )
)
rem if exist "%TEMP%\files_before.txt" del "%TEMP%\files_before.txt"

goto processLinks

:acabou
echo.

echo !BLUE!========================================!RESET!
echo !BLUE!        Arquivo(s) processado(s)        !RESET!
echo !BLUE!========================================!RESET!
echo.
echo Pressione qualquer tecla para continuar...
pause >nul

goto :eof

:history

del /f /q "%TEMP%\tazz_history.tmp" 2>nul
if "!limitHistory!"=="1" (
    powershell -NoProfile -Command "if(Test-Path '!logFile!'){$l=Get-Content '!logFile!' -Encoding UTF8;if($l){($l|select -Last 10|%%{$i++;\"$i@@$_\"})|Out-File -FilePath '%TEMP%\tazz_history.tmp' -Encoding UTF8}}"
) else (
    powershell -NoProfile -Command "if(Test-Path '!logFile!'){$l=Get-Content '!logFile!' -Encoding UTF8;if($l){$l|%%{$i++;\"$i@@$_\"}|Out-File -FilePath '%TEMP%\tazz_history.tmp' -Encoding UTF8}}"
)
if not exist "%TEMP%\tazz_history.tmp" goto historyEmpty
for %%Z in ("%TEMP%\tazz_history.tmp") do if %%~zZ==0 goto historyEmpty

goto historyList

:historyEmpty
cls
echo !BLUE!===========================!RESET!
echo !BLUE!       HISTÓRICO           !RESET!
echo !BLUE!===========================!RESET!
echo.
echo !YELLOW!Nenhum registro no histórico.!RESET!
echo.
echo !BLUE!---------------------------!RESET!
echo.
echo !BLUE![00]!RESET! Mais opções
echo !BLUE![0]!RESET! Voltar
echo.
set "sel="
set /p sel="Digite o número de uma opção: "
if "!sel!"=="00" goto historyMoreOptions
if "!sel!"=="0" goto initMenu
goto historyEmpty

:historyList
cls
echo !BLUE!===========================!RESET!
echo !BLUE!       HISTÓRICO           !RESET!
echo !BLUE!===========================!RESET!
echo.
for /f "usebackq tokens=1* delims=@" %%a in ("%TEMP%\tazz_history.tmp") do (
    set "idx=%%a"
    set "line=%%b"
    if "%%b" neq "" call :renderHistLine
)
echo.
echo !BLUE!---------------------------!RESET!
echo.
echo !BLUE![00]!RESET! Mais opções
echo !BLUE![0]!RESET! Voltar
echo.
set "sel="
set /p sel="Digite o número de uma opção: "
if "!sel!"=="00" goto historyMoreOptions
if "!sel!"=="0" goto initMenu
set "chosen="
for /f "usebackq tokens=1* delims=@" %%a in ("%TEMP%\tazz_history.tmp") do (
    if "%%a"=="!sel!" set "chosen=%%b"
)
if "!chosen!"=="" goto history
set "chosenMod=!chosen:%separator%=|!"
for /f "tokens=1-6 delims=|" %%a in ("!chosenMod!") do (
    set "ch_dt=%%a"
    set "ch_status=%%b"
    set "ch_type=%%c"
    set "ch_filename=%%d"
    set "ch_url=%%e"
    set "ch_location=%%f"
)

:showAactInputonScreen
cls
echo !BLUE!===========================!RESET!
echo !BLUE!      AÇÃO NO REGISTRO     !RESET!
echo !BLUE!===========================!RESET!
echo.
echo Data/Hora: !YELLOW!!ch_dt!!RESET!
if /I "!ch_status!"=="SUCESSO" (
    echo Status: !GREEN!!ch_status!!RESET!
) else if /I "!ch_status!"=="ERRO" (
    echo Status: !RED!!ch_status!!RESET!
) else (
    echo Status: !YELLOW!!ch_status!!RESET!
)
echo Tipo: !YELLOW!!ch_type!!RESET!
echo Local: !YELLOW!!ch_location!!RESET!
if "!ch_filename!"=="Nenhum" (
    echo Arquivo: !RED!!ch_filename!!RESET!
) else (
    echo Arquivo: !YELLOW!!ch_filename!!RESET!
)
echo URL: !BLUE!!ch_url!!RESET!
echo.
echo !BLUE![1]!RESET! Apagar registro
echo !BLUE![2]!RESET! Executar novamente
echo !BLUE![3]!RESET! Copiar link
echo !BLUE![4]!RESET! Acessar link
echo !BLUE![5]!RESET! Abrir pasta de download
echo !BLUE![6]!RESET! Deletar arquivo
echo !BLUE![7]!RESET! Abrir arquivo
echo.
echo !BLUE![0]!RESET! Voltar
echo.
set "actInput="
set /p actInput="Digite o número de uma opção: "
if "!actInput!"=="1" goto hist_delete
if "!actInput!"=="2" call :hist_rerun
if "!actInput!"=="3" call :hist_copylink
if "!actInput!"=="4" call :hist_openlink
if "!actInput!"=="5" call :hist_openfolder
if "!actInput!"=="6" call :hist_deletefile
if "!actInput!"=="7" call :hist_openfile
if "!actInput!"=="0" goto history

goto showAactInputonScreen

:renderHistLine
set "lineMod=!line:%separator%=|!"
for /f "tokens=1-6 delims=|" %%a in ("!lineMod!") do (
    set "dt=%%a"
    set "st=%%b"
    set "tp=%%c"
    set "fn=%%d"
    set "vl=%%e"
    set "lc=%%f"
)
if /I "!st!"=="SUCESSO" (
    echo !BLUE![!idx!]!RESET! !dt! - !GREEN!!st!!RESET! - !tp! - !fn!
) else if /I "!st!"=="ERRO" (
    echo !BLUE![!idx!]!RESET! !dt! - !RED!!st!!RESET! - !tp! - !vl!
) else (
    echo !BLUE![!idx!]!RESET! !dt! - !YELLOW!!st!!RESET! - !tp! - !vl!
)
goto :eof

:hist_delete
set "lineToDelete=!chosen!"
powershell -NoProfile -Command "$line='%lineToDelete%'.Replace(\"'\",\"''\");$content=Get-Content '!logFile!' -Encoding UTF8;$content|Where-Object{$_ -ne $line}|Set-Content '!logFile!' -Encoding UTF8"
echo !GREEN!Registro apagado.!RESET!
pause>nul
goto history

:hist_copylink
echo !ch_url!| clip
echo !GREEN!Link copiado para a área de transferência!RESET!
pause>nul
goto :eof

:hist_openlink
start "" "!ch_url!"
echo !GREEN!Link aberto no navegador padrão!RESET!
pause>nul
goto :eof

:hist_openfolder
if exist "!ch_location!" (
    start "" "!ch_location!"
    echo !GREEN!Pasta aberta no explorador!RESET!
) else (
    echo !RED!Pasta não encontrada!RESET!
)
pause>nul
goto :eof

:hist_deletefile
if "!ch_filename!"=="Nenhum" (
    echo !RED!Nenhum arquivo para deletar!RESET!
    pause>nul
    goto :eof
)
set "fullPath=!ch_location!\!ch_filename!"
if exist "!fullPath!" (
    del "!fullPath!"
    echo !GREEN!Arquivo deletado!RESET!
) else (
    echo !RED!Arquivo não encontrado!RESET!
)
pause>nul
goto :eof

:hist_openfile
if "!ch_filename!"=="Nenhum" (
    echo !RED!Nenhum arquivo para abrir!RESET!
    pause>nul
    goto :eof
)
if exist "!ch_location!\!ch_filename!" (
    start "" "!ch_location!\!ch_filename!"
    echo !GREEN!Arquivo aberto no explorador!RESET!
) else (
    echo !RED!Arquivo não encontrado!RESET!
)
pause>nul
goto :eof

:hist_rerun
set "initMenuInput="
set "url=%ch_url%"
cd /d "%ch_location%"
set "fileType=%ch_type%"
set "downloadPlaylist=1"

set counter= !sel!
set /a counter-=1
call :executeDownload
call :readConfig
goto :eof

:historyMoreOptions
cls
echo !BLUE!===========================!RESET!
echo !BLUE!        MAIS OPÇÕES        !RESET!
echo !BLUE!===========================!RESET!
echo.
if "!historyEnabled!"=="1" (
    echo !BLUE![1]!RESET! Histórico de downloads: !GREEN!Ativado!GRAY! / !GRAY!Desativado!RESET!
) else (
    echo !BLUE![1]!RESET! Histórico de downloads: !GRAY!Ativado!GRAY! / !RED!Desativado!RESET!
)
if "!limitHistory!"=="1" (
    echo !BLUE![2]!RESET! Mostrar apenas 10 últimos: !GREEN!Ativado!GRAY! / !GRAY!Desativado!RESET!
) else if "!limitHistory!"=="0" (
    echo !BLUE![2]!RESET! Mostrar apenas 10 últimos: !GRAY!Ativado!GRAY! / !RED!Desativado!RESET!
)
echo !BLUE![3]!RESET! Apagar histórico
echo.
echo !BLUE![0]!RESET! Voltar
echo.
set "mo="
set /p mo="Digite o número de uma opção: "
if "!mo!"=="1" goto hist_toggle
if "!mo!"=="2" goto hist_toggle_limit
if "!mo!"=="3" goto hist_clear
if "!mo!"=="0" goto history
goto historyMoreOptions

:hist_clear
if exist "!logFile!" del "!logFile!"
echo !YELLOW!Histórico apagado.!RESET!
pause>nul
goto history

:hist_toggle
cls
if "!historyEnabled!"=="1" (
    set "historyEnabled=0"
) else (
    set "historyEnabled=1"
)
call :saveConfig
goto historyMoreOptions

:hist_toggle_limit
cls
if "!limitHistory!"=="1" (
    set "limitHistory=0"
) else (
    set "limitHistory=1"
)
call :saveConfig
goto historyMoreOptions

:mudarLocal
cls
echo !BLUE!===========================!RESET!
echo !BLUE!  MUDAR LOCAL DE DOWNLOAD !RESET!
echo !BLUE!===========================!RESET!
echo.
echo Local atual: !YELLOW!%downloadDir%!RESET!
echo.
echo !GRAY!Abrindo seletor de diretórios...!RESET!
echo.

for /f "delims=" %%i in ('powershell -Command "Add-Type -AssemblyName System.Windows.Forms; $folderBrowser = New-Object System.Windows.Forms.FolderBrowserDialog; $folderBrowser.Description = 'Selecione o local de download'; $folderBrowser.SelectedPath = '%downloadDir%'; if ($folderBrowser.ShowDialog() -eq 'OK') { $folderBrowser.SelectedPath }"') do set novoLocal=%%i

if NOT "!novoLocal!"!=="" (
    set downloadDir=!novoLocal!
    call :saveConfig
)
goto initMenu

:mudarTipo
cls
if "!fileType!"=="mp4" (
    set fileType=mp3
) else (
    set fileType=mp4
)

call :saveConfig
goto initMenu

:mudarPlaylist
cls
if "!downloadPlaylist!"=="0" (
    set downloadPlaylist=1
) else (
    set downloadPlaylist=0
)
call :saveConfig
goto initMenu

:readConfig
for /f "usebackq tokens=1,2 delims==" %%a in ("!configFile!") do (
    if "%%a"=="downloadDir" set downloadDir=%%b
    if "%%a"=="fileType" set fileType=%%b
    if "%%a"=="downloadPlaylist" set downloadPlaylist=%%b
    if "%%a"=="historyEnabled" set historyEnabled=%%b
    if "%%a"=="limitHistory" set limitHistory=%%b
)
goto :eof

:saveConfig
(
    echo [SETTINGS]
    echo downloadDir=!downloadDir!
    echo fileType=!fileType!
    echo downloadPlaylist=!downloadPlaylist!
    echo historyEnabled=!historyEnabled!
    echo limitHistory=!limitHistory!
) > "!configFile!"
goto :eof