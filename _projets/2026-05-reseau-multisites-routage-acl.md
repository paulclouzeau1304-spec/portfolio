---
title: "Réseau multisite : routage statique et filtrage par ACL"
date: 2026-05-01
cadre: "Projet de Bac Pro CIEL"
resume: "Interconnexion de quatre réseaux par trois routeurs Cisco, avec routes statiques et listes de contrôle d'accès qui protègent le réseau du support technique."
competences: [c1, c5]
---

## Contexte

Dans le cadre de mon baccalauréat professionnel CIEL en alternance, j'ai réalisé un projet qui consistait à relier plusieurs réseaux locaux répartis sur deux sites, puis à protéger le réseau du support technique : les autres réseaux ne doivent pas pouvoir l'atteindre, alors que le support technique doit pouvoir tester la connexion vers tous les postes. J'ai présenté ce projet à l'oral.

## Conditions et moyens

Le réseau a d'abord été conçu et testé sur Cisco Packet Tracer 9, puis monté sur du matériel physique. Équipements : un routeur Cisco 1941, deux routeurs Cisco 2911, quatre commutateurs Cisco 2960 et cinq postes clients, reliés par des câbles croisés entre routeurs et des câbles droits vers les commutateurs.

## Description de l'activité

**1. Plan d'adressage.** Découpage du réseau 192.168.1.0/24 en quatre sous-réseaux de 62 hôtes (masque /26) pour les réseaux locaux, et deux liaisons point à point en /30 entre les routeurs.

| Réseau | Adresse | Passerelle | Postes |
|---|---|---|---|
| LAN de Router0 | 192.168.1.0/26 | 192.168.1.1 | PC0, PC1 |
| LAN de Router1 | 192.168.1.64/26 | 192.168.1.65 | PC2 |
| Site 2, LAN support technique | 192.168.1.128/26 | 192.168.1.129 | PC13 |
| Site 2, LAN Switch5 | 192.168.1.192/26 | 192.168.1.193 | PC16 |
| Liaison Router0 – Router1 | 10.0.0.0/30 | – | – |
| Liaison Router1 – Router2 | 10.0.0.4/30 | – | – |

**2. Adressage des interfaces** des routeurs, par exemple sur Router1 :

```
interface GigabitEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
```

**3. Routage statique.** Sur chaque routeur, une route vers chaque réseau distant, avec pour prochain saut le routeur voisin. Par exemple sur Router2 :

```
ip route 192.168.1.0 255.255.255.192 10.0.0.5
ip route 192.168.1.64 255.255.255.192 10.0.0.5
```

**4. Filtrage par ACL étendues.** Sur chaque routeur, une ACL appliquée en entrée bloque le trafic vers le site 2 mais laisse passer les réponses aux ping (`echo-reply`). Par exemple sur Router2, côté liaison WAN :

```
access-list 103 permit icmp any 192.168.1.128 0.0.0.63 echo-reply
access-list 103 deny ip any 192.168.1.128 0.0.0.63
access-list 103 permit ip any any
interface GigabitEthernet0/0
 ip access-group 103 in
```

**5. Tests** par ping depuis les postes, puis sauvegarde des configurations avec `copy running-config startup-config`.

**6. Montage sur matériel physique** et présentation du réseau à l'oral.

## Productions et preuves

![Topologie du réseau dans Packet Tracer : trois routeurs, quatre commutateurs et cinq postes]({{ "/images/reseau-topologie.png" | relative_url }})

![Ping depuis PC13, au support technique, vers PC0 : trois réponses reçues]({{ "/images/reseau-ping-pc13.png" | relative_url }})

![Pings depuis PC0 : PC2 répond, PC13 est inaccessible, Router0 renvoie « Destination host unreachable »]({{ "/images/reseau-ping-pc0.png" | relative_url }})

| Test | Résultat obtenu |
|---|---|
| PC0 vers PC2 (192.168.1.74) | Réussi, 4 réponses sur 4 au second essai |
| PC13 vers PC0 (192.168.1.10) | Réussi : le support technique peut tester les postes des autres réseaux |
| PC0 vers PC13 (192.168.1.138) | Bloqué : l'ACL 101 de Router0 refuse le paquet, et Router0 (192.168.1.1) répond « Destination host unreachable » |

Les premiers paquets perdus (« Request timed out ») ne sont pas une panne : pendant ce temps, les postes et les routeurs résolvent les adresses MAC avec le protocole ARP. Dès que c'est fait, toutes les réponses arrivent.

## Ce que j'en retiens

En relisant les configurations, j'ai trouvé des routes statiques dont le prochain saut n'était pas le routeur voisin, par exemple sur Router1 une route vers 192.168.1.0/26 qui passait par 192.168.1.193, une adresse de Router2. Le réseau semblait fonctionner, mais ces routes en double pouvaient renvoyer une partie des paquets dans la mauvaise direction. Je les ai supprimées avec `no ip route`. J'ai retenu que le prochain saut d'une route statique doit toujours être l'adresse du voisin direct, sur la liaison qui le relie au routeur.
