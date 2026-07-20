# Übungsblatt 07, Aufgabe 17(a) und 17(b)

# IPv6-Reversezone und Delegation der Reversezone

## Kontext der Aufgabe

In Aufgabe 17 geht es um Reverse DNS für IPv6.

Die Aufgabenstellung lautet sinngemäß:

```text
17. IPv6 Reversezone

(a) Konfigurieren Sie auf pcsec1 die IPv6 Reversezone ihres Netzes.
    Tragen Sie PTR-Einträge für die drei Hosts Ihres Subnetzes ein.

(b) Konfigurieren Sie auf dns eine Delegation der Reversezone.
```

Die beiden Teilaufgaben gehören zusammen:

- In **17(a)** wird auf `pcsec1` die eigentliche Reversezone angelegt.
- In **17(b)** wird auf `dns` eingetragen, dass Anfragen für diese Reversezone an `pcsec1` weitergeleitet beziehungsweise delegiert werden.

Ohne 17(a) existieren die PTR Records nicht.

Ohne 17(b) kann der übergeordnete DNS-Server die Reversezone nicht finden.

---

# 1. Ausgangszustand aus der vorhandenen Konfiguration

Aus der vorhandenen Konfiguration ergeben sich folgende Werte:

| Wert | Bedeutung |
|---|---|
| Gruppe | `gruppe09` |
| Subzone | `sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de` |
| DNS-Master der Subzone | `pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de` |
| Parent-DNS | `dns.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de` beziehungsweise `10.153.210.251` |
| Forward-Zonendatei auf `pcsec1` | `/etc/bind/db.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de` |
| Reverse-Zonendatei auf `pcsec1` | `/etc/bind/db.2001.4ca0.4001.f21` |

Die Forward-Zone auf `pcsec1` enthält aktuell:

```dns
$TTL 3600
@   IN  SOA pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de. root.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de. (
        2026060804
        3600
        900
        604800
        86400 )

@       IN  NS      pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
@       IN  NS      dns.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.

pcsec1  IN  A       10.153.210.2
pcsec1  IN  AAAA    2001:4ca0:4001:f21::2

pcsec2  IN  A       10.153.210.3
pcsec2  IN  AAAA    2001:4ca0:4001:f21::3

gwsec   IN  A       10.153.210.193
gwsec   IN  AAAA    2001:4ca0:4001:f00::21

dns     IN  A       10.153.210.251
dns     IN  AAAA    2001:4ca0:4001:f00::251
```

Wichtig ist hier:

```text
pcsec1  -> 2001:4ca0:4001:f21::2
pcsec2  -> 2001:4ca0:4001:f21::3
```

In deiner späteren Reverse-Konfiguration wurde außerdem die Reversezone für dieses Netz eingetragen:

```dns
zone "1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa" {
    type master;
    file "/etc/bind/db.2001.4ca0.4001.f21";
};
```

Diese Zone gehört zum IPv6-Präfix:

```text
2001:4ca0:4001:f21::/64
```

---

# 2. Wichtiger Hinweis zu `2001:db6::/64`

Auf den Interfaces von `gwsec`, `pcsec1` und `pcsec2` waren auch diese Adressen sichtbar:

| Host | Interface-Adresse |
|---|---|
| `gwsec` | `2001:db6::2/64` |
| `pcsec1` | `2001:db6::3/64` |
| `pcsec2` | `2001:db6::4/64` |

Dafür hattest du zwischendurch auch diese Reversezone angelegt:

```dns
zone "0.0.0.0.0.0.0.0.6.b.d.0.1.0.0.2.ip6.arpa" {
    type master;
    file "/etc/bind/db.2001.db6";
};
```

Diese Zone gehört zu:

```text
2001:db6::/64
```

Der zuletzt gezeigte Zustand von `named.conf.local` nutzt aber nicht mehr diese `2001:db6`-Zone, sondern die Zone für:

```text
2001:4ca0:4001:f21::/64
```

Deshalb wird im Hauptteil dieser Dokumentation die `f21`-Reversezone beschrieben.

Falls das Testsystem ausdrücklich die Interface-Adressen `2001:db6::2`, `2001:db6::3` und `2001:db6::4` prüft, muss stattdessen die `2001:db6::/64`-Variante verwendet werden. Diese Variante steht am Ende dieses Dokuments als Kontrollhinweis.

---

# 3. Warum Reverse DNS notwendig ist

