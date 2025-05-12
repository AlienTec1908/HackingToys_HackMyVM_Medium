# Hackingtoys - HackMyVM (Medium) - Writeup

![HackingToys Icon](HackingToys.png)

Dieses Repository enthält einen zusammengefassten Bericht über die Kompromittierung der HackMyVM-Maschine "Hackingtoys" (Schwierigkeitsgrad: Medium).

## Metadaten

*   **Maschine:** Hackingtoys (HackMyVM - Medium)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=HackingToys](https://hackmyvm.eu/machines/machine.php?vm=HackingToys)
*   **Autor des Writeups:** DarkSpirit
*   **Datum:** 4. September 2024
*   **Original Writeup:** [https://alientec1908.github.io/HackingToys_HackMyVM_Medium/](https://alientec1908.github.io/HackingToys_HackMyVM_Medium/)

## Zusammenfassung des Angriffspfads

Die initiale Erkundung mit `nmap` offenbarte einen SSH-Dienst (Port 22) und einen Ruby on Rails / Puma Webserver auf Port 3000 (HTTPS). Die manuelle Untersuchung der Webanwendung deckte eine Suchfunktion auf (`/search`), bei der der GET-Parameter `message` direkt in der Antwort reflektiert wurde. Dies führte zur Entdeckung einer Server-Side Template Injection (SSTI)-Schwachstelle in der ERB-Template-Engine.

Durch Einfügen eines ERB-Payloads (`<%= \`id\` %>`) wurde Codeausführung als Benutzer `lidia` bestätigt. Eine Ruby-Reverse-Shell-Payload wurde dann über die SSTI-Schwachstelle ausgeführt, um initialen Zugriff als `lidia` zu erlangen.

Als `lidia` wurde mittels `ss` ein lokaler FastCGI-Dienst (PHP-FPM) auf Port 9000 identifiziert. Über eine SSH-Portweiterleitung wurde dieser Dienst vom Angreifer-System aus erreichbar gemacht. Ein angepasstes Skript (`fast_CGI.sh`), das `cgi-fcgi` und die `auto_prepend_file`-Direktive nutzte, ermöglichte die Ausführung von PHP-Code im Kontext des PHP-FPM-Prozesses, der als Benutzer `dodi` lief. So wurde eine weitere Reverse Shell als `dodi` etabliert (Laterale Bewegung).

Als `dodi` wurde die User-Flag gelesen. Die `sudo -l`-Abfrage zeigte, dass `dodi` das Skript `/usr/local/bin/rvm_rails.sh` als `root` ohne Passwort ausführen durfte. Dieses Skript rief das Binary `/usr/local/rvm/gems/ruby-3.1.0/bin/rails` auf. Da der Benutzer `lidia` (aufgrund von Gruppenzugehörigkeiten oder Dateiberechtigungen) Schreibrechte auf dieses `rails`-Binary hatte, wurde es (als `lidia`) durch ein Ruby-Reverse-Shell-Skript ersetzt. Anschließend wurde als `dodi` der `sudo`-Befehl (`sudo /usr/local/bin/rvm_rails.sh`) ausgeführt. Dies triggerte das manipulierte `rails`-Skript, das mit Root-Rechten eine Reverse Shell zum Angreifer startete. Schließlich wurde die Root-Flag gelesen.

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `vi` / `nano` (Editor)
*   `nmap`
*   `nikto`
*   `curl`
*   `nc` (netcat)
*   Web Browser
*   Burp Suite (Implied/Manual Request for SSTI testing)
*   urlencoder.io (Implied for URL encoding payloads)
*   `stty`
*   `ls`, `cd`
*   `cat`
*   `wget`
*   `python3 -m http.server`
*   `ssh`
*   `ssh-keygen` (implied for SSH key setup for dodi)
*   `ss`
*   `find`
*   `cgi-fcgi` (im Exploit-Skript)
*   `mktemp` (im Exploit-Skript)
*   `env` (im Exploit-Skript)
*   `base64` (im Exploit-Skript)
*   `grep`
*   `chmod`
*   `mkdir`
*   `sudo`
*   `mv`
*   `ruby`
*   Ruby Modules: `require`, `socket`, `syscall`, `exec`

## Angriffsschritte (Übersicht)

1.  **Reconnaissance:** Ziel-IP (`arp-scan`), Hostname (`hackingtoys.hmv` in `/etc/hosts`). `nmap` -> Port 22 (SSH), Port 3000 (HTTPS - Ruby/Puma).
2.  **Web Enumeration (Port 3000):** `nikto` (wenig Erfolg). Manuelle Browser-Untersuchung -> `/search` mit Parameter `message`.
3.  **SSTI Discovery:** Testen des `message`-Parameters -> ERB SSTI mit `<%= 7+8 %>` bestätigt. RCE mit `<%= \`id\` %>` als `lidia`.
4.  **Initial Access (`lidia`):** `nc`-Listener starten. Ruby-Reverse-Shell-Payload (`<%= \`nc -e ...\` %>`) via SSTI in `message`-Parameter ausführen -> Shell als `lidia`. Stabilisieren (`stty`).
5.  **Lateral Movement Vector (`lidia` -> `dodi`):** `ss -altpn` als `lidia` -> FastCGI/PHP-FPM auf `127.0.0.1:9000`. SSH Port Forward (`ssh -L 9000:127.0.0.1:9000 lidia@...`).
6.  **FastCGI Exploit:** `fast_CGI.sh`-Skript (nutzt `cgi-fcgi`, `auto_prepend_file`) anpassen (Payload: `whoami` -> `dodi`). `nc`-Listener starten. Payload auf Reverse Shell ändern. Exploit gegen lokalen Port 9000 ausführen -> Shell als `dodi`. Stabilisieren.
7.  **User Flag:** `cat /home/dodi/user.txt`.
8.  **Privilege Escalation Vector (`dodi` -> `root`):** `sudo -l` als `dodi` -> `(ALL : ALL) NPASSWD: /usr/local/bin/rvm_rails.sh`. Skript `/usr/local/bin/rvm_rails.sh` analysieren -> führt `/usr/local/rvm/gems/ruby-3.1.0/bin/rails` aus.
9.  **Binary Hijack (als `lidia`):** Als `lidia` Ruby-Reverse-Shell-Skript erstellen, in `rails` umbenennen. Original `/usr/local/rvm/gems/ruby-3.1.0/bin/rails` damit überschreiben (Schreibrechte für `lidia` nötig).
10. **Privilege Escalation Execution (als `dodi`):** `nc`-Listener (für Root-Shell) starten. Als `dodi`: `sudo -u root /usr/local/bin/rvm_rails.sh` ausführen.
11. **Root Access:** Root-Shell erhalten.
12. **Root Flag:** `cat /root/root.txt`.

## Flags

*   **User Flag (`/home/dodi/user.txt`):** `b075b24bdb11990e185c32c43539c39f`
*   **Root Flag (`/root/root.txt`):** `64aa5a7aaf42af74ee6b59d5ac5c1509`

---

## Disclaimer

Die hier dargestellten Informationen und Techniken dienen ausschließlich Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Methoden dürfen nur auf Systemen angewendet werden, für die eine ausdrückliche Genehmigung vorliegt (z.B. in CTF-Umgebungen wie HackMyVM, Penetrationstests mit schriftlicher Erlaubnis). Das unbefugte Eindringen in fremde Computersysteme ist illegal und strafbar. Die Autoren übernehmen keine Haftung für missbräuchliche Verwendung der bereitgestellten Informationen. Handeln Sie stets legal und ethisch verantwortlich.

---
