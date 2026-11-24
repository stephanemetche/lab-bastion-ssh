# Procédure personnelle — Bastion SSH (Debian 13 / VMware Workstation / Windows)

> Ma procédure pour **refaire ce lab de zéro**, avec mes vraies valeurs et les pièges que j'ai rencontrés.
> À garder dans ma base de connaissances (Trilium) et à adapter pour d'autres projets.

## 0. Mes valeurs

| Élément | Valeur |
|---|---|
| Hôte | PC Windows, VMware Workstation Pro, PowerShell 5.1 |
| Dossier des VM | `D:\VMs\bastion-lab\` |
| ISO | `D:\ISO\debian-13.x-amd64-netinst.iso` |
| Utilisateur Debian | `stephane` (+ mot de passe root séparé) |
| Réseau NAT (VMnet8) | `192.168.60.0/24`, passerelle **`.2`**, DHCP `.128–.254` |
| Réseau interne (VMnet2) | `10.10.10.0/24`, hôte uniquement, **sans** adaptateur hôte ni DHCP |
| Cartes Debian | `ens32` (1re carte), `ens34` (2e carte du bastion) |
| Bastion | `192.168.60.10` + `10.10.10.1` |
| Web / DB | `10.10.10.10` / `10.10.10.20`, passerelle `10.10.10.1` |
| Clé SSH | `%USERPROFILE%\.ssh\bastion_key` (ed25519) |

## Les 5 réflexes (tous mes blocages venaient d'ici)

1. **Regarder l'invite avant de taper** : `PS C:\>` = PC Windows · `stephane@…$` = VM, utilisateur · `root@…#` = VM, root.
2. **`su -` seul sur sa ligne**, jamais dans un bloc collé (la ligne suivante serait prise pour le mot de passe).
3. **Copier-coller les fichiers de config** avec `cat > fichier << 'EOF' … EOF` plutôt que les retaper.
4. **Pas de message = succès** sous Linux. Une ligne rouge = je m'arrête et je lis.
5. **Console VMware ouverte** avant de toucher au réseau, à SSH ou au pare-feu ; **tester dans une 2e fenêtre** avant de fermer la 1re.

---

## Phase 1 — Réseaux VMware

1. **Modifier → Éditeur de réseau virtuel → Modifier les paramètres** (bouclier admin).
2. **VMnet8 (NAT)** : vérifier **DHCP activé** et **Paramètres du NAT → Passerelle = `192.168.60.2`**.
   > ⚠️ Piège rencontré : un ancien TP avait mis la passerelle en `.254` (= adresse du DHCP) → plus d'internet.
3. **Ajouter un réseau → VMnet2** → Hôte uniquement → **décocher** « Connecter un hôte… » et « Utiliser le service DHCP… » → sous-réseau `10.10.10.0` / `255.255.255.0` → Appliquer.
4. Vérifier : `ipconfig | Select-String "VMnet"` → **pas** de VMnet2.

## Phase 2 — VM modèle et clones liés

1. Nouvelle VM « Typique » → ISO → Debian 13.x 64 bits → 8 Go, fractionné → RAM 1 Go, 1 CPU, NAT, **supprimer** carte son / imprimante / USB.
2. Installation **en mode texte** (« Install ») : hostname du modèle, utilisateur `stephane`, partition unique, miroir `deb.debian.org`, logiciels : **serveur SSH + utilitaires usuels uniquement**.
3. Préparer le modèle (root) :
   ```bash
   apt update && apt full-upgrade -y
   apt install -y curl nftables
   apt clean
   poweroff
   ```
4. VM → Paramètres → CD/DVD : décocher « connecter à la mise sous tension ».
5. **Instantané** `base-propre`, puis **VM → Gérer → Cloner → instantané existant → clone lié** ×3 : `lab-bastion01`, `lab-web01`, `lab-db01`.
   > Ne jamais supprimer le modèle ni son instantané.
