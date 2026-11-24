# Preuves de fonctionnement

Chaque capture est recadrée sur le terminal ou la page concernée. Aucun mot de passe n'apparaît.

## Phase 6 — Carnet SSH et rebond par le bastion

`ssh bastion` échoue avant la création de `~/.ssh/config`, puis entre directement après :

![Carnet SSH avant / après](../captures/06-config-ssh-avant-apres.png)

`ssh web` et `ssh db` traversent le bastion, sans mot de passe :

![Rebond SSH vers web et db](../captures/06-rebond-ssh-web-db.png)

## Phase 7 — Nginx et tunnel SSH

![Nginx actif](../captures/07-nginx-active-running.png)

Le bastion voit la page de web :

![curl depuis le bastion](../captures/07-curl-depuis-bastion.png)

Le PC affiche la page via `ssh -N -L 8080:10.10.10.10:80 bastion` alors qu'il ne peut pas joindre web directement :

![Tunnel SSH dans Firefox](../captures/07-tunnel-firefox-localhost8080.png)

## Phase 8 — MariaDB

MariaDB n'écoute que sur l'adresse interne :

![MariaDB écoute sur 10.10.10.20](../captures/08-mariadb-ecoute-10.10.10.20.png)

Le compte `webuser` fonctionne depuis web :

![Connexion web vers db](../captures/08-connexion-web-vers-db.png)

## Phase 9 — Pare-feu nftables

![Règles du bastion](../captures/09-bastion-nft-ruleset.png)

![SSH toujours fonctionnel après pare-feu](../captures/09-bastion-tests-apres-parefeu.png)

![Règles de web](../captures/09-web-nft-ruleset.png)

![MariaDB toujours accessible depuis web après pare-feu](../captures/09-mariadb-ok-apres-parefeu.png)

## Phase 10 — Vérifications finales

| Test | Attendu | Résultat |
|---|---|---|
| `ping 10.10.10.10` depuis le PC | Échec (isolation) | ✅ 100 % de perte |
| `ssh -o PubkeyAuthentication=no stephane@192.168.60.10` | `Permission denied (publickey)` | ✅ |
| `ssh bastion`, `ssh web`, `ssh db` | Connexion par clé, rebond | ✅ |
| `ip -br a` / `ip route` / `ip_forward` sur le bastion | 2 adresses, sortie `.2`, routage = 1 | ✅ |
| `curl http://10.10.10.10` depuis le bastion | Page HTML | ✅ |
| Tunnel + `http://localhost:8080` | Page HTML | ✅ |
| `mariadb -h db -u webuser` depuis web | Connexion OK | ✅ |
| `nc -zv -w 3 10.10.10.20 3306` depuis le bastion | **Bloqué** (timed out) | ✅ |

Isolation : le PC ne joint pas le réseau interne.

![Ping PC vers web](../captures/10-isolation-ping-pc-vers-web.png)

État réseau du bastion :

![Bastion : adresses, routes, routage](../captures/10-bastion-etat-reseau-routage.png)

Défense en profondeur : même le bastion ne peut pas ouvrir la base.

![nc bastion vers db bloqué](../captures/10-nc-bastion-vers-db-bloque.png)