Normales DNS löst Namen zu IP-Adressen auf.

Beispiel:

```text
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
        -> 2001:4ca0:4001:f21::2
```

Reverse DNS macht das Gegenteil.

Beispiel:

```text
2001:4ca0:4001:f21::2
        -> pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

Dafür werden DNS Records vom Typ `PTR` verwendet.

Ein PTR Record sagt:

```text
Diese IP-Adresse gehört zu diesem Hostnamen.
```

Bei IPv6 liegen Reversezonen unter:

```text
ip6.arpa
```

---

# 4. Wie der Name der IPv6-Reversezone entsteht

Die relevante Zone ist:

```text
2001:4ca0:4001:f21::/64
```

Zuerst wird das Präfix vollständig ausgeschrieben.

```text
2001:4ca0:4001:0f21::/64
```

Für ein `/64`-Präfix werden die ersten 64 Bit verwendet. Das sind die ersten vier Blöcke:

```text
2001:4ca0:4001:0f21
```

Jetzt wird alles in einzelne Hex-Zeichen zerlegt:

```text
2 0 0 1 4 c a 0 4 0 0 1 0 f 2 1
```

Danach werden diese Zeichen einzeln rückwärts geschrieben und mit Punkten getrennt:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2
```

Am Ende kommt `.ip6.arpa` dazu.

Die vollständige Reversezone lautet also:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

Das ist genau der Zonenname, der in deiner aktuellen `named.conf.local` steht.

---

# 5. Aufgabe 17(a): Reversezone auf `pcsec1`

## Ziel von 17(a)

Auf `pcsec1` soll eine IPv6-Reversezone für das eigene Subnetz angelegt werden.

`pcsec1` ist dabei der autoritative Nameserver für diese Reversezone.

Die Zone soll PTR Records für die drei Hosts des Subnetzes enthalten.

Für das Subnetz `2001:4ca0:4001:f21::/64` sind die Hostanteile relevant.

Typische Zuordnung in diesem Subnetz:

| IPv6-Adresse | Reverse-Name innerhalb der Zone | PTR-Ziel |
|---|---|---|
| `2001:4ca0:4001:f21::1` | `1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0` | `gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.` |
| `2001:4ca0:4001:f21::2` | `2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0` | `pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.` |
| `2001:4ca0:4001:f21::3` | `3.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0` | `pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.` |

Achtung:

In deiner alten Reverse-Datei stand teilweise `gwsec1.sub09...`.

In deiner Forward-Zone steht aber `gwsec.sub09...`.

Wenn das Testsystem `gwsec` erwartet, sollte der PTR Record ebenfalls auf `gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.` zeigen.

Wenn im Praktikum explizit `gwsec1` verwendet wurde, dann muss der PTR Record entsprechend auf `gwsec1.sub09...` zeigen. Die Namen müssen zur vorherigen DNS-Konfiguration passen.

---

# 6. Backup der bestehenden BIND-Dateien

Vor Änderungen sollte man die aktuellen Dateien sichern.

Auf `pcsec1`:

```bash
cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak-aufgabe17
cp /etc/bind/db.2001.4ca0.4001.f21 /etc/bind/db.2001.4ca0.4001.f21.bak-aufgabe17 2>/dev/null || true
```

Damit kann man bei Fehlern den vorherigen Zustand wiederherstellen.

---

# 7. `named.conf.local` auf `pcsec1`

Die Datei wird geöffnet mit:

```bash
nano /etc/bind/named.conf.local
```

Der relevante Inhalt sollte so aussehen:

```dns
include "/etc/bind/tsig-sub09.key";

zone "sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de" {
    type master;
    file "/etc/bind/db.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de";
    allow-transfer { key "sub09-transfer."; };
    also-notify { 10.153.210.251 key "sub09-transfer."; };
    notify yes;
};

zone "1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa" {
    type master;
    file "/etc/bind/db.2001.4ca0.4001.f21";
};
```

Erklärung:

```dns
zone "1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa"
```

legt die IPv6-Reversezone für das Netz `2001:4ca0:4001:f21::/64` an.

```dns
type master;
```

bedeutet, dass `pcsec1` die primäre Quelle dieser Zone ist.

```dns
file "/etc/bind/db.2001.4ca0.4001.f21";
```

legt fest, aus welcher Datei BIND die Zonendaten liest.

---

# 8. Zonendatei für die Reversezone erstellen

Die Datei wird auf `pcsec1` erstellt oder überschrieben:

