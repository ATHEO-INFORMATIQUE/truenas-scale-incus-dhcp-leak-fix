# TrueNAS SCALE — Correction de la fuite DHCP d'Incus sur le réseau local

> **En bref** — Après un redémarrage ou une mise à jour, le bridge Incus (`incusbr0`) de TrueNAS SCALE peut faire fuiter son serveur DHCP interne sur votre réseau local, cassant la connectivité de tous vos appareils. Ce guide explique comment détecter et corriger définitivement le problème.

---

## Le problème

TrueNAS SCALE utilise **Incus** (un système de containers/VMs) en interne. Son bridge réseau `incusbr0` peut se retrouver à écouter sur votre interface réseau physique et répondre aux requêtes DHCP **à la place de votre routeur/box**.

Quand cela arrive, les appareils de votre réseau reçoivent :
- Un masque de sous-réseau cassé `/30` (`255.255.255.252`) au lieu de `/24` (`255.255.255.0`)
- Une passerelle pointant vers TrueNAS au lieu de votre routeur
- Des adresses IPv6 inattendues même si IPv6 est désactivé sur votre routeur
- Plus d'accès internet

Le problème est **aléatoire et intermittent** — il réapparaît après chaque redémarrage ou mise à jour de TrueNAS car Incus réinitialise sa configuration réseau à ses valeurs par défaut.

---

## Symptômes

- Appareils WiFi et filaires n'obtenant plus d'adresse IP
- Message "Internet peut ne pas être disponible" sur Android
- Appareils recevant des adresses IPv6 alors que c'est désactivé sur le routeur
- Certains appareils obtiennent une IP mais avec le masque `255.255.255.252` au lieu de `255.255.255.0` → pas d'accès internet
- Le problème disparaît et revient aléatoirement, souvent après un redémarrage de TrueNAS

---

## Cause réelle

Incus configure `incusbr0` avec une IP dans le même sous-réseau que le réseau local (ex: `192.168.1.x/30`), ce qui fait que son serveur DHCP `dnsmasq` répond aux requêtes DHCP broadcast de tous les appareils du réseau au lieu d'être isolé au bridge interne.

Il envoie également des annonces de routeur (`--enable-ra`) ce qui explique les adresses IPv6 non désirées.

---

## Détection

Depuis n'importe quelle machine Linux sur votre réseau (ex: un Raspberry Pi) :

```bash
sudo nmap --script broadcast-dhcp-discover -e eth0 2>/dev/null \
  | grep -E "Server Identifier|Router|Subnet Mask|Domain Name"
```

**Résultat normal** (un seul serveur, masque /24) :
```
Server Identifier: 192.168.1.254
Subnet Mask: 255.255.255.0
Router: 192.168.1.254
```

**Résultat anormal** (deux serveurs ou masque /30) :
```
Server Identifier: 192.168.1.254
Subnet Mask: 255.255.255.0
Router: 192.168.1.254
Server Identifier: 192.168.1.x    ← TrueNAS parasite
Subnet Mask: 255.255.255.252      ← masque cassé
Domain Name: incus                ← signature Incus
```

Vous pouvez aussi vérifier directement sur TrueNAS :

```bash
# Doit afficher incusbr0:67, PAS 0.0.0.0:67
sudo ss -ulnp | grep :67

# Trouver le processus dnsmasq parasite
ps aux | grep dnsmasq
```

---

## Correction permanente

### Étape 1 — Créer le script de correction

Stockez le script sur un pool ZFS persistant (survit aux mises à jour) :

```bash
sudo mkdir -p /mnt/VOTRE_POOL/scripts
sudo nano /mnt/VOTRE_POOL/scripts/fix-incus-network.sh
```

Collez ce contenu (remplacez `10.10.10.1/24` par n'importe quel sous-réseau privé qui ne chevauche pas votre réseau local) :

```bash
#!/bin/bash
# Correction de la fuite DHCP Incus — isoler incusbr0 sur un sous-réseau privé
sleep 10
incus network set incusbr0 ipv4.address 10.10.10.1/24
incus network set incusbr0 ipv4.dhcp true
incus network set incusbr0 ipv6.address none
incus network set incusbr0 ipv6.nat false
incus network set incusbr0 dns.mode managed
```

Rendez-le exécutable :

```bash
sudo chmod +x /mnt/VOTRE_POOL/scripts/fix-incus-network.sh
```

---

### Étape 2 — Enregistrer le script dans TrueNAS

Utilisez le mécanisme officiel init/shutdown de TrueNAS — il survit aux mises à jour :

```bash
sudo midclt call initshutdownscript.create '{
  "type": "SCRIPT",
  "script": "/mnt/VOTRE_POOL/scripts/fix-incus-network.sh",
  "when": "POSTINIT",
  "enabled": true,
  "timeout": 10,
  "comment": "Fix Incus DHCP leak"
}'
```

Vous pouvez vérifier qu'il est bien enregistré dans **WebUI TrueNAS → Système → Paramètres avancés → Scripts Init/Shutdown**.

---

### Étape 3 — Appliquer immédiatement (sans redémarrage)

```bash
sudo /mnt/VOTRE_POOL/scripts/fix-incus-network.sh
```

---

### Étape 4 — Vérifier la correction

```bash
sudo ss -ulnp | grep :67
```

✅ **Correct** — DHCP isolé sur le bridge interne :
```
UNCONN 0  0  0.0.0.0%incusbr0:67  0.0.0.0:*  users:(("dnsmasq",...))
```

❌ **Toujours cassé** — DHCP exposé sur toutes les interfaces :
```
UNCONN 0  0  0.0.0.0:67  0.0.0.0:*  users:(("dnsmasq",...))
```

Relancez la commande nmap depuis une autre machine pour confirmer que seul votre routeur répond aux requêtes DHCP.

---

## Après les mises à jour TrueNAS

Après chaque mise à jour TrueNAS, vérifiez :

```bash
sudo ss -ulnp | grep :67
```

Si le problème revient, le script init enregistré le corrigera automatiquement au prochain redémarrage. Pour corriger immédiatement sans redémarrer :

```bash
sudo /mnt/VOTRE_POOL/scripts/fix-incus-network.sh
```

---

## Environnement concerné

| Composant | Version |
|-----------|---------|
| TrueNAS SCALE | 25.04+ |
| Incus | intégré à TrueNAS |
| Affecté depuis | TrueNAS SCALE Electric Eel+ |

---

## Pourquoi ne pas simplement désactiver Incus ?

Le système d'applications de TrueNAS (Apps) s'appuie sur Incus en interne — le désactiver casserait complètement le système d'applications. La correction ci-dessus isole Incus sur un sous-réseau privé tout en maintenant toutes les fonctionnalités.

---

## Contribution

Si ce guide vous a été utile ou si vous avez des améliorations à proposer, n'hésitez pas à ouvrir une PR ou une issue. Si vous êtes sur une version différente de TrueNAS et que la correction doit être adaptée, partagez vos découvertes.

---

## Licence

MIT — faites-en ce que vous voulez.
