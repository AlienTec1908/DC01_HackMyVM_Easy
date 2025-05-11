# DC01 - HackMyVM (Easy)

![DC01.png](DC01.png)

## Übersicht

*   **VM:** DC01
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=DC01) (Hinweis: Auch als Vulnhub-Maschine bekannt)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 9. August 2024
*   **Original-Writeup:** https://alientec1908.github.io/DC01_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, User- und Root-Zugriff auf der Maschine "DC01", einem Active Directory Domain Controller, zu erlangen. Die Erkundung identifizierte diverse AD-Dienste (LDAP, Kerberos, SMB). Die Enumeration über SMB mit dem Gastkonto ermöglichte das Auslesen von Benutzernamen mittels RID-Cycling. Ein Passwort-Spraying-Angriff, bei dem Benutzername als Passwort verwendet wurde, deckte das Konto `ybob317` auf, dessen Passwort (`ybob317`) jedoch abgelaufen war. Dies verhinderte den direkten Zugriff auf SMB-Shares oder die erfolgreiche Durchführung von Kerberoasting. Der dokumentierte Lösungsweg endete an diesem Punkt, da keine weiteren unmittelbaren Angriffsvektoren mit den abgelaufenen Credentials gefunden wurden. Die Flags wurden separat erlangt und werden hier der Vollständigkeit halber aufgeführt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `enum4linux`
*   `nikto`
*   `smbclient`
*   `nxc` (NetExec, auch als `crackmapexec` referenziert)
*   `GetUserSPNs.py` (Impacket)
*   `lookupsid.py` (Impacket)
*   Standard Linux-Befehle (`vi`, `ping`, `grep`, `cat`, `awk`, `cut`, `locate`, `tr`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "DC01" gliederte sich in folgende Phasen (bis zum dokumentierten Stillstand):

1.  **Reconnaissance & Enumeration:**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.114`, Hostname `dc01.hmv`). TTL von Ping deutete auf Windows hin.
    *   `enum4linux` identifizierte die Domain `SUPEDECDE` und den Host `DC01` als Domain Controller, Null-Session-Zugriff wurde blockiert.
    *   Umfassender `nmap`-Scan zeigte typische AD-Ports: 53 (DNS), 88 (Kerberos), 135 (RPC), 139 (NetBIOS), 389 (LDAP), 445 (SMB), 464 (Kerberos PW Change), 636 (LDAPS), 3268 (LDAP GC), 5985 (WinRM). Domain wurde als `SUPEDECDE.LOCAL` bestätigt. SMB-Signing war aktiviert.

2.  **Web Enumeration:**
    *   Der HTTP-Dienst auf Port 5985 (WinRM) gab bei direktem Aufruf einen 404-Fehler.
    *   `nikto` fand auf Port 5985 keine relevanten Schwachstellen, nur fehlende Security Header.

3.  **SMB/AD Enumeration:**
    *   `smbclient -L` listete Shares auf, darunter einen `backup`-Share. Dies funktionierte trotz initialer Passwortabfrage (möglicherweise über Gastzugriff).
    *   `nxc smb` (NetExec) bestätigte Systeminformationen.
    *   `nxc smb --shares` mit Null-Session (`-u "" -p ""`) schlug fehl.
    *   `nxc smb --shares` mit Gastkonto (`-u guest -p ""`) war erfolgreich und listete die Shares auf.
    *   `nxc smb --rid-brute` mit Gastkonto enumerierte erfolgreich Benutzer und Gruppen (z.B. Administrator, Guest, krbtgt, Domain Admins, sowie viele Benutzer nach dem Muster `[name][number]`).
    *   Die extrahierten Benutzernamen wurden in `users.txt` gespeichert.

4.  **Exploitation Attempts / Findings (bis zum Stillstand):**
    *   Ein Passwort-Spraying-Angriff (`nxc smb -u users.txt -p users.txt --no-brute`) wurde durchgeführt.
    *   Die gefilterte Ausgabe zeigte, dass der Login für `ybob317` mit dem Passwort `ybob317` den Status `STATUS_PASSWORD_EXPIRED` ergab.
    *   Der Versuch, mit `ybob317:ybob317` via `smbclient` auf den `backup`-Share zuzugreifen, scheiterte aufgrund des abgelaufenen Passworts.
    *   Kerberoasting mit `GetUserSPNs.py` und den Credentials `SUPEDECDE.LCAL/ybob317:ybob317` schlug ebenfalls wegen `invalidCredentials` fehl.
    *   Weitere Versuche mit `lookupsid.py` (anonym) und einem erneuten Passwort-Spray (mit `crackmapexec` und einer unklaren Userlist `u1.txt`) brachten keine neuen, validen Zugangsdaten, außer der erneuten Bestätigung des abgelaufenen `ybob317`-Kontos.
    *   **Schlussfolgerung des Writeups:** Der Fortschritt wurde durch das abgelaufene Passwort blockiert.

## Wichtige Schwachstellen und Konzepte (identifiziert)

*   **Aktives Gastkonto:** Erlaubte die Enumeration von SMB-Shares und die Durchführung von RID-Cycling zur Aufdeckung von Benutzernamen und Gruppen.
*   **Schwaches (aber abgelaufenes) Passwort:** Das Konto `ybob317` hatte den Benutzernamen als Passwort. Obwohl abgelaufen, ist dies eine unsichere Praxis.
*   **Vorhandensein eines `backup`-Shares:** Potenzielles Ziel für Informationsdiebstahl, falls zugänglich (Zugriff wurde hier durch abgelaufenes Passwort verhindert).
*   **Möglichkeit der Benutzerenumeration:** Durch RID-Cycling konnten viele Benutzerkonten identifiziert werden.

## Flags

*Hinweis: Der Zugriff auf das System zur Erlangung dieser Flags wurde im analysierten Writeup nicht erfolgreich dokumentiert. Die Flags werden hier der Vollständigkeit halber aufgeführt.*

*   **User Flag (`C:\Users\*\Desktop\user.txt`):** `6bab1f09a7403980bfeb4c2b412be47b`
*   **Root Flag (`C:\Users\Administrator\Desktop\root.txt`):** `a9564ebc3289b7a14551baf8ad5ec60a`

## Tags

`HackMyVM`, `DC01`, `Easy`, `Vulnhub`, `Active Directory`, `Windows`, `SMB`, `LDAP`, `Kerberos`, `nxc`, `NetExec`, `crackmapexec`, `enum4linux`, `smbclient`, `Passwort Spraying`, `RID Cycling`, `Gastkonto`