```bash
cat > /etc/bind/db.2001.4ca0.4001.f21 <<'ZONE'
$TTL 3600
@   IN  SOA pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de. root.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de. (
        2026062301 ; Serial
        3600       ; Refresh
        900        ; Retry
        604800     ; Expire
        86400 )    ; Negative Cache TTL

@       IN  NS      pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.

; ------------------------------------------------------------------
; PTR Records für IPv6-Reverseauflösung
; Zone: 2001:4ca0:4001:f21::/64
; ------------------------------------------------------------------

1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
3.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
ZONE
```

Damit existieren drei PTR Records.

---

# 9. Erklärung der PTR Records

## PTR für `gwsec`

```dns
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Dieser Eintrag bedeutet:

```text
2001:4ca0:4001:f21::1
        -> gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

Der Hostanteil `::1` wird bei IPv6-Reverse DNS als Nibble-Reihenfolge geschrieben.

`::1` wird innerhalb der `/64`-Zone zu:

```text
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0
```

## PTR für `pcsec1`

```dns
2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Dieser Eintrag bedeutet:

```text
2001:4ca0:4001:f21::2
        -> pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

## PTR für `pcsec2`

```dns
3.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Dieser Eintrag bedeutet:

```text
2001:4ca0:4001:f21::3
        -> pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

---

# 10. Zone auf `pcsec1` prüfen

Nach dem Erstellen der Dateien wird zuerst die komplette BIND-Konfiguration geprüft:

```bash
named-checkconf
```

Wenn keine Ausgabe kommt, ist die Syntax grundsätzlich korrekt.

Danach wird die Reversezone geprüft:

```bash
named-checkzone \
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa \
/etc/bind/db.2001.4ca0.4001.f21
```

Erwartete Ausgabe:

```text
zone 1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa/IN: loaded serial 2026062301
OK
```

Wenn `OK` angezeigt wird, ist die Zonendatei syntaktisch korrekt.

---

# 11. BIND auf `pcsec1` neu laden

Nach erfolgreicher Prüfung wird BIND neu geladen:

```bash
systemctl reload bind9
```

Alternativ:

```bash
systemctl restart bind9
```

Danach wird geprüft, ob BIND läuft:

```bash
systemctl status bind9 --no-pager
```

Wichtig ist, dass dort kein Fehler zur neuen Reversezone angezeigt wird.

---

# 12. Lokaler Test der Reversezone auf `pcsec1`

Direkt gegen `pcsec1` kann die Reverseauflösung getestet werden.

```bash
dig @127.0.0.1 -x 2001:4ca0:4001:f21::1 +short
dig @127.0.0.1 -x 2001:4ca0:4001:f21::2 +short
dig @127.0.0.1 -x 2001:4ca0:4001:f21::3 +short
```

Erwartete Ausgabe:

```text
gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Falls beim ersten Eintrag im Praktikum der Name `gwsec1` erwartet wird, wäre die erwartete Ausgabe entsprechend:

```text
gwsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

---

# 13. Was 17(a) damit erfüllt

Mit den bisherigen Schritten ist Aufgabe 17(a) erfüllt:

1. Auf `pcsec1` existiert eine IPv6-Reversezone.
2. Die Zone ist als `master` in BIND eingetragen.
3. Die Zone enthält PTR Records für die drei Hosts des Subnetzes.
4. `pcsec1` kann die Reverse-DNS-Anfragen lokal beantworten.

Beispiel:

```text
2001:4ca0:4001:f21::2
        -> pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

---

# 14. Aufgabe 17(b): Delegation der Reversezone auf `dns`

## Ziel von 17(b)

In 17(a) kennt nur `pcsec1` die Reversezone.

Damit andere Resolver die Zone finden können, muss der übergeordnete DNS-Server `dns` wissen:

```text
Für die Reversezone 1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa ist pcsec1 zuständig.
```

Das wird durch einen NS Record in der Parent-Reversezone gemacht.

Das Prinzip ist:

```text
Parent-Reversezone auf dns
        |
        | delegiert
        v
Reversezone auf pcsec1
```

---

# 15. Auf `dns` einloggen

Login auf `dns`:

```bash
ssh root@10.153.210.251
```

Falls der Hostname funktioniert:

```bash
ssh root@dns.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

---

# 16. Parent-Reversezone auf `dns` finden

Auf `dns` muss zuerst geprüft werden, welche Reversezonen dort bereits konfiguriert sind.

