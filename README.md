@echo off
setlocal enabledelayedexpansion

if "%~1"=="" (
    echo Usage: check_align.cmd yourfile.so
    exit /b 1
)

set FILE=%~1
set /a OFFSET=0

:: Read ELF header e_phoff (program header offset) at offset 0x20 (32)
:: e_phentsize (size of each entry) at offset 0x36
:: e_phnum (number of entries) at offset 0x38

:: Use certutil to dump hex
certutil -dump %FILE% > dump.txt

for /f "tokens=1-2 delims=:" %%a in ('findstr /b "  0020" dump.txt') do (
    set EPHOFF=%%b
)

for /f "tokens=1-2 delims=:" %%a in ('findstr /b "  0030" dump.txt') do (
    set TEMP=%%b
    set EPHENTSIZE=!TEMP:~12,4!
    set EPHNUM=!TEMP:~16,4!
)

:: Parse e_phoff
set EPHOFF=!EPHOFF: =!
set EPHOFF=%EPHOFF:~6,2%%EPHOFF:~4,2%%EPHOFF:~2,2%%EPHOFF:~0,2%

:: Convert hex to decimal
set /a PH_OFFSET=0x%EPHOFF%
set /a PH_ENTRY_SIZE=0x%EPHENTSIZE%
set /a PH_NUM=0x%EPHNUM%

echo Program Header offset = %PH_OFFSET%
echo Entry size = %PH_ENTRY_SIZE%
echo Number of entries = %PH_NUM%

:: Now read p_align for each segment (offset + 0x20 into each header)
for /l %%i in (0,1,%PH_NUM%) do (
    set /a CURR_OFFSET=%PH_OFFSET% + %%i * %PH_ENTRY_SIZE% + 0x20
    call :readalign %CURR_OFFSET%
)
goto :eof

:readalign
set /a LINE=%1/16
set /a LINEOFF=%1%%16
set /a NEXTLINE=%LINE%+1

set LINEHEX=000%LINE%
set LINEHEX=!LINEHEX:~-4!

for /f "tokens=1-2 delims=:" %%a in ('findstr /b "  !LINEHEX!" dump.txt') do (
    set DATA=%%b
)

:: Read 4 bytes from position in line
set BYTES=!DATA:~%LINEOFF%,8!
echo Align at offset 0x%1 = 0x!BYTES!

goto :eof
