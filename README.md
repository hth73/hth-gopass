# Gopass Passwortmanager für das Team

<img src="https://img.shields.io/badge/Ubuntu-fd712f?logo=ubuntu&logoColor=white&style=flat" /> <img src="https://img.shields.io/badge/Bash-3e484d?logo=gnu-bash&logoColor=white&style=flat" /> <img src="https://img.shields.io/badge/gopass-58d7e3?style=flat&logo=gnuprivacyguard&logoColor=white&style=flat" />

---
## Beschreibung
gopass ist ein git-basierter Passwortmanager, der auf dem Unix-Tool "pass" aufbaut.<br> Secrets werden verschlüsselt (GPG) im Repository gespeichert und können versionsiert, geteilt und automatisiert verarbeitet werden.<br> Typische Anwendungsfälle sind Zugangsdaten für Datenbanken, Cloud-Provider, APIs oder interne Services. Ziel ist es, Secrets nicht im Code, nicht per Mail und nicht manuell zu verteilen, sondern zentral, reproduzierbar und sicher bereitzustellen.

#### Funktionsprinzip
- Verschlüsselung über GPG (Public/Private Key)
- Speicherung als Dateien im Git-Repo
- Zugriff nur für bekannte Schlüssel (Identität = Key)
- Nachvollziehbarkeit über Git-Historie

#### Einsatz im Team
- Gemeinsamer Zugriff auf Secrets ohne Klartext
- Integration in Ansible / Terraform / CI-CD
- Kein Secret Leaks in Repositories oder Logs
- Reproduzierbare Umgebung (Clone --> Zugriff)

#### RBAC (Zugriffskontrolle)
- Zugriff wird über ".gpg-id" Dateien gesteuert
- Verzeichnisse definieren den Scope (z. B. infra/dev, infra/pro/webserver)
.gpg-id enthält erlaubte GPG Identitäten (Name, E-Mail Adressen)
- Vererbung nach unten (hierarchisches Modell)

---

### Gopass installation
```bash
## pre-installation steps
sudo apt update
sudo apt install git gnupg rng-tools-debian -y

## debug switch
# GOPASS_DEBUG=true gopass <command>

## gopass install steps
cd /tmp
wget https://github.com/gopasspw/gopass/releases/download/v1.16.1/gopass_1.16.1_linux_amd64.deb
sudo dpkg -i gopass_1.16.1_linux_amd64.deb

gopass -v
gopass 1.16.1 (b2fb8ba9) go1.25.5 linux amd64
```

### Einen neuen GPG-Schlüssel erstellen
Hier wird gezeigt, wie ein GPG-Key erstellt wird. Dieser wird später benötigt, um Secrets in "gopass" zu verschlüsseln und zu entschlüsseln.<br>
Für den Einsatz im Team sollte initial eine Person die Grundkonfiguration von "gopass" übernehmen (Repository, Struktur, erste Schlüssel). Sobald das Setup steht, können weitere Teammitglieder über ihre GPG-Keys hinzugefügt oder entfernt werden.

```bash
## gpg key erstellen
gpg --expert --full-generate-key

# Please select what kind of key you want:
# (9) ECC (sign and encrypt) *default*
Your selection? 9

# Please select which elliptic curve you want:
# (1) Curve 25519 *default*
Your selection? 1

# Please specify how long the key should be valid.
# ...     
# 0 = key does not expire
Key is valid for? (0) 0

# Key does not expire at all
Is this correct? (y/N) y

Real name: Helmut Thurnhofer
Email address: hth@htdom.local
Comment: gopass gpg key
Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O

# ---

## gpg key ausgeben lassen
gpg --list-public-keys

# /home/${USER}/.gnupg/pubring.kbx
# ----------------------------
# pub   ed25519 2026-04-09 [SC]
#       05F2E468.....1574D76
# uid           [ultimate] Helmut Thurnhofer (gopass gpg key) <hth@htdom.local>
# sub   cv25519 2026-04-09 [E]
```

### Gopass initial einrichten und mit Repo syncen
```bash
## Lokalen Passwort Store anlegen
##
mkdir /home/${USER}/.gopass-store
gopass init --path /home/${USER}/.gopass-store

cd /home/${USER}/.gopass-store
git init
git remote add origin git@github.com:hth73/hth-gopass.git

git add .
git commit -m "initial gopass store"
git push -u origin master #(main)

## gopass config überprüfen
cat /home/${USER}/.config/gopass/config
# [mounts]
#    path = /home/${USER}/.gopass-store
# [recipients]
#    hash = 7546a39fa4...b2418

## gopass init legt hier Defaultmäßig einen Ordner an [wird aber nicht benötigt!]
ls -la /home/${USER}/.local/share/gopass/stores/root
# total 8
# drwx------ 2 ${USER} ${USER} 4096 Apr  9 12:29 .
# drwx------ 3 ${USER} ${USER} 4096 Apr  9 12:29 ..
```

