# NFSv4 mit Kerberos unter Debian 12 – Administratorhandbuch

**Dokumenttyp:** Technische Betriebs- und Übergabedokumentation  
**Stand:** 2026-11-17  
**Betriebssystem:** Debian GNU/Linux 12 (Bookworm)  
**NFS-Server:** `server2.srv.lcd` (`192.168.1.210`)  
**NFS-Client:** `client2.srv.lcd` (`192.168.1.209`)  
**NFS-Version:** 4.2  
**Sicherheitsmodus:** `sec=krb5p` (Authentifizierung, Integrität, Verschlüsselung)

> **Geltungsbereich:** Diese Datei ist eine Referenzkonfiguration und wurde nicht auf den Zielsystemen ausgeführt. Vor Produktivbetrieb sind insbesondere vorhandene Identitätsdienste, DNS, Benutzer-IDs, Firewall, Backups und Mounts zu prüfen. Der hier angenommene MIT-Kerberos-KDC läuft auf `server2.srv.lcd`. Falls bereits AD, FreeIPA oder ein anderer KDC vorhanden ist, die KDC-Abschnitte nicht ausführen, sondern die vorhandene Infrastruktur verwenden.

## Inhaltsverzeichnis

1. [Systemübersicht](#1-systemübersicht)
2. [Voraussetzungen und Netzwerk](#2-voraussetzungen-und-netzwerk)
3. [Pakete installieren](#3-pakete-installieren)
4. [Kerberos einrichten](#4-kerberos-einrichten)
5. [NFS-Server einrichten](#5-nfs-server-einrichten)
6. [NFS-Client einrichten](#6-nfs-client-einrichten)
7. [Benutzer und Rechte](#7-benutzer-und-rechte)
8. [Firewall](#8-firewall)
9. [Abnahme und Funktionstests](#9-abnahme-und-funktionstests)
10. [Betrieb und Wartung](#10-betrieb-und-wartung)
11. [Fehlerdiagnose](#11-fehlerdiagnose)
12. [Backup und Wiederherstellung](#12-backup-und-wiederherstellung)
13. [Sicherheit und Änderungsmanagement](#13-sicherheit-und-änderungsmanagement)
14. [Dateiübersicht](#14-dateiübersicht)
15. [Quellen](#15-quellen)

## 1. Systemübersicht

| Rolle | Hostname | IPv4 | Funktion |
|---|---|---|---|
| NFS-Server / Referenz-KDC | `server2.srv.lcd` | `192.168.1.210` | NFSv4.2-Export, Kerberos-KDC |
| NFS-Client | `client2.srv.lcd` | `192.168.1.209` | Kerberos-authentifizierter NFS-Mount |

- **Kerberos-Realm (angenommen):** `SRV.LCD`
- **NFSv4-ID-Mapping-Domain:** `srv.lcd`
- **NFS-Pseudowurzel:** `/srv/nfs`
- **Exportverzeichnis:** `/srv/nfs/daten`
- **Client-Mountpunkt:** `/mnt/nfs/daten`
- **Vom Client sichtbarer Export:** `server2.srv.lcd:/daten`
- **Sicherheitsmodus:** `krb5p` – keine Freigabe mit `sec=sys`

### 1.1 Sicherheitsmodi

| Modus | Kerberos-Authentifizierung | Integritätsschutz | Verschlüsselung |
|---|---|---|---|
| `sec=sys` | Nein | Nein | Nein |
| `sec=krb5` | Ja | Kein zusätzlicher RPC-Integritätsschutz | Nein |
| `sec=krb5i` | Ja | Ja | Nein |
| `sec=krb5p` | Ja | Ja | Ja |

### 1.2 Netzwerkports

| Port | Protokoll | Zweck |
|---|---|---|
| 2049 | TCP | NFSv4 |
| 88 | TCP/UDP | Kerberos-KDC |
| 749 | TCP | Kerberos-Administration, nur für autorisierte Admins |
| 464 | TCP/UDP | Kerberos-Passwortänderung |
| 53 | TCP/UDP | DNS, sofern verwendet |
| 123 | UDP | NTP-Zeitabgleich |

NFSv4-only benötigt keine NFSv3-Mountd-Freigabe. Dienste wie `rpcbind` nicht blind maskieren, sondern Abhängigkeiten und den tatsächlichen Dienstzustand prüfen.

## 2. Voraussetzungen und Netzwerk

Alle folgenden administrativen Befehle werden, sofern nicht anders angegeben, als `root` ausgeführt:

```bash
sudo -i
```

### 2.1 Hostnamen

**Server:**

```bash
hostnamectl set-hostname server2.srv.lcd
hostname -f
```

**Client:**

```bash
hostnamectl set-hostname client2.srv.lcd
hostname -f
```

### 2.2 `/etc/hosts` (beide Systeme)

Nur verwenden bzw. ergänzen, wenn die lokale DNS-Auflösung nicht zuverlässig ist. Widersprüchliche bestehende Einträge (insbesondere `127.0.1.1` mit dem eigenen FQDN) vorher bereinigen.

```text
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
192.168.1.210   server2.srv.lcd server2
192.168.1.209   client2.srv.lcd client2
```

Prüfen:

```bash
getent hosts server2.srv.lcd
getent hosts client2.srv.lcd
ping -c 3 server2.srv.lcd
ping -c 3 client2.srv.lcd
```

### 2.3 Zeitabgleich (beide Systeme)

Kerberos setzt weitgehend synchronisierte Uhren voraus.

```bash
apt update
apt install -y chrony
systemctl enable --now chrony
chronyc tracking
chronyc sources -v
timedatectl status
```

In isolierten Netzen eine interne, zuverlässige Zeitquelle konfigurieren.

## 3. Pakete installieren

### 3.1 Server (`server2.srv.lcd`)

```bash
apt update
apt install -y nfs-kernel-server nfs-common krb5-user krb5-kdc krb5-admin-server keyutils chrony acl
```

### 3.2 Client (`client2.srv.lcd`)

```bash
apt update
apt install -y nfs-common krb5-user keyutils chrony
```

Bei der Paketinstallation ggf. den Standard-Realm `SRV.LCD` und den KDC `server2.srv.lcd` angeben.

## 4. Kerberos einrichten

> **Achtung:** Die folgenden KDC-Befehle gelten ausschließlich für einen **neuen** MIT-Kerberos-Realm auf `server2.srv.lcd`. Einen bestehenden Realm niemals neu initialisieren. Keytabs und KDC-Dumps enthalten geheimes Schlüsselmaterial und dürfen nicht ins Git-Repository.

### 4.1 `/etc/krb5.conf` (Server und Client)

```ini
[libdefaults]
    default_realm = SRV.LCD
    dns_lookup_realm = false
    dns_lookup_kdc = false
    rdns = false
    dns_canonicalize_hostname = false
    ticket_lifetime = 10h
    renew_lifetime = 7d
    forwardable = true
    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
    SRV.LCD = {
        kdc = server2.srv.lcd
        admin_server = server2.srv.lcd
        default_domain = srv.lcd
    }

[domain_realm]
    .srv.lcd = SRV.LCD
    srv.lcd = SRV.LCD
```

Der Kernel-Keyring-Credential-Cache benötigt passende Kernel-/PAM-Unterstützung. Bei Problemen den Credential-Cache und `rpc-gssd` gezielt prüfen. Logs werden hier über systemd-Journal ausgewertet.

### 4.2 `/etc/krb5kdc/kdc.conf` (nur Server)

```ini
[kdcdefaults]
    kdc_ports = 88
    kdc_tcp_ports = 88

[realms]
    SRV.LCD = {
        database_name = /var/lib/krb5kdc/principal
        admin_keytab = FILE:/etc/krb5kdc/kadm5.keytab
        acl_file = /etc/krb5kdc/kadm5.acl
        key_stash_file = /etc/krb5kdc/stash
        max_life = 10h 0m 0s
        max_renewable_life = 7d 0h 0m 0s
        default_principal_flags = +preauth
    }
```

### 4.3 `/etc/krb5kdc/kadm5.acl` (nur Server)

```text
*/admin@SRV.LCD *
```

Diese ACL ist für die Beispielinstallation breit gefasst. Produktiv besser nur konkret benannte Admin-Principals zulassen.

### 4.4 KDC-Datenbank initialisieren (nur bei Erstinstallation)

```bash
krb5_newrealm
systemctl enable --now krb5-kdc
systemctl enable --now krb5-admin-server
systemctl status krb5-kdc
systemctl status krb5-admin-server
```

Das abgefragte Master-Passwort sicher und getrennt aufbewahren. Prüfen, ob der erzeugte Stash-Pfad mit `kdc.conf` übereinstimmt.

### 4.5 Administrator und Service-Principals

```bash
kadmin.local
```

Innerhalb von `kadmin.local`:

```text
addprinc admin/admin
addprinc -randkey nfs/server2.srv.lcd
addprinc -randkey host/server2.srv.lcd
addprinc -randkey host/client2.srv.lcd
listprincs
quit
```

Admin-Ticket testen:

```bash
kinit admin/admin@SRV.LCD
klist
```

### 4.6 Server-Keytab erstellen

Auf dem Server:

```bash
kadmin.local
```

```text
ktadd -k /etc/krb5.keytab nfs/server2.srv.lcd
ktadd -k /etc/krb5.keytab host/server2.srv.lcd
quit
```

```bash
chown root:root /etc/krb5.keytab
chmod 600 /etc/krb5.keytab
klist -kte /etc/krb5.keytab
```

**Wichtig:** `ktadd` rotiert standardmäßig Principal-Schlüssel. Nicht erneut auf bereits verteilte Principals anwenden, ohne die Auswirkungen auf vorhandene Keytabs zu planen.

### 4.7 Client-Keytab erstellen und geschützt übertragen

Auf dem KDC:

```bash
kadmin.local
```

```text
ktadd -k /root/client2.keytab host/client2.srv.lcd
quit
```

```bash
chmod 600 /root/client2.keytab
```

Beispiel für eine SSH-Übertragung **nur wenn Root-SSH explizit und sicher autorisiert ist**:

```bash
scp /root/client2.keytab root@client2.srv.lcd:/root/client2.keytab
```

Alternativ einen freigegebenen, abgesicherten Administrationsweg verwenden. Auf dem Client:

```bash
install -o root -g root -m 600 /root/client2.keytab /etc/krb5.keytab
klist -kte /etc/krb5.keytab
rm -f /root/client2.keytab
```

Auf dem KDC nach erfolgreicher Übergabe die temporäre Datei entfernen:

```bash
rm -f /root/client2.keytab
```

Eine normale Dateilöschung garantiert auf SSDs oder Copy-on-Write-Dateisystemen keine physische Vernichtung; temporäre Schlüsseldateien möglichst von Anfang an nur auf geschützten Datenträgern anlegen.

## 5. NFS-Server einrichten

### 5.1 Verzeichnisse und Gruppe

Vor dem Anlegen prüfen, ob GID 2000 bereits belegt ist:

```bash
getent group 2000
getent group nfsdaten
```

Wenn frei:

```bash
mkdir -p /srv/nfs/daten
groupadd -g 2000 nfsdaten
chown root:root /srv/nfs
chmod 755 /srv/nfs
chown root:nfsdaten /srv/nfs/daten
chmod 2770 /srv/nfs/daten
```

### 5.2 `/etc/exports`

```exports
# NFSv4-Pseudowurzel, nur für client2
/srv/nfs 192.168.1.209(ro,fsid=0,sync,no_subtree_check,root_squash,sec=krb5p)

# Datenexport, nur für client2
/srv/nfs/daten 192.168.1.209(rw,sync,no_subtree_check,root_squash,sec=krb5p)
```

| Option | Bedeutung |
|---|---|
| `fsid=0` | NFSv4-Pseudowurzel |
| `ro` / `rw` | Lesen / Lesen und Schreiben |
| `sync` | Synchronisierte Schreiboperationen |
| `no_subtree_check` | Kein Subtree-Checking |
| `root_squash` | Einschränkung von Root-Zugriffen |
| `sec=krb5p` | Kerberos mit Verschlüsselung |

### 5.3 `/etc/nfs.conf` (Server)

Vorhandene Sektionen bearbeiten und keine widersprüchlichen Doppeleinträge erzeugen:

```ini
[general]
pipefs-directory=/run/rpc_pipefs

[nfsd]
vers2=n
vers3=n
vers4=y
vers4.0=y
vers4.1=y
vers4.2=y
tcp=y
udp=n
threads=8
```

### 5.4 `/etc/idmapd.conf` (beide Systeme)

```ini
[General]
Verbosity = 0
Domain = srv.lcd

[Mapping]
Nobody-User = nobody
Nobody-Group = nogroup
```

### 5.5 Server aktivieren und prüfen

```bash
exportfs -rav
systemctl enable --now nfs-kernel-server
systemctl restart nfs-kernel-server
systemctl status nfs-kernel-server
exportfs -v
cat /proc/fs/nfsd/versions
```

Erwartung: NFSv2/v3 deaktiviert, NFSv4 aktiviert. RPCSEC_GSS-Dienste und verfügbare systemd-Units prüfen:

```bash
systemctl list-unit-files | grep -E 'gss|nfs'
systemctl status rpc-gssd.service
journalctl -u nfs-kernel-server -b
```

Nicht voraussetzen, dass `rpc-svcgssd` oder `gssproxy` auf jeder Debian-12-Installation vorhanden sein müssen.

## 6. NFS-Client einrichten

### 6.1 Keytab und GSS prüfen

```bash
klist -kte /etc/krb5.keytab
systemctl enable --now rpc-gssd.service
systemctl status rpc-gssd.service
```

Falls der Dienst nicht läuft:

```bash
systemctl cat rpc-gssd.service
journalctl -u rpc-gssd.service -b
```

### 6.2 Mountpunkt und manueller Mount

```bash
mkdir -p /mnt/nfs/daten
mount -t nfs -o vers=4.2,sec=krb5p,proto=tcp server2.srv.lcd:/daten /mnt/nfs/daten
findmnt /mnt/nfs/daten
nfsstat -m
```

### 6.3 `/etc/fstab` (Client)

```fstab
server2.srv.lcd:/daten /mnt/nfs/daten nfs rw,vers=4.2,sec=krb5p,proto=tcp,hard,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=600 0 0
```

- `hard`: I/O-Anfragen werden bei Serverausfall erneut versucht; Prozesse können dadurch blockieren.
- `_netdev`: Netzwerkdateisystem.
- `nofail`: Kein zwingender Bootfehler bei nicht verfügbarem Mount.
- `x-systemd.automount`: Mount bei Zugriff.
- `x-systemd.idle-timeout=600`: Mögliches Aushängen nach 600 Sekunden Inaktivität.

Aktivieren:

```bash
findmnt --verify
systemctl daemon-reload
systemctl start mnt-nfs-daten.automount
ls -la /mnt/nfs/daten
findmnt /mnt/nfs/daten
nfsstat -m
```

## 7. Benutzer und Rechte

Die Kerberos-Authentifizierung ersetzt **nicht** die Dateisystemberechtigungen und die Linux-Identitätszuordnung. Die UID/GID muss auf Server und Client konsistent sein oder über einen zentralen Identitätsdienst (z. B. SSSD mit LDAP/AD/FreeIPA) bereitgestellt werden.

### 7.1 Beispielidentität

- Benutzer: `nfsuser`
- UID: `2001`
- Gruppe: `nfsdaten`
- GID: `2000`
- Kerberos-Principal: `nfsuser@SRV.LCD`

Auf beiden Systemen vorab prüfen:

```bash
getent passwd 2001
getent group 2000
```

Falls noch nicht vorhanden, auf dem Client und – für konsistente lokale Rechte – auf dem Server:

```bash
groupadd -g 2000 nfsdaten
useradd -m -u 2001 -g nfsdaten -s /bin/bash nfsuser
passwd nfsuser
```

**Hinweis:** Auf dem Server wurde die Gruppe bereits in Abschnitt 5 angelegt. Dort den Befehl `groupadd` **nicht erneut** ausführen. Die obigen Befehle nur für fehlende Identitäten verwenden.

Auf dem KDC:

```bash
kadmin.local
```

```text
addprinc nfsuser
quit
```

Auf dem Client als Benutzer:

```bash
su - nfsuser
kinit nfsuser@SRV.LCD
klist
touch /mnt/nfs/daten/test-nfsuser.txt
echo 'NFSv4 mit Kerberos funktioniert' > /mnt/nfs/daten/test-nfsuser.txt
cat /mnt/nfs/daten/test-nfsuser.txt
```

Auf dem Server kontrollieren:

```bash
ls -ln /srv/nfs/daten/
ls -l /srv/nfs/daten/
```

### 7.2 ACLs (Server)

```bash
apt install -y acl
setfacl -m g:nfsdaten:rwx /srv/nfs/daten
setfacl -d -m g:nfsdaten:rwx /srv/nfs/daten
getfacl /srv/nfs/daten
```

Default-ACLs gelten für neu erstellte Unterobjekte; effektive Berechtigungen hängen zusätzlich von der ACL-Maske ab.

## 8. Firewall

Vor Aktivierung einer Firewall zuerst SSH- und Administrationszugang sicherstellen. Das folgende Beispiel gilt **nur**, wenn UFW tatsächlich als Firewall verwendet wird. Vorhandene nftables-Regeln und vorgeschaltete Firewalls berücksichtigen.

```bash
apt install -y ufw
ufw allow from 192.168.1.209 to 192.168.1.210 port 2049 proto tcp
ufw allow from 192.168.1.209 to 192.168.1.210 port 88 proto tcp
ufw allow from 192.168.1.209 to 192.168.1.210 port 88 proto udp
ufw allow from 192.168.1.209 to 192.168.1.210 port 464 proto tcp
ufw allow from 192.168.1.209 to 192.168.1.210 port 464 proto udp
ufw status verbose
```

Port 749/TCP nur für ausdrücklich autorisierte Administrationssysteme freigeben. Erst nach Sicherung des SSH-Zugangs:

```bash
ufw enable
ufw status numbered
```

## 9. Abnahme und Funktionstests

| Test | Befehl | Erwartung |
|---|---|---|
| Namensauflösung | `getent hosts server2.srv.lcd` | `192.168.1.210` |
| Zeitsynchronisation | `chronyc tracking` | Synchronisiert |
| KDC-Status | `systemctl is-active krb5-kdc` | `active` |
| Benutzer-Ticket | `kinit nfsuser@SRV.LCD && klist` | Gültiges Ticket |
| NFS-Status | `systemctl is-active nfs-kernel-server` | `active` |
| Exporte | `exportfs -v` | Nur Client-IP, `sec=krb5p` |
| Protokoll | `cat /proc/fs/nfsd/versions` | NFSv4 aktiviert |
| Mount | `nfsstat -m` | `vers=4.2`, `sec=krb5p` |
| Schreibzugriff | `touch /mnt/nfs/daten/abnahme.txt` | Erfolgreich als berechtigter Benutzer |
| Rechte | `ls -ln /srv/nfs/daten/abnahme.txt` | Erwartete UID/GID |
| Neustart | Client neu starten und Zugriff testen | Automount funktioniert |

**Wichtig:** `showmount -e` ist für NFSv4-only kein verlässlicher Funktionstest, weil es auf ältere Mount-RPC-Mechanismen zurückgreift.

### 9.1 Negativtest

Als `nfsuser`:

```bash
kdestroy
klist
touch /mnt/nfs/daten/ohne-ticket.txt
```

Ohne gültige Benutzer-Credentials muss ein **neuer** benutzerbezogener Zugriff abgewiesen werden. Bereits zwischengespeicherte GSS-Kontexte können zeitweise weitergelten. Danach:

```bash
kinit nfsuser@SRV.LCD
```

## 10. Betrieb und Wartung

### 10.1 Häufige Befehle

| Aufgabe | Befehl |
|---|---|
| NFS-Status | `systemctl status nfs-kernel-server` |
| NFS-Neustart | `systemctl restart nfs-kernel-server` |
| Exporte neu laden | `exportfs -rav` |
| Exporte anzeigen | `exportfs -v` |
| KDC-Status | `systemctl status krb5-kdc` |
| Kerberos-Adminstatus | `systemctl status krb5-admin-server` |
| GSS-Clientstatus | `systemctl status rpc-gssd` |
| Tickets | `klist` |
| Keytab-Einträge | `klist -kte /etc/krb5.keytab` |
| Mountparameter | `nfsstat -m` |
| NFS-Statistik | `nfsstat -s` |

### 10.2 Logs

Server:

```bash
journalctl -u nfs-kernel-server -b
journalctl -u krb5-kdc -b
journalctl -u krb5-admin-server -b
journalctl -k -b | grep -i nfs
```

Client:

```bash
journalctl -u rpc-gssd -b
journalctl -k -b | grep -Ei 'nfs|rpc|gss'
```

### 10.3 Wartungsintervalle

| Intervall | Aufgabe |
|---|---|
| Täglich | Dienstzustand, Fehlermeldungen und Backupstatus |
| Wöchentlich | Speicherplatz und NFS-Fehlerstatistiken |
| Monatlich | Sicherheitsupdates, Firewall, Berechtigungen |
| Vierteljährlich | Restore-Test und Keytab-/Principal-Inventar |
| Nach Änderungen | Vollständige Abnahmetests |

## 11. Fehlerdiagnose

| Symptom | Häufige Ursache | Prüfung |
|---|---|---|
| `access denied by server` | Export/IP/Sicherheitsmodus | `exportfs -v`, `sec=krb5p` |
| `Permission denied` | Ticket, UID/GID oder ACL | `klist`, `id`, `getfacl` |
| `Clock skew too great` | Zeitabweichung | `chronyc tracking` |
| `Cannot contact any KDC` | DNS, Netzwerk, KDC/Firewall | `getent hosts`, KDC-Status, Port 88 |
| `Server not found in Kerberos database` | Falscher Dienstprincipal/FQDN | `klist -kte`, Hostnamen |
| `Key table entry not found` | Keytab unvollständig | `klist -kte /etc/krb5.keytab` |
| `No credentials cache found` | Kein Ticket | `kinit`, `klist` |
| `Protocol not supported` | NFSv4.2 nicht verfügbar | Kernel, `nfsd`-Versionen |
| `Stale file handle` | Geänderter Export/Dateisystem | Export und Mountzustand |
| Hängender Dateizugriff | Server/Netz nicht erreichbar | Netzwerk und `hard`-Mount |

Client-Diagnose:

```bash
getent hosts server2.srv.lcd
ping -c 3 server2.srv.lcd
klist
systemctl status rpc-gssd
mount -v -t nfs -o vers=4.2,sec=krb5p server2.srv.lcd:/daten /mnt/nfs/daten
```

Server-Diagnose:

```bash
systemctl status nfs-kernel-server
systemctl status krb5-kdc
exportfs -v
ss -lntup | grep -E ':(88|2049|749|464)\b'
journalctl -k -n 100
```

## 12. Backup und Wiederherstellung

### 12.1 Sicherungsumfang

| Daten | Ort | Schutzbedarf |
|---|---|---|
| NFS-Nutzdaten | `/srv/nfs/daten` | Kritisch |
| NFS-Exporte | `/etc/exports` | Hoch |
| NFS-/ID-Mapping-Konfiguration | `/etc/nfs.conf`, `/etc/idmapd.conf` | Hoch |
| Kerberos-Konfiguration | `/etc/krb5.conf`, `/etc/krb5kdc/` | Kritisch |
| Kerberos-Datenbank | Geschützter `kdb5_util dump` | Kritisch |
| Host-Keytabs | `/etc/krb5.keytab` je Host | Kritisch/geheim |
| Client-Mounts | `/etc/fstab` | Hoch |

### 12.2 KDC-Datenbank sichern (Server)

```bash
install -d -m 700 /root/kerberos-backup
kdb5_util dump /root/kerberos-backup/principal.dump
tar -czf /root/kerberos-backup/kdc-config.tar.gz /etc/krb5.conf /etc/krb5kdc
```

Die Sicherung muss verschlüsselt und zugriffsgeschützt aufbewahrt werden. Keytabs und Datenbank müssen bezüglich der Principal-Schlüsselversionen zusammenpassen.

### 12.3 Nutzdaten sichern

```bash
rsync -aHAX --numeric-ids /srv/nfs/daten/ /backup/nfs-daten/
```

`/backup/nfs-daten` muss ein zuvor eingerichtetes Backupziel sein. Eine Sicherung auf derselben physischen Festplatte ist kein ausreichender Ausfallschutz.

### 12.4 Wiederherstellungsreihenfolge

1. Debian und Pakete bereitstellen.
2. Hostnamen, DNS, Netzwerk und Zeit prüfen.
3. KDC-Konfiguration und Kerberos-Datenbank wiederherstellen.
4. Principals und Keytabs abgleichen.
5. NFS-Verzeichnisse, Konfiguration und Exporte wiederherstellen.
6. Nutzdaten mit UID/GID, ACLs und xattrs zurückspielen.
7. Dienste starten und Abnahme durchführen.

Die tatsächlichen Restore-Befehle sind anhand der verwendeten Sicherungssoftware und MIT-Kerberos-Version in einem separaten Wiederherstellungsverfahren zu testen und festzuhalten.

## 13. Sicherheit und Änderungsmanagement

### 13.1 Härtung

- SSH nur für berechtigte Administratoren und Management-Netze.
- Root-SSH-Login nach Möglichkeit deaktivieren.
- Keytabs und Kerberos-Dumps **niemals** in Git einchecken.
- Kein `no_root_squash` ohne ausdrücklich genehmigte Ausnahme.
- Kein unverschlüsselter `sec=sys`-Fallback.
- NFS-Exporte auf konkrete Client-IP-Adressen begrenzen.
- KDC-Administrationsport 749 nicht allgemein öffnen.
- Änderungen an Principals, Keytabs, ACLs und Exports protokollieren.
- Sicherheitsupdates und Restore-Tests planen.

### 13.2 Änderungsprotokoll

| Datum | System | Änderung | Verantwortlich | Ergebnis |
|---|---|---|---|---|
| YYYY-MM-DD | `server2.srv.lcd` | NFS/KDC-Installation | TBD | Offen |
| YYYY-MM-DD | `client2.srv.lcd` | NFS-/Kerberos-Client | TBD | Offen |
| YYYY-MM-DD | Beide | Abnahme | TBD | Offen |

### 13.3 Produktiv-Checkliste

- [ ] Bestehenden Kerberos-/AD-/FreeIPA-Dienst abgeklärt
- [ ] Hostnamen und IP-Adressen bestätigt
- [ ] DNS und Reverse-DNS geprüft
- [ ] NTP synchronisiert
- [ ] UID/GID-Kollisionen ausgeschlossen
- [ ] Principals und Keytabs geprüft
- [ ] Benutzeridentitäten korrekt zugeordnet
- [ ] NFSv4.2 und `sec=krb5p` bestätigt
- [ ] Schreibrechte und ACLs getestet
- [ ] Firewall und SSH-Zugang geprüft
- [ ] Backup und Wiederherstellung getestet
- [ ] Neustart-/Automount-Test bestanden

## 14. Dateiübersicht

| Datei | Server | Client |
|---|---|---|
| `/etc/hostname` | Ja | Ja |
| `/etc/hosts` | Falls erforderlich | Falls erforderlich |
| `/etc/krb5.conf` | Ja | Ja |
| `/etc/krb5.keytab` | Ja, geheim | Ja, geheim |
| `/etc/krb5kdc/kdc.conf` | Ja | Nein |
| `/etc/krb5kdc/kadm5.acl` | Ja | Nein |
| `/etc/exports` | Ja | Nein |
| `/etc/nfs.conf` | Ja | Optional |
| `/etc/idmapd.conf` | Ja | Ja |
| `/etc/fstab` | Keine neue NFS-Zeile | Ja |
| `/etc/chrony/chrony.conf` | Bei Bedarf | Bei Bedarf |
| `/etc/nsswitch.conf` | Identitätsverwaltung prüfen | Identitätsverwaltung prüfen |
| `/etc/sssd/sssd.conf` | Nur bei SSSD | Nur bei SSSD |

### 14.1 Git-Hinweise

Dieses Markdown-Dokument enthält **nur Beispielkonfigurationen und keine echten Geheimnisse**. Keine realen Keytabs, KDC-Dumps, Passwörter, Tickets oder privaten Schlüssel in dasselbe Repository aufnehmen.

Beispiel `.gitignore`:

```gitignore
*.keytab
*.kt
principal.dump
*.ccache
*.secret
*.pem
```

Ein `.gitignore` ersetzt keine Prüfung bereits versionierter Dateien. Vor dem Commit `git status` und `git diff --cached` kontrollieren.

## 15. Quellen

- [Debian Administrator's Handbook – NFS](https://www.debian.org/doc/manuals/debian-handbook/sect.nfs-file-server.de.html)
- [Debian Wiki – NFS/Kerberos](https://wiki.debian.org/NFS/Kerberos)
- [Debian Bookworm – exports(5)](https://manpages.debian.org/bookworm/nfs-kernel-server/exports.5.en.html)
- [Debian Bookworm – nfs(5)](https://manpages.debian.org/bookworm/nfs-common/nfs.5.en.html)
- [MIT Kerberos Documentation](https://web.mit.edu/kerberos/krb5-latest/doc/)

---

**Dokumentstatus:** Referenzdokumentation, technische Abnahme ausstehend.  
**Pflege:** Änderungen an Konfiguration und Infrastruktur müssen in diesem Dokument nachgezogen und im Git-Verlauf nachvollziehbar dokumentiert werden.