6. Matériel : 1 Go / 1 CPU / CD décoché. **Bastion : Ajouter → Carte réseau 2 → Personnalisé VMnet2** (et garder la carte 1 en NAT !).
7. Identité unique de **chaque** clone (root, adapter le nom) :
   ```bash
   hostnamectl set-hostname lab-bastion01
   sed -i 's/ANCIEN_NOM/lab-bastion01/g' /etc/hosts
   rm -f /etc/machine-id /var/lib/dbus/machine-id
   systemd-machine-id-setup
   rm -f /etc/ssh/ssh_host_*
   ssh-keygen -A
   reboot
   ```
   Côté PC, oublier l'ancienne empreinte : `ssh-keygen -R <ip>`.

## Phase 3 — Réseau du bastion (console VMware, root)

```bash
cat > /etc/network/interfaces << 'EOF'
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens32
iface ens32 inet static
    address 192.168.60.10/24
    gateway 192.168.60.2

auto ens34
iface ens34 inet static
    address 10.10.10.1/24
EOF
echo "nameserver 192.168.60.2" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
printf '10.10.10.10 lab-web01 web\n10.10.10.20 lab-db01 db\n' >> /etc/hosts
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-ip-forward.conf
reboot
```
> ⚠️ Piège : en passant en IP fixe, `resolv.conf` devient **vide** → plus de DNS. Toujours le remplir.

Après reboot : `ip a`, `ip route` (→ `default via 192.168.60.2`), `sysctl net.ipv4.ip_forward` (→ 1), `ping -c 3 deb.debian.org`.
Puis NAT : voir le `nftables.conf` final (phase 9) — en phase 3, seule la table `ip nat` suffit.

## Phase 4 — Web et DB dans VMnet2

1. **VMware : carte réseau → Personnalisé VMnet2** (⚠️ piège : l'IP s'affiche même si le « câble » est sur le mauvais réseau — si le ping du bastion échoue, vérifier le VMnet).
2. Root, sur web (`.10`) puis db (`.20`) :
   ```bash
   cat > /etc/network/interfaces << 'EOF'
   source /etc/network/interfaces.d/*

   auto lo
   iface lo inet loopback

   auto ens32
   iface ens32 inet static
       address 10.10.10.10/24
       gateway 10.10.10.1
   EOF
   echo "nameserver 192.168.60.2" > /etc/resolv.conf
   echo "nameserver 1.1.1.1" >> /etc/resolv.conf
   reboot
   ```
   Sur web uniquement : `echo "10.10.10.20 lab-db01 db" >> /etc/hosts`
3. Diagnostic dans l'ordre : `ping 10.10.10.1` (local) → `ping 1.1.1.1` (routage/NAT) → `ping deb.debian.org` (DNS).
4. Depuis le PC : `ping 10.10.10.10` doit **échouer** (isolation).

## Phase 5 — Clé SSH et durcissement du bastion

PC (PowerShell) :
```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.ssh"
ssh-keygen -t ed25519 -C "bastion-key" -f "$env:USERPROFILE\.ssh\bastion_key"
type "$env:USERPROFILE\.ssh\bastion_key.pub" | ssh stephane@192.168.60.10 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
> `ssh-copy-id` n'existe pas sous Windows → la ligne `type … | ssh …` le remplace.
> Syntaxe : `ssh utilisateur@ip` (collés, sans espace).

Bastion (root) : fichier `configs/bastion/10-bastion.conf` collé via `cat > /etc/ssh/sshd_config.d/10-bastion.conf << 'EOF'`, puis :
```bash
sshd -t && systemctl restart ssh
```
Tests (2e fenêtre) : `ssh -i … stephane@192.168.60.10` ✅ · `ssh -o PubkeyAuthentication=no stephane@192.168.60.10` → `Permission denied (publickey)` ✅

## Phase 6 — Carnet SSH et rebond

1. `notepad "$env:USERPROFILE\.ssh\config"` → **ajouter** (sans effacer les autres fiches `Host`) le contenu de `configs/client/ssh_config.example`. Vérifier que le fichier s'appelle `config` et pas `config.txt`.
2. `ssh bastion` doit entrer directement.
3. **Depuis le PC** (pas depuis le bastion !) :
   ```powershell
   type "$env:USERPROFILE\.ssh\bastion_key.pub" | ssh web "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   type "$env:USERPROFILE\.ssh\bastion_key.pub" | ssh db  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   ```