```bash
grep -R "ip6.arpa" /etc/bind/named.conf.local /etc/bind/named.conf* 2>/dev/null
```

Man sucht nach einer Zone, die ein Parent der folgenden Zone ist:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

Mögliche Parent-Zone könnte zum Beispiel sein:

```text
1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

Das wäre die Reversezone für ein größeres Netz wie:

```text
2001:4ca0:4001::/48
```

In diesem Fall wäre der Delegationsname innerhalb der Parent-Zone:

```text
1.2.f.0
```

Denn:

```text
1.2.f.0 + 1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
=
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

---

# 17. Parent-Zonendatei auf `dns` sichern

Sobald die Parent-Zone gefunden wurde, die zugehörige Zonendatei sichern.

Beispiel:

```bash
cp /etc/bind/db.2001.4ca0.4001 /etc/bind/db.2001.4ca0.4001.bak-aufgabe17
```

Der Dateiname kann bei euch anders sein.

Deshalb vorher immer mit `grep` und `cat /etc/bind/named.conf.local` prüfen, welche Datei wirklich verwendet wird.

---

# 18. Delegation in der Parent-Reversezone eintragen

In der Parent-Zone auf `dns` wird ein NS Record für die Reversezone eingetragen.

Wenn die Parent-Zone zum Beispiel lautet:

```text
1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

und die Child-Zone lautet:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

dann lautet der Eintrag in der Parent-Zonendatei:

```dns
1.2.f.0     IN  NS  pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Vollständig bedeutet das:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
        NS pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

Damit sagt `dns`:

```text
Für diese Reversezone ist pcsec1 zuständig.
```

---

# 19. Beispiel für eine Parent-Zone auf `dns`

Die konkrete Datei kann je nach vorhandener Konfiguration anders heißen.

Das Prinzip sieht so aus:

```dns
$TTL 3600
@   IN  SOA dns.gruppe09.secp-int.lab.nm.ifi.lmu.de. root.gruppe09.secp-int.lab.nm.ifi.lmu.de. (
        2026062302 ; Serial erhöhen
        3600
        900
        604800
        86400 )

@       IN  NS      dns.gruppe09.secp-int.lab.nm.ifi.lmu.de.

; Delegation der IPv6-Reversezone von sub09 an pcsec1
1.2.f.0 IN  NS      pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Wichtig:

- Die Seriennummer im SOA Record muss erhöht werden.
- Der NS Record muss in der richtigen Parent-Zone stehen.
- Am Ende des Nameservernamens muss ein Punkt stehen.

Der Punkt am Ende ist wichtig:

```dns
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Ohne Punkt würde BIND den Namen relativ zur aktuellen Zone interpretieren und daraus einen falschen Namen machen.

---

# 20. Sind Glue Records für 17(b) nötig?

Für diese Reverse-Delegation sind normalerweise keine Glue Records in der Reversezone nötig.

Grund:

Der delegierte Nameserver heißt:

```text
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

Dieser Name liegt nicht unter der delegierten Reversezone:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

Glue Records braucht man vor allem dann, wenn der Nameservername selbst innerhalb der Zone liegt, die gerade delegiert wird.

Beispiel für Glue-Notwendigkeit:

```text
Zone: sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
Nameserver: pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

Hier liegt der Nameserver innerhalb der delegierten Forward-Zone.

Bei der Reversezone ist das anders, weil der Nameservername in der Forward-DNS-Hierarchie liegt.

---

# 21. Parent-Zone auf `dns` prüfen

Nachdem die Delegation eingetragen wurde, wird die Konfiguration auf `dns` geprüft.

Zuerst:

```bash
named-checkconf
```

Dann die Parent-Zone prüfen.

Beispiel:

```bash
named-checkzone \
1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa \
/etc/bind/db.2001.4ca0.4001
```

Der genaue Zonenname und Dateiname müssen zur echten Konfiguration auf `dns` passen.

Erwartete Ausgabe:

```text
zone <parent-zone>/IN: loaded serial 2026062302
OK
```

---

# 22. BIND auf `dns` neu laden

Nach erfolgreicher Prüfung:

```bash
systemctl reload bind9
```

Oder:

```bash
systemctl restart bind9
```

Status prüfen:

```bash
systemctl status bind9 --no-pager
```

---

# 23. Delegation auf `dns` testen

Zuerst wird getestet, ob `dns` die Delegation kennt.

