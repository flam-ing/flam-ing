---
name: kakaotalk-pc-wine
description: Install and run the Windows KakaoTalk PC app on Linux with Wine 10.0. Use when setting up KakaoTalk on a Linux machine, repairing a failed Wine/KakaoTalk installation, or operating the installed app (launch, verify, automate the GUI).
---

# KakaoTalk PC on Linux via Wine

Verified working recipe (November 2026): Wine 10.0 + fresh 64-bit prefix + official installer (KakaoTalk_Setup.exe 26.9.2) with silent install. Renders the full Korean login window and stays stable.

Do NOT use Wine 11.x: the installer hangs even in silent mode, and the app crashes during startup when its bundled SQLCipher/OpenSSL crypto fails to initialize (RAND_bytes "entropy source strength too weak", exception c0000409) before the login window ever appears. Wine 10.0 passes the same path cleanly.

## Prerequisites

- x86_64 Linux. Root is convenient but not required.
- An X display for the whole session: a real desktop, or Xvfb (`Xvfb :3 -screen 0 1280x800x24 &`). Window creation fails with "no driver could be loaded" if no display server is actually alive when the app starts.
- Wine 10.0 with both wine64 and wine32 support (e.g. distro packages wine64 + wine32, or the portable Kron4ek Wine-Builds `wine-10.0-amd64` tarball, which runs 32-bit apps via WoW64).
- 32-bit system libraries (glibc, gnutls, freetype, fontconfig, libX11 and friends). No-root option: download the i386 .deb packages, extract them into a local dir, and point the 32-bit wine launcher at the local loader with `patchelf --set-interpreter`, with `LD_LIBRARY_PATH` set to the extracted lib dirs.
- A Korean font: Noto Sans CJK KR or Nanum.
- ko_KR.UTF-8 locale available (generate locally with `localedef` into a LOCPATH dir if you have no root).

## Setup steps

1. Fresh 64-bit prefix:
   ```
   WINEPREFIX=$HOME/.wine-kakao WINEDLLOVERRIDES="mscoree=d;mshtml=d" wineboot --init
   ```
   The `WINEDLLOVERRIDES` matters: without it, first-time init blocks silently on the Wine Mono/Gecko download dialog. An interrupted init leaves a wedged, half-populated prefix (empty syswow64, 32-bit apps die with "could not load kernel32.dll") - delete the prefix and redo the init in one uninterrupted run.

2. Korean fonts (without this step all Korean text renders as boxes):
   - Copy the font file into `$WINEPREFIX/drive_c/windows/Fonts/`
   - Register it:
     ```
     wine reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Fonts" /v "Noto Sans CJK KR (OpenType)" /t REG_SZ /d NotoSansCJKkr-Regular.otf /f
     wine reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes" /v "Malgun Gothic" /t REG_SZ /d "Noto Sans CJK KR" /f
     wine reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes" /v "NanumGothic" /t REG_SZ /d "Noto Sans CJK KR" /f
     wine reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes" /v "Gulim" /t REG_SZ /d "Noto Sans CJK KR" /f
     wineboot --update
     ```

3. Download the official installer from Kakao's CDN:
   ```
   curl -O https://lk.kakaocdn.net/talkpc/talk/win32/KakaoTalk_Setup.exe
   ```

4. Silent install (completes in ~15 seconds on Wine 10.0):
   ```
   WINEPREFIX=$HOME/.wine-kakao wine KakaoTalk_Setup.exe /S
   ```
   Installs to `C:\Program Files (x86)\Kakao\KakaoTalk\KakaoTalk.exe`.

5. Launch with the display, locale, and prefix set:
   ```
   WINEPREFIX=$HOME/.wine-kakao LANG=ko_KR.UTF-8 LC_ALL=ko_KR.UTF-8 DISPLAY=:3 \
     wine "C:\Program Files (x86)\Kakao\KakaoTalk\KakaoTalk.exe" &
   ```

6. Verify visually: screenshot the display (`import -window root shot.png` from ImageMagick). Expect the yellow KakaoTalk window with 카카오계정/비밀번호 fields, 로그인 and QR코드 로그인 buttons. A small empty tray bubble window alongside it is a harmless Wine tray artifact.

## Login

- Password login: enter the Kakao account email and password, then approve the new-device verification prompt that appears in the mobile KakaoTalk app. Keep "자동 로그인" (auto-login) checked so later launches skip credentials.
- QR login: click QR코드 로그인 and scan the code with the mobile KakaoTalk scanner. The QR must be visible on a different screen than the phone doing the scanning, so this needs a second device/monitor.
- Login state lives in the prefix. If the prefix directory is deleted, the login is gone.

## GUI automation notes

- Use `xdotool` against the X display for clicks and typing (window name "KakaoTalk" / 카카오톡).
- Always verify state with a fresh screenshot before and after interacting; never trust window IDs or sleeps alone.

## Troubleshooting

- Installer or app "hangs" on first run: prefix init was interrupted or the Mono/Gecko dialog is blocking. Check for the override, and if the prefix is half-built, wipe and re-init.
- "could not load kernel32.dll (c0000135)" for 32-bit apps: syswow64 is empty - same wedged-prefix cause, wipe and re-init.
- Every wine launch hangs silently: a leftover wedged wineserver session. Kill all wine processes (`wineserver -k` or pkill) and retry.
- Crash with exception c0000409 / SQLCipher entropy error at startup: you are on Wine 11.x. Switch to Wine 10.0.
- Korean text as boxes: font not registered or FontSubstitutes missing (step 2).