### Gopass Struktur anlegen
Die Struktur der Verzeichnisse in "gopass" bestimmt direkt das RBAC-Modell.<br>
Berechtigungen werden über ".gpg-id" Dateien pro Verzeichnis definiert und gelten rekursiv für alle darunterliegenden Secrets.
```bash
## Meine Beispiel Struktur
gopass insert -m infra/pro/web/web01.htdom.local/cred
gopass insert -m infra/pro/database/db01.htdom.local/db_pro/cred

gopass insert -m infra/dev/web/web01.dev.htdom.local/cred
gopass insert -m infra/dev/database/db01.dev.htdom.local/db_dev/cred

gopass insert -m infra/share/fileshare/filesrv01.htdom.local/cred

gopass sync
# 🚥 Syncing with all remotes ...
# [<root>] 
#    gitfs pull and push
# OK (no changes)
#    done
# ✅ All done
```

### Team-Member berechtigen
Um Teammitglieder zur "gopass"-Struktur hinzuzufügen, wird der öffentliche GPG-Key jedes Benutzers benötigt. Dieser wird vom jeweiligen Nutzer exportiert und dem Administrator bereitgestellt.<br>
Der Administrator importiert die Public Keys, ergänzt sie in der entsprechenden ".gpg-id" und verschlüsselt die betroffenen Secrets neu, sodass die neuen Mitglieder Zugriff erhalten.
```bash
## Aufgabe vom jeweiligen Nutzer
gpg --armor --export clara.korn@htdom.local ~/gopass_public-key.asc
cat ~/gopass_public-key.asc

# -----BEGIN PGP PUBLIC KEY BLOCK-----
# mDMEadfCcxYJKwYBBAHaRw8BAQdAef0HPxwucioH0ciS2bcmvCgwApFhpOZELJS5
# ...
# LAIvAQ==
# -----END PGP PUBLIC KEY BLOCK-----

# ---

## Aufgabe vom Administrator
gpg --import ~/clara-korn_public-key.asc

## Der Public Key des jeweiligen Nutzers muss vertraut werden [siehe unknown]
##
gpg --list-public-keys

# ...
# 05F2E46818F29FDD88C78301CB7796F9F1574D76
# uid [unknown] Clara Korn (gopass gpg key) <clara.korn@htdom.local>

gpg --edit-key 05F2E46818F29FDD88C78301CB7796F9F1574D76
gpg> lsign

# Are you sure that you want to sign this key with your
# key "Helmut Thurnhofer <hth@hthom.local>" (DFB127D314BDA507)
Really sign? (y/N) y

gpg> trust
# Please decide how far you trust this user to correctly verify other users' keys
# (by looking at passports, checking fingerprints from different sources, etc.)
# ...
5 = I trust ultimately

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

gpg> save

gpg --list-public-keys

# ...
# 05F2E46818F29FDD88C78301CB7796F9F1574D76
# uid [ultimate] Clara Korn (gopass gpg key) <clara.korn@htdom.local>

## Public Key des jeweiligen Nutzers gopass hinzufügen
gopass recipients add clara.korn@htdom.local
# Do you want to add "0xCB7796F9F1574D76 - Clara Korn (gopass gpg key) <clara.korn@htdom.local>" (key "clara.korn@htdom.local") as a recipient to the store ""? [y/N/q]: y
# Reencrypting existing secrets. This may take some time ...
# Starting reencrypt
] 5 / 5 [Goooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooopass] 100.00% 
# Added 1 recipients
# You need to run 'gopass sync' to push these changes

gopass sync

# ---

## Die Public Keys der jeweiligen Benutzer werden dann in folgende Struktur abgelegt. 
## Die Datei ".gpg-id" ist für spätere RBAC konfigurationen wichtig!
cat /home/${USER}/.gopass-store/.gpg-id 
# 0xDFB297D217BDA107
# clara.korn@htdom.local

ls -la .public-keys/
# -rw------- 1 hth hth  656 Apr  9 17:38 0xDFB297D217BDA107
# -rw------- 1 hth hth  673 Apr  9 21:27 clara.korn@htdom.local
```