```bash
dig @127.0.0.1 1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa NS +short
```

Erwartete Ausgabe:

```text
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Danach kann die tatsächliche Reverseauflösung über `dns` getestet werden.

```bash
dig @10.153.210.251 -x 2001:4ca0:4001:f21::2 +short
```

Erwartete Ausgabe:

```text
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Für `pcsec2`:

```bash
dig @10.153.210.251 -x 2001:4ca0:4001:f21::3 +short
```

Erwartete Ausgabe:

```text
pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Für den Gateway-Host im Subnetz:

```bash
dig @10.153.210.251 -x 2001:4ca0:4001:f21::1 +short
```

Erwartete Ausgabe:

```text
gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Falls `dns` keine rekursive Auflösung macht, kann die Antwort leer sein, aber in der Authority Section sollte die Delegation sichtbar sein.

Dann prüft man mit:

```bash
dig @10.153.210.251 -x 2001:4ca0:4001:f21::2 +norecurse
```

Dort sollte im Authority-Bereich der NS Record auf `pcsec1` stehen.

---

# 24. Direkter Vergleich: Test gegen `pcsec1` und gegen `dns`

## Test direkt gegen `pcsec1`

```bash
dig @10.153.210.2 -x 2001:4ca0:4001:f21::2 +short
```

Wenn das funktioniert, ist 17(a) korrekt.

## Test gegen `dns`

```bash
dig @10.153.210.251 -x 2001:4ca0:4001:f21::2 +short
```

Wenn das funktioniert, ist 17(b) wahrscheinlich auch korrekt.

Merksatz:

```text
Direkt gegen pcsec1 erfolgreich  -> Reversezone selbst funktioniert.
Gegen dns erfolgreich            -> Delegation funktioniert zusätzlich.
```

---

# 25. Typische Fehlerquellen

## Fehler 1: Falscher Reversezonenname

Für `2001:4ca0:4001:f21::/64` ist richtig:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

Falsch wäre zum Beispiel eine komplette 128-Bit-Adresse als Zone zu verwenden.

Die komplette 128-Bit-Reverseform gehört zu einem einzelnen PTR-Namen, nicht zur `/64`-Zone.

## Fehler 2: `f21` nicht als `0f21` behandelt

Der Block `f21` muss für IPv6 intern als `0f21` verstanden werden.

Deshalb taucht in der Reversezone auch das zusätzliche `0` auf:

```text
1.2.f.0
```

nicht nur:

```text
1.2.f
```

## Fehler 3: Punkt am Ende vergessen

Richtig:

```dns
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Falsch:

```dns
pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

Ohne Punkt ergänzt BIND automatisch die aktuelle Zone.

Dadurch entstehen falsche Namen.

## Fehler 4: Seriennummer nicht erhöht

Wenn eine Zonendatei geändert wird, sollte die Seriennummer im SOA Record erhöht werden.

Beispiel:

```dns
2026062301
```

Bei der nächsten Änderung:

```dns
2026062302
```

## Fehler 5: `gwsec` und `gwsec1` verwechselt

In deiner Konfiguration tauchte teilweise dieser PTR Record auf:

```dns
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR gwsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Die Forward-Zone enthält aber:

```dns
gwsec IN A    10.153.210.193
gwsec IN AAAA 2001:4ca0:4001:f00::21
```

Wenn `gwsec1` nirgends als Forward Record existiert, ist `gwsec1` wahrscheinlich falsch oder zumindest inkonsistent.

Für sauberes DNS sollten Forward- und Reverse-Namen zusammenpassen.

## Fehler 6: Falsche Zone auf `dns` bearbeitet

Bei 17(b) darf die Delegation nicht irgendwo eingetragen werden.

Sie muss in die passende Parent-Reversezone.

Deshalb zuerst auf `dns` suchen:

```bash
grep -R "ip6.arpa" /etc/bind/named.conf.local /etc/bind/named.conf* 2>/dev/null
```

Dann die dort eingetragene Zonendatei bearbeiten.

---

# 26. Kontrollbefehle für die Abgabe

Auf `pcsec1`:

```bash
named-checkconf
named-checkzone 1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa /etc/bind/db.2001.4ca0.4001.f21
systemctl reload bind9

