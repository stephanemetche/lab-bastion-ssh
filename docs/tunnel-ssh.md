# Fiche mémo — Tunnel SSH local (`-L`) à travers un bastion

## À quoi ça sert

Afficher sur **mon PC** un service qui tourne sur un serveur **caché** (non joignable directement),
en passant par le bastion. Le trafic est **chiffré** de bout en bout et le serveur n'est jamais
exposé. Cas réels : consulter GLPI, Zabbix, une interface d'admin, une base de données…

**Image :** un tuyau secret. Un bout est branché sur mon PC (port local), il traverse le bastion
et ressort devant la porte du service sur le serveur caché.

## La commande (PowerShell ou Linux, identique)

```
ssh -N -L <port_local>:<ip_du_serveur_caché>:<port_du_service> bastion
```

Exemple du lab (Nginx sur lab-web01) :

```
ssh -N -L 8080:10.10.10.10:80 bastion
```

| Morceau | Sens |
|---|---|
| `-L` | **L**ocal forwarding : ouvrir un port sur MON PC |
| `8080` | Port côté PC (n'importe quel port libre > 1024) |
| `10.10.10.10:80` | Destination, **vue depuis le bastion** |
| `-N` | Pas de terminal, seulement le tunnel (la fenêtre semble figée : normal) |
| `bastion` | Alias défini dans `~/.ssh/config` (clé + IP du bastion) |

Puis dans le navigateur : **http://localhost:8080**
Fermer le tunnel : **Ctrl + C** dans la fenêtre.

## Autres exemples utiles

| Besoin | Commande |
|---|---|
| Site web interne | `ssh -N -L 8080:10.10.10.10:80 bastion` → http://localhost:8080 |
| Site HTTPS interne | `ssh -N -L 8443:10.10.10.10:443 bastion` → https://localhost:8443 |
| Base MariaDB interne | `ssh -N -L 3307:10.10.10.20:3306 bastion` → client SQL sur `127.0.0.1:3307` |
| GLPI / Zabbix internes | `ssh -N -L 8081:<ip>:80 bastion` → http://localhost:8081 |
| Tunnel en arrière-plan (Linux) | ajouter `-f` : `ssh -f -N -L …` |

## Points d'attention

- L'adresse de destination est résolue **par le bastion** : il faut que le bastion puisse la joindre.
- Si le port local est déjà pris : « bind: Address already in use » → choisir un autre port local.
- Le pare-feu du serveur caché doit autoriser le port depuis **l'IP du bastion** (c'est le bastion
  qui se présente à la porte, pas le PC).
- `AllowTcpForwarding` ne doit **pas** être à `no` sur le bastion, sinon le tunnel est refusé.

## Preuve

![Tunnel SSH vers Nginx dans Firefox](../captures/07-tunnel-firefox-localhost8080.png)
