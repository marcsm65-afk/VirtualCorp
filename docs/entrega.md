# Activitat 1: desplegament dels quatre routers MikroTik

**Alumne:** Marc Sánchez Martínez
**Delegació:** 24 (rang 10.24.0.0/16)
**Repositori:** https://github.com/marcsm65-afk/VirtualCorp

## 1. Què he fet

He agafat el repositori inicial, l'he adaptat a la delegació 24 i he desplegat els quatre routers (mikrotik0 a mikrotik3) a VirtualBox amb `make deploy-routers`. Després he comprovat que es veuen entre ells i que tenen sortida a Internet.

## 2. Disseny d'adreçament

El repositori venia amb les adreces de la delegació 1 (10.1.x.x) escrites directament a `infrastructure/bash/deploy_routers.sh`. Les he canviat a 10.24.x.x seguint l'enunciat.

| Router | Interfície | Xarxa interna | Adreça |
|---|---|---|---|
| mikrotik0 | ether2 | r0-r1 | 10.24.0.225/30 |
| mikrotik0 | ether3 | dmz | 10.24.0.193/27 |
| mikrotik0 | ether4 | priv | 10.24.0.129/26 |
| mikrotik1 | ether2 | r0-r1 | 10.24.0.226/30 |
| mikrotik1 | ether3 | r1-r2 | 10.24.0.229/30 |
| mikrotik1 | ether4 | pl0 | 10.24.0.1/25 |
| mikrotik2 | ether2 | r1-r2 | 10.24.0.230/30 |
| mikrotik2 | ether3 | r2-r3 | 10.24.0.233/30 |
| mikrotik2 | ether4 | pl1 | 10.24.1.1/24 |
| mikrotik2 | ether5 | pl2 | 10.24.2.1/24 |
| mikrotik3 | ether2 | r2-r3 | 10.24.0.234/30 |
| mikrotik3 | ether3 | pl3 | 10.24.3.1/25 |
| mikrotik3 | ether4 | pl3-direccio | 10.24.3.129/25 |

Passarel·les per defecte: mikrotik1 usa 10.24.0.225, mikrotik2 usa 10.24.0.229 i mikrotik3 usa 10.24.0.233. mikrotik0 surt a Internet amb NAT (masquerade) per ether1.

Decisions que he près:
- **Planta 3 partida en dos /25:** el repo ja la tenia així, i m'encaixa amb l'enunciat perquè la xarxa de direcció (10.24.3.128/25) queda aïllada de la resta d'usuaris de la planta.
- **Encaminament:** el repo usa RIP (redistribuint les xarxes connectades), i l'he deixat així perquè és simple i amb quatre routers en cadena funciona bé.

## 3. Accés SSH

- **El meu accés:** el script genera una clau `vcorp_sysadmin` i la instal·la a l'usuari `sysadmin` de cada router. Entro amb `ssh -i ~/.ssh/vcorp_sysadmin -p 2200 sysadmin@127.0.0.1`.
- **Accés del professor:** he afegit al bootstrap (`infrastructure/bash/router_base.sh`) la instal·lació de `configs/ssh/vcorp_auditor.pub` als quatre routers. Es crea un usuari `auditor` (grup `full`) amb una contrasenya aleatòria que no es guarda enlloc, així que només es pot entrar amb la clau. No he tocat el contingut de la clau del professor.
- Les claus privades queden a `~/.ssh` i no s'han pujat al repositori.

## 4. Problemes que he trobat i com els he resolt

1. **Faltava la zona privada.** L'enunciat reserva 10.24.0.128/26 (pve01, dbsrv, infosrv), però el repo no la tenia a cap router. L'he afegit a mikrotik0 com a xarxa `priv` amb la 10.24.0.129/26, perquè el CPD és a la planta baixa, al costat de la DMZ.
2. **`make ssh-keys` sobreescrivia la clau del professor.** El script generava una parella `vcorp_auditor` nova i copiava la pública sobre `configs/ssh/vcorp_auditor.pub`, i l'enunciat diu que no es pot substituir. He eliminat aquesta línia del script `create_ssh_keys.sh`.
3. **VirtualBox no arrencava les VMs** (error `VERR_VMX_IN_VMX_ROOT_MODE`). El mòdul KVM de Linux ocupava la virtualització per maquinari. Ho he solucionat descarregant els mòduls amb `sudo modprobe -r kvm_intel kvm_amd kvm`, i després els he blacklistejat a `/etc/modprobe.d/blacklist-kvm.conf` perquè no es tornin a carregar en reiniciar.
4. **Un `make deploy-routers` fallit deixa VMs a mig fer.** El script se salta els routers que ja existeixen, així que quan va fallar vaig haver d'executar `make delete-routers` abans de tornar a desplegar.
5. **Accés als routers 1, 2 i 3.** Només mikrotik0 té NAT cap al meu PC (port 2200). Els altres tres perden aquest accés quan acaba el desplegament, i hi vaig entrar des de la consola de VirtualBox o saltant a través de mikrotik0.

## 5. Validació

mikrotik0 amb les adreces i rutes correctes (`/ip address print` i `/ip route print`):