dig @127.0.0.1 -x 2001:4ca0:4001:f21::1 +short
dig @127.0.0.1 -x 2001:4ca0:4001:f21::2 +short
dig @127.0.0.1 -x 2001:4ca0:4001:f21::3 +short
```

Auf `dns`:

```bash
grep -R "ip6.arpa" /etc/bind/named.conf.local /etc/bind/named.conf* 2>/dev/null
named-checkconf
named-checkzone <parent-reverse-zone> <parent-zone-file>
systemctl reload bind9

dig @127.0.0.1 1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa NS +short
```

Von einem anderen Host, zum Beispiel `gwsec`:

```bash
dig @10.153.210.251 -x 2001:4ca0:4001:f21::2 +short
dig @10.153.210.251 -x 2001:4ca0:4001:f21::3 +short
```

---

# 27. Zusammenfassung von Aufgabe 17(a)

Für 17(a) wurde auf `pcsec1` die IPv6-Reversezone eingerichtet.

Die aktive Reversezone lautet:

```text
1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
```

Diese Zone gehört zu:

```text
2001:4ca0:4001:f21::/64
```

Die Zonendatei ist:

```text
/etc/bind/db.2001.4ca0.4001.f21
```

Sie enthält PTR Records für die Hosts des Subnetzes:

```text
2001:4ca0:4001:f21::1 -> gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
2001:4ca0:4001:f21::2 -> pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
2001:4ca0:4001:f21::3 -> pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de
```

Damit kann `pcsec1` Reverse-DNS-Anfragen für dieses Subnetz beantworten.

---

# 28. Zusammenfassung von Aufgabe 17(b)

Für 17(b) wird auf `dns` eine Delegation der Reversezone eingetragen.

Die Parent-Reversezone verweist mit einem NS Record auf `pcsec1`.

Der Delegationseintrag sieht grundsätzlich so aus:

```dns
1.2.f.0 IN NS pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Dieser Eintrag bedeutet:

```text
Die Reversezone 1.2.f.0.1.0.0.4.0.a.c.4.1.0.0.2.ip6.arpa
wird von pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de verwaltet.
```

Damit können auch andere Resolver über `dns` herausfinden, dass sie für PTR-Anfragen dieses IPv6-Subnetzes `pcsec1` fragen müssen.

---

# 29. Was danach kommt

Nach 17(a) und 17(b) sollte man in dieser Reihenfolge weitergehen:

1. Auf `pcsec1` prüfen, ob die Reversezone lokal funktioniert.
2. Auf `dns` prüfen, ob die Delegation existiert.
3. Von `gwsec` oder einem anderen Host testen, ob Reverseauflösung über `dns` funktioniert.
4. Im Praktikums-Testsystem Aufgabe 17 prüfen lassen.
5. Wenn Aufgabe 17 erfolgreich ist, mit Aufgabe 18 DNSSEC weitermachen.

Für Aufgabe 18 ist wichtig:

Jede spätere Änderung an einer DNS-Zone erfordert nach dem Signieren ein neues Signieren der Zone.

Deshalb sollte Aufgabe 17 vorher sauber funktionieren, bevor DNSSEC eingerichtet wird.

---

# 30. Kontrollhinweis: Variante für `2001:db6::/64`

Falls das Testsystem nicht die `f21`-Adressen prüft, sondern die Interface-Adressen aus `ip -6 addr`, dann ist stattdessen diese Zone relevant:

```text
2001:db6::/64
```

Die Reversezone dazu lautet:

```text
0.0.0.0.0.0.0.0.6.b.d.0.1.0.0.2.ip6.arpa
```

Dann wäre der Zone-Block auf `pcsec1`:

```dns
zone "0.0.0.0.0.0.0.0.6.b.d.0.1.0.0.2.ip6.arpa" {
    type master;
    file "/etc/bind/db.2001.db6";
};
```

Die Zonendatei wäre:

```dns
$TTL 3600
@   IN  SOA pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de. root.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de. (
        2026062301
        3600
        900
        604800
        86400 )

@       IN  NS      pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.

2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  gwsec.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
3.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  pcsec1.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
4.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0    IN  PTR  pcsec2.sub09.gruppe09.secp-int.lab.nm.ifi.lmu.de.
```

Tests dazu:

```bash
dig @127.0.0.1 -x 2001:db6::2 +short
dig @127.0.0.1 -x 2001:db6::3 +short
dig @127.0.0.1 -x 2001:db6::4 +short
```

Diese Variante sollte nur verwendet werden, wenn das Praktikum beziehungsweise das Testsystem ausdrücklich das `2001:db6::/64`-Netz meint.

