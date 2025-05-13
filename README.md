# Ripper - HackMyVM (Easy)

![Ripper.png](Ripper.png)

## Übersicht

*   **VM:** Ripper
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Ripper)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 12. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Ripper_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Ripper" zu erlangen. Der Weg dorthin begann mit der Entdeckung einer Backup-Datei eines privaten SSH-Schlüssels (`id_rsa.bak`) auf dem Webserver. Nach dem Knacken der Schlüssel-Passphrase (`bananas`) mit `RSAcrack.sh` (einem Wrapper für `ssh2john`/`john`) wurde SSH-Zugriff als Benutzer `jack` erlangt. Durch Ausführung des Enumerationsskripts `Linpeas.sh` wurden die Credentials (`Il0V3lipt0n1c3t3a`) für den Benutzer `helder` gefunden. Die finale Rechteausweitung zu Root gelang durch Ausnutzung eines unsicheren Cronjobs. Dieser Cronjob führte `nc localhost 10000` aus und schrieb die erste Zeile der Antwort in eine Datei. Wenn eine andere Datei (`/home/helder/passwd.txt`) denselben Inhalt hatte wie eine Datei unter `/root/.local/`, wurde das SUID-Bit auf einer Binärdatei gesetzt, deren Name über den Netcat-Port empfangen wurde. Durch Erstellen eines Symlinks, Senden von "bash" an den Port und anschließendes Ausführen von `bash -p` wurden Root-Rechte erlangt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `curl`
*   `wget`
*   `ssh-keyscan`
*   `RSAcrack.sh` (Wrapper für `ssh2john`/`john`)
*   `ssh`
*   `sudo` (versucht)
*   `ls`
*   `cat`
*   `hydra` (versucht)
*   `dig` (impliziert für Hostnamenauflösung)
*   `su`
*   `Linpeas.sh`
*   `chmod`
*   `pspy64`
*   `which`
*   `ln`
*   `echo`
*   `nc` (netcat)
*   Standard Linux-Befehle (`bash`, `cd`, `id`, `find`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Ripper" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.143) mit `arp-scan` identifiziert. Hostname `ripper.vm` (oder `otte.hmv`) in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 7.9p1) und Port 80 (HTTP, Apache 2.4.38).
    *   `gobuster` auf Port 80 fand `index.html`, `/server-status` (403) und kritischerweise `/id_rsa.bak`.
    *   Die Datei `/staff_statements.txt` (Herkunft unklar) enthielt einen Hinweis auf "old ssh connection files".

2.  **Initial Access (SSH als `jack` via SSH Key Crack):**
    *   Die Datei `id_rsa.bak` wurde von `http://192.168.2.143/id_rsa.bak` heruntergeladen. Es handelte sich um einen verschlüsselten privaten SSH-Schlüssel.
    *   Der Benutzername `jack` wurde vom Login-Bildschirm der VM entnommen.
    *   Mittels `./RSAcrack.sh -w rockyou.txt -k id_rsa.bak` (lokal umbenannt zu `idroot`) wurde die Passphrase des Schlüssels zu `bananas` geknackt.
    *   Erfolgreicher SSH-Login als `jack` mit dem privaten Schlüssel und der Passphrase `bananas`.

3.  **Privilege Escalation (von `jack` zu `helder` via Credential Leak):**
    *   Als `jack` wurde `sudo -l` versucht (erfolglos).
    *   Das Enumerationsskript `Linpeas.sh` wurde auf das Zielsystem hochgeladen und ausgeführt.
    *   Linpeas fand die Credentials für den Benutzer `helder`: `Il0V3lipt0n1c3t3a`.
    *   Mit `su helder` und dem gefundenen Passwort wurde erfolgreich zum Benutzer `helder` gewechselt.
    *   Die User-Flag (`5c38d7d08c687355cb0ae3b6025cbe99`) wurde in `/home/helder/user.txt` gefunden.

4.  **Privilege Escalation (von `helder` zu `root` via Cronjob SUID Exploit):**
    *   Als `helder` wurde mittels `pspy64` ein minütlich als Root laufender Cronjob identifiziert.
    *   Der Cronjob führte `/bin/sh -c nc -vv -q 1 localhost 10000 > /root/.local/out` aus und prüfte danach, ob der Inhalt von `/root/.local/helder.txt` identisch mit `/home/helder/passwd.txt` ist. Wenn ja, wurde `chmod +s "/usr/bin/$(cat /root/.local/out)"` ausgeführt.
    *   Der Exploit-Pfad:
        1.  Ein symbolischer Link wurde erstellt: `ln -s /root/.local/helder.txt /home/helder/passwd.txt`.
        2.  Ein Netcat-Listener wurde gestartet, der auf Port 10000 lauscht und bei Verbindung den String "bash" sendet: `echo 'bash' | nc -lvnp 10000`.
        3.  Nachdem der Cronjob verbunden und "bash" empfangen hatte, wurde `/root/.local/out` mit "bash" überschrieben.
        4.  Die `if`-Bedingung war erfüllt (durch den Symlink).
        5.  `chmod +s /usr/bin/bash` wurde als Root ausgeführt.
    *   Das SUID-Bit auf `/usr/bin/bash` wurde bestätigt (`ls -al /usr/bin/bash` zeigte `-rwsr-sr-x`).
    *   Mittels `bash -p` wurde eine Root-Shell erlangt (`euid=0(root)`).
    *   Die Root-Flag (`e28f578a17a5f99c0381c4b689d96f9f`) wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Exponierter privater SSH-Schlüssel (Backup):** Eine Backup-Datei eines privaten SSH-Schlüssels (`id_rsa.bak`) war über den Webserver öffentlich zugänglich.
*   **Passwort-Cracking (SSH-Key):** Die Passphrase eines verschlüsselten privaten SSH-Schlüssels wurde mit `RSAcrack.sh` (John the Ripper) geknackt.
*   **Credential Leak durch Enumerationsskript:** `Linpeas.sh` fand Klartext-Credentials für einen anderen Benutzer.
*   **Unsicherer Cronjob (Wildcard-ähnliche SUID-Setzung):** Ein Cronjob, der als Root lief, nahm Eingaben über einen Netzwerkport (Netcat) entgegen und verwendete diese Eingabe, um das SUID-Bit auf einer System-Binary (`/usr/bin/...`) zu setzen. Dies wurde ausgenutzt, um `/usr/bin/bash` SUID zu machen.
*   **SUID Exploit (bash -p):** Eine SUID-gesetzte Bash-Executable wurde mit der Option `-p` ausgeführt, um Root-Rechte zu erlangen.
*   **Symbolic Link Abuse:** Ein Symlink wurde verwendet, um eine Dateivergleichsbedingung im Cronjob zu erfüllen.

## Flags

*   **User Flag (`/home/helder/user.txt`):** `5c38d7d08c687355cb0ae3b6025cbe99`
*   **Root Flag (`/root/root.txt`):** `e28f578a17a5f99c0381c4b689d96f9f`

## Tags

`HackMyVM`, `Ripper`, `Easy`, `SSH Key Leak`, `RSAcrack`, `Linpeas`, `Credential Leak`, `Cronjob Exploit`, `SUID Exploit`, `bash -p`, `Symlink Abuse`, `Linux`, `Web`, `Privilege Escalation`, `Apache`