[sysadmin@mikrotik0] > /ip address print
Flags: D - DYNAMIC
Columns: ADDRESS, NETWORK, INTERFACE, VRF
#   ADDRESS         NETWORK      INTERFACE  VRF 
0   10.24.0.225/30  10.24.0.224  ether2     main
1   10.24.0.193/27  10.24.0.192  ether3     main
2   10.24.0.129/26  10.24.0.128  ether4     main
3 D 10.0.2.15/24    10.0.2.0     ether1     main
[sysadmin@mikrotik0] > /ip route print  
Flags: D - DYNAMIC; A - ACTIVE; c - CONNECT, r - RIP, d - DHCP
Columns: DST-ADDRESS, GATEWAY, ROUTING-TABLE, DISTANCE
    DST-ADDRESS     GATEWAY             ROUTING-TABLE  DISTANCE
DAd 0.0.0.0/0       10.0.2.2            main                  1
DAc 10.0.2.0/24     ether1              main                  0
DAr 10.24.0.0/25    10.24.0.226%ether2  main                120
DAc 10.24.0.128/26  ether4              main                  0
DAc 10.24.0.192/27  ether3              main                  0
DAc 10.24.0.224/30  ether2              main                  0
DAr 10.24.0.228/30  10.24.0.226%ether2  main                120
DAr 10.24.0.232/30  10.24.0.226%ether2  main                120
DAr 10.24.1.0/24    10.24.0.226%ether2  main                120
DAr 10.24.2.0/24    10.24.0.226%ether2  main                120
DAr 10.24.3.0/25    10.24.0.226%ether2  main                120
DAr 10.24.3.128/25  10.24.0.226%ether2  main                120


Pings des de mikrotik0 a les adreces dels altres routers (10.24.0.226, 10.24.0.230 i 10.24.0.234) i a Internet (1.1.1.1 i google.com): tots 4/4 sense pèrdues. També he fet els pings des de mikrotik1, mikrotik2 i mikrotik3 cap als altres tres routers i a 1.1.1.1, i funcionen tots.

Flags: D - DYNAMIC
Columns: ADDRESS, NETWORK, INTERFACE, VRF
#   ADDRESS         NETWORK      INTERFACE  VRF 
0   10.24.0.225/30  10.24.0.224  ether2     main
1   10.24.0.193/27  10.24.0.192  ether3     main
2   10.24.0.129/26  10.24.0.128  ether4     main
3 D 10.0.2.15/24    10.0.2.0     ether1     main
[sysadmin@mikrotik0] > ping 10.24.0.225 count=4
  SEQ HOST                                     SIZE TTL TIME       STATUS      
    0 10.24.0.225                                56  64 57us      
    1 10.24.0.225                                56  64 94us      
    2 10.24.0.225                                56  64 88us      
    3 10.24.0.225                                56  64 95us      
    sent=4 received=4 packet-loss=0% min-rtt=57us avg-rtt=83us max-rtt=95us 

[sysadmin@mikrotik0] > ping 10.24.0.230 count=4
  SEQ HOST                                     SIZE TTL TIME       STATUS      
    0 10.24.0.230                                56  63 1ms354us  
    1 10.24.0.230                                56  63 1ms985us  
    2 10.24.0.230                                56  63 1ms882us  
    3 10.24.0.230                                56  63 2ms356us  
    sent=4 received=4 packet-loss=0% min-rtt=1ms354us avg-rtt=1ms894us 
   max-rtt=2ms356us 

[sysadmin@mikrotik0] > ping 10.24.0.234 count=4
  SEQ HOST                                     SIZE TTL TIME       STATUS      
    0 10.24.0.234                                56  62 2ms572us  
    1 10.24.0.234                                56  62 2ms703us  
    2 10.24.0.234                                56  62 2ms474us  
    3 10.24.0.234                                56  62 2ms990us  
    sent=4 received=4 packet-loss=0% min-rtt=2ms474us avg-rtt=2ms684us 
   max-rtt=2ms990us 

[sysadmin@mikrotik0] > ping 1.1.1.1 count=4
  SEQ HOST                                     SIZE TTL TIME       STATUS      
    0 1.1.1.1                                    56 254 16ms941us 
    1 1.1.1.1                                    56 254 16ms590us 
    2 1.1.1.1                                    56 254 16ms613us 
    3 1.1.1.1                                    56 254 18ms227us 
    sent=4 received=4 packet-loss=0% min-rtt=16ms590us avg-rtt=17ms92us 
   max-rtt=18ms227us 

Usuaris i claus SSH (`/user print` i `/user ssh-keys print` a mikrotik0), on es veuen `sysadmin` i `auditor`, cadascun amb la seva clau:

[sysadmin@mikrotik0] > /user print
Flags: E - EXPIRED
Columns: NAME, GROUP, LAST-LOGGED-IN, INACTIVITY-POLICY
#   NAME      GROUP  LAST-LOGGED-IN       INACTIVITY-POLICY
;;; system default user
0 E admin     full   2026-11-19 17:48:41  none             
;;; sysadmin user
1   sysadmin  full   2026-11-23 18:42:42  none             
;;; professor audit access
2   auditor   full                        none             
[sysadmin@mikrotik0] > /user ssh-keys print
Columns: USER, KEY-TYPE, BITS, INFO
#  USER      KEY-TYPE  BITS  INFO                      
0  sysadmin  ed25519    256  vcorp_sysadmin@virtualcorp
1  auditor   ed25519    256  vcorp_auditor@virtualcorp 

## 6. Limitacions

- Només mikrotik0 és accessible directament des del PC amb el port 2200. Per arribar a mikrotik1-3 s'ha de passar per mikrotik0.
- La contrasenya `sysadmin` dels routers surt escrita als scripts del repositori. Serveix per al laboratori, però en un entorn real no es faria així.

