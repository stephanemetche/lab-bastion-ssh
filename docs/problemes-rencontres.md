# Problèmes rencontrés et diagnostic

Les vraies pannes du lab, dans l'ordre, avec la méthode pour les trouver.
Méthode générale : **tester du plus proche au plus lointain** (carte → passerelle → internet par IP → DNS).

| # | Symptôme | Diagnostic | Cause | Correction |
|---|---|---|---|---|
| 1 | L'installeur Debian : « La configuration automatique a échoué » | Éditeur de réseau virtuel : colonne DHCP de VMnet8 = « - » | Un ancien TP avait désactivé le DHCP de VMnet8 | Réactiver le service DHCP local sur VMnet8 |
| 2 | « Miroir de l'archive Debian corrompu », `Erreur temporaire de résolution` | `ip route` → `default via 192.168.60.254` ; `ping 192.168.60.2` et `ping 1.1.1.1` échouent | Passerelle NAT VMware réglée sur `.254`, adresse déjà prise par le DHCP | Paramètres du NAT → passerelle `192.168.60.2`, puis redémarrer la VM |
| 3 | `ping deb.debian.org` échoue après passage en IP fixe, mais `ping 1.1.1.1` répond | `cat /etc/resolv.conf` → aucune ligne `nameserver` | En IP statique, plus personne ne remplit le DNS | Écrire `nameserver 192.168.60.2` et `nameserver 1.1.1.1` |
| 4 | Web a bien `10.10.10.10` mais ne ping pas le bastion | `ip a` correct ; paramètres VMware de la VM | Carte VMware restée sur NAT au lieu de VMnet2 (l'IP s'affiche quand même) | Carte réseau → Personnalisé VMnet2 |
| 5 | `Permission non accordée`, `sshd : commande introuvable`, `reboot : commande introuvable` | Invite en `$` | Commandes d'administration lancées sans être root | `su -` (avec le tiret) avant |
| 6 | `ssh @stephane 192.168.60.131` refusé pour l'utilisateur `steph` | Message `steph@…'s password` | Syntaxe : SSH a pris le nom Windows | `ssh stephane@192.168.60.131` |
| 7 | `type : non trouvé` et `Could not resolve hostname web` | Invite `stephane@lab-bastion01` | Commande Windows lancée **dans** le bastion | `exit`, relancer depuis le PC |
| 8 | Fichier `sshd` non pris en compte / erreurs | Relecture du fichier | Fautes de frappe (`sshd_configd`, « Authentification ») | Écrire les fichiers par `cat << 'EOF'` collé |
| 9 | `REMOTE HOST IDENTIFICATION HAS CHANGED` | — | Clés hôte régénérées sur les clones (voulu) | `ssh-keygen -R <ip>` |

## Leçons retenues

- Une adresse IP affichée ne prouve pas que le « câble » virtuel est sur le bon réseau.
- `ping 1.1.1.1` OK mais pas les noms = problème de DNS, pas de routage.
- Les réglages VMware (VMnet, NAT, DHCP) sont **globaux** : un TP peut casser le suivant.
- Les clones doivent avoir leur propre hostname, `machine-id` et clés SSH hôte.