4. Durcir web et db (root) : `configs/web/10-hardening.conf` → `/etc/ssh/sshd_config.d/10-hardening.conf`, puis `sshd -t && systemctl restart ssh`.

## Phase 7 — Nginx + tunnel

Web (root) :
```bash
apt update && apt install -y nginx
systemctl enable --now nginx
echo "<h1>Boulangerie Martin - VM Web via bastion</h1>" > /var/www/html/index.html
```
Depuis le bastion : `curl http://10.10.10.10`.
Depuis le PC : `ssh -N -L 8080:10.10.10.10:80 bastion` → http://localhost:8080 (voir [tunnel-ssh.md](tunnel-ssh.md)).

## Phase 8 — MariaDB

DB (root) :
```bash
apt update && apt install -y mariadb-server
systemctl enable --now mariadb
DBPASS=$(tr -dc 'A-Za-z0-9' </dev/urandom | head -c 20)
echo "MOT DE PASSE webuser A NOTER : $DBPASS"
mariadb <<EOF
DELETE FROM mysql.global_priv WHERE User='';
DROP DATABASE IF EXISTS test;
CREATE DATABASE appdb;
CREATE USER 'webuser'@'10.10.10.10' IDENTIFIED BY '$DBPASS';
GRANT ALL PRIVILEGES ON appdb.* TO 'webuser'@'10.10.10.10';
FLUSH PRIVILEGES;
EOF
sed -i 's/^bind-address.*/bind-address = 10.10.10.20/' /etc/mysql/mariadb.conf.d/50-server.cnf
systemctl restart mariadb
ss -tlnp | grep 3306
```
Web : `apt install -y mariadb-client` (root), puis en `stephane` :
`mariadb -h db -u webuser -p appdb -e "SELECT 'Connexion OK depuis Web' AS test;"`
> Ne jamais publier de mot de passe, même dans une capture.

## Phase 9 — Pare-feu nftables

Fichiers : `configs/bastion/nftables.conf`, `configs/web/nftables.conf`, `configs/db/nftables.conf`.
Méthode sur chaque VM (root, console VMware ouverte) :
```bash
cat > /etc/nftables.conf << 'EOF'
… contenu du fichier …
EOF
nft -c -f /etc/nftables.conf && systemctl enable --now nftables && systemctl restart nftables && nft list ruleset
```
Ordre : **bastion → web → db**, en testant `ssh bastion`, `ssh web`, `ssh db` après chaque machine.

## Phase 10 — Vérifications

Voir la checklist de [preuves.md](preuves.md).

## Dépannage express

| Symptôme | Cause probable |
|---|---|
| `Permission non accordée`, `commande introuvable` (sshd, reboot) | Pas root → `su -` |
| `ping: invalid argument` | Oubli du chiffre après `-c` (`ping -c 3 …`) |
| Invite `>` inattendue | Guillemet non fermé → Ctrl + C |
| `Échec temporaire dans la résolution du nom` | `resolv.conf` vide |
| `ping 1.1.1.1` OK mais pas les noms | DNS |
| Ping du bastion impossible depuis web/db | Carte VMware sur le mauvais VMnet |
| `Could not resolve hostname web` | Commande lancée depuis le bastion au lieu du PC, ou `config` absent |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | VM recréée → `ssh-keygen -R <ip>` |
| `ssh @user ip` refusé | Syntaxe : `ssh user@ip` |
