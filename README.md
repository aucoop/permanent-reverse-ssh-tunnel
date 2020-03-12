# Permanent ssh reverse-tunnel tool using autossh

## Estructura de la xarxa

**Refèrencies:**

* [Persistent reverse (NAT bypassing) SSH tunnel access with autossh](https://raymii.org/s/tutorials/Autossh_persistent_tunnels.html)
* [Bypassing corporate firewall with reverse ssh port forwarding](http://toic.org/blog/2009/reverse-ssh-port-forwarding/#.VbdKjWEvBGF)
* [Conexión a un servidor mediante un tunel inverso ssh](https://openwebinars.net/blog/conexion-servidor-mediante-tunel-inverso-ssh/)

### Idea general

Es vol tenir accés remot SSH permanent a una màquina a la qual no es pot accedir desde l'exterior. A continuació s'exemplifica la problemàtica amb una diagrama.

![reverse-ssh](/img/firewall_problematic.png)

El diagrama ilustra:
* Una màquina a la qual sempre es té accés desde l'exterior (Home pc en la imatge) que d'ara en endavant es dirà **middleman server**.

* Una màquina a la qual no es té accés desde l'exterior de la xarxa degut a un firewall (com exemplifica el dibuix), NAT, etc. A aquesta màquina (en el dibuix Office pc) l'anomenarem d'ara en endavant **restricted server**.



Una sol·lució a aquesta problemàtica és el reverse-tunneling. És un mètode senzill, però eficaç, ja que ha donat resultats amb una xarxa certament complexes.

Bàsicament, consisteix d'una connexió entre màquines port a port via ssh, amb la característica de que aquesta petició la realitza el middleman server i no la màquina desde la qual accedirem. Una imatge general que exemplifica aquesta idea:

![reverse-ssh](/img/reverese-ssh.png)

El que farem per conectar-nos al **servidor restringit** (servidor al qual no tenim accès), és establir una connexió desde el **servidor restringit** a una màquina que si tenim accès ssh desde l'exterior, és a dir el **middleman server**. Després utilitzarem aquesta connexió permanent per accedir desde una tercera màquina al **servidor restringit** via el **middleman server**. Amb una imatge espero que quedi molt més clar:

TODO actualitzar el diagrama amb IPs generals

![reverse-tuneling](/img/reverse-tuneling.png)


### Implementació: 

Per implementar el mètode explicat anteriorment es fa ús del programa `autossh`. Desde la [man page](https://linux.die.net/man/1/autossh):

> autossh is a program to start a copy of ssh and monitor it, restarting it as necessary should it die or stop passing traffic.

Bàsicament aixeca un tunel ssh i el manté sempre viu. Una vegada tenim això, cal configurar un daemon que aixequi aquest programa sempre que l'ordinador es reencengui, perdi la connexió, etc. Aquest servei s'ha configurat a través de `systemd`. L'autoaixecament, però s'explica a la secció [Auto-aixecament del tunel ssh](#auto-aixecament-del-tunel-ssh) al final del document.

#### Provant el tunel autossh

Les següents ordres s'executen al **server restringit**:

Primer, assegurar que està instalat el client `ssh` i el software `autossh`.

```bash
sudo apt-get install autossh ssh
```

Ara, es genera el parell de claus a través del programa `ssh-keygen`. Es generarà una clau RSA de 4096 bits sense passphrase per encriptar la clau privada. (*Això és simplement per simplicitat, en un futur es podria ~~s'hauria~~ d'encriptar i afegir al `ssh-keygen` per millorar la seguretat*).

```source
ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): ~/.ssh/restricted_server.key
Enter passphrase (empty for no passphrase): *leave empty*
Enter same passphrase again: *leave empty*
```

Ara cal afegir la clau pública del client al fitxer `authorized_hosts` del servidor. Hi han dos mètodes per fer-ho:
##### Utilitzant `ssh-copy-id`

  * A traves de la utilitat `ssh-copy-id` és la forma més senzilla de realitzar aquest procés. Per executar-ho:
  
    ```bash
    ssh-copy-id -i restricted_server.key.pub user@middleman_IP
    ```
    On `middleman_IP` és l'adreça IP del middleman server.


   
    Aquesta utilitat, però, no funciona si el middleman server no accepta contrasenyes per accedir. Això és degut a que , realment, això és el que fa `ssh-copy-id` *under the hood*:
  
    ```bash
    scp .ssh/id_rsa.pub user@other-host:
    ssh user@other-host 'cat id_rsa.pub >> .ssh/authorized_keys'
    ssh user@other-host 'rm id_rsa.pub'
    ```
##### Copiant la clau al fitxer `authorized_keys` en el middleman server

  * Si el middleman server no accepta passwords per accedir via ssh, simplement s'ha de copiar la clau pública del client al fitxer `authorized_keys` del middleman server. Per fer-ho, cal accedir desde una màquina que ja tingui accés via clau pública al middleman server i copiar la  clau pública de la màquina desitjat en una línia al final del fitxer `authorized_keys`.
  
    ```bash
    user@middleman_server: echo "*contingut del portapapeles, es a dir, la clau publica del client*" >> ~/.ssh/authorized_keys`
    ```

Ara, per establir el primer tunel utilitzem el programa autossh.

TODO

```bash
autossh -M 10984 -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no" -i /home/diadem/.ssh/nopwd -R 6666:localhost:22 aucoop@147.83.200.187
```

Desde una altra màquina qualsevol, per exemple el mòbil connectat a la estació base (3G/4G):

```bash
ssh -p 22 aucoop@147.83.200.187
```

Ara, dins del servidor a la UPC:

```bash
ssh -p 6666 diadem@127.0.0
```

Si tot va bé, hauries de veure un prompt per fer login a diadem/duniakato. Entra la password i endavant. Si això no funciona, revisa els valors de l'ordre i la configuració del ssh. També assegura't de que tots els passos s'han executat correctament.

#### Gestió de hosts i claus

Per afegir un nou client SSH cal que el **servidor** tingui la clau pública del **client** en el fitxer `authorized_keys`. Despres, el **client** ha de realitzar la connexio ssh utilitzant com a indentificador la seva clau privada.

Les ordres per realitzar el proces descrit abans son les seguents:

* Client side:

```source
user@client: ssh-keygen -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): client.key
Enter passphrase (empty for no passphrase): *leave empty*
Enter same passphrase again: *leave empty*
```

Aquesta ordre crea un parell de claus pública i privada. Un cop creades, cal copiar la **clau pública del client** i enganxarla al fitxer `authorized_keys` del servidor.

* Server side:

```bash
user@server: echo "*contenido del portapapeles, es decir, la clave publica del cliente*" >> ~/.ssh/authorized_keys`
```

* Client side:

```bash
user@client: ssh -i ~/.ssh/client.key user@X.X.X.X 
```

On `client.key` és la clau privada del client i `X.X.X.X` és l'adressa IP del servidor.

Un cop fet aquest procés, el client ja estarà en la llista de hosts autoritzats.

#### Auto-aixecament del tunel ssh

Referències:
molt recomanable llegir-les per entendre bé que és i com funciona tot el tema del tunneling ssh i autossh.
* [Access your home linux box from anywhere with SSH tunnels](https://starbeamrainbowlabs.com/blog/article.php?article=posts%2F243-Access-Your-Linux-Box-From-Anywhere-SSH-Tunnels.html)
* [SSH Tunneling For Fun And Profit: autossh](https://www.everythingcli.org/ssh-tunnelling-for-fun-and-profit-autossh/)
* [How does reverse SSH tunneling work?](https://unix.stackexchange.com/questions/46235/how-does-reverse-ssh-tunneling-work/118650#118650)

**Important**

>En aquesta seccio, s'assumeix que el client ssh s'ha generat un parell de claus publica/privada i que la clau publica del client ssh es troba en el fitxer `authorized_hosts` del servidor ssh.

Hi ha dos possibles solucions per tal d'evitar que el tunel ssh es caigui. A continuació es mostren els passos a seguir per configurar les dues alternatives. (*Encara queda pendent acabar de determinar quina de les dos es la mes adecuada pel nostre cas d'us*).

##### Tunel SSH permanent amb el flag -M:

Al **servidor restringit**, escriure el unit file que aixecara el proces d'autossh:

```bash
sudo nano /etc/systemd/system/autossh-tunnel.service
```
S'obrira un editor, afegir el contingut seguent:

```source
[Unit]

Description=AutoSSH tunnel service 
Documentation=https://github.com/aucoop/origineOAdiadem/

# This will ensure that all configured network devices are up and have an IP address assigned before the
# service is started. network-online.target will time out after 90s.
# Enabling this might considerably delay your boot even if the timeout is not reached.
After=network-online.target ssh.service
Wants=network-online.target

[Service]

# autossh is a program to start a copy of ssh and monitor it, restarting it as necessary should it die or stop passing traffic.
# Man page: https://linux.die.net/man/1/autossh
# Flags:
# * AUTOSSH_GATETIME=0 Equivale a el flag -f
# * -M : specifies the base monitoring port to use
# * -N : Just establish the tunnel, no command input (no interactive).
# * -o "ExitOnForwardFailure=yes" : f the client cuts the connection to the server (like power goes off), the port may still be considered in use on the server.
# * -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no": Force key exchange authentication, avoiding password auth.
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 20000 -N -o "ExitOnForwardFailure=yes" -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no" -i /home/konte/.ssh/konte_server.key -R 5555:localhost:22 aucoop@147.83.200.148

# Restart every >2 seconds to avoid StartLimitInterval failure
RestartSec=30
Restart=always
User=konte
[Install]
WantedBy=multi-user.target

```

Cal prestar atenció a aquesta línia:

```source
ExecStart=/usr/bin/autossh -M 20000 -N -o "ExitOnForwardFailure=yes" -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no" -i /home/username/.ssh/restricted_server_private.key -R 5555:localhost:22 aucoop@147.83.200.148
```

* `username` és el nom del usuari de la màquina restringida a la qual es vol tenir accés.
* `restricted_server_private.key` clau privada del servidor al qual es vol tenir accés.

##### Tunel permanent configurant els parametres de la configuracio ssh.

---

**IMPORTANT!**
De moment aquesta configuració s'ha tret de la màquina restringida perquè existeix la hipòtesi de que podria entrar en conflicte amb el paràmetre -M

---

* Al costat del client, es a dir, **al servidor restringit** editar el fitxer `ssh_config`:

  ```source
  sudo nano /etc/ssh/ssh_config
  ```

  Afegir aquestes dos linies

  ```source
  ServerAliveInterval 20
  ServerAliveCountMax 6
  ```

* Al costat servidor, es a dir, **la maquina del despatx d'AUCOOP**, escriure en el fitxer de configuracio del daemon `sshd_config`:

  ```source
  sudo nano /etc/ssh/sshd_config  
  ```

  Afegir aquestes dos linies

  ```source
  ClientAliveInterval 20
  ClientAliveCountMax 6
  ```
  
Per mes informacio sobre que fan aquests parametres, aqui hi han les referencies en les que ens hem basat: [ref1](https://patrickmn.com/aside/how-to-keep-alive-ssh-sessions/), [ref2](https://unix.stackexchange.com/questions/3026/what-options-serveraliveinterval-and-clientaliveinterval-in-sshd-config-exac/3027), [ref3](https://superuser.com/questions/744606/ssh-timeout-clientaliveinterval-clientalivecountmax-vs-serveraliveinterval), [ref4](https://serverfault.com/questions/626461/autossh-does-not-kill-ssh-when-link-down).

Un cop modificats aquests parametres, afegir el seguent contingut al unit file en el **servidor restringit**.

```source
sudo nano /etc/systemd/system/autossh-tunnel-primary.service
```
**Unit file Dunia Kato**
```source
[Unit]

Description=AutoSSH tunnel service 
Documentation=https://github.com/aucoop/origineOAdiadem/

# This will ensure that all configured network devices are up and have an IP address assigned before the
# service is started. network-online.target will time out after 90s.
# Enabling this might considerably delay your boot even if the timeout is not reached.
After=network-online.target ssh.service
Wants=network-online.target

[Service]

# autossh is a program to start a copy of ssh and monitor it, restarting it as necessary should it die or stop passing traffic.
# Man page: https://linux.die.net/man/1/autossh
# Flags:
# * AUTOSSH_GATETIME=0 Equivale a el flag -f
# * -M : specifies the base monitoring port to use
# * -N : Just establish the tunnel, no command input (no interactive).
# * -o "ExitOnForwardFailure=yes" : f the client cuts the connection to the server (like power goes off), the port may still be considered in use on the server.
# * -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no": Force key exchange authentication, avoiding password auth.
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 30000 -N -o "ExitOnForwardFailure=yes" -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no" -i /home/dk/.ssh/server_duniakato.key -R 9999:localhost:22 aucoop@147.83.200.148

# Restart every >2 seconds to avoid StartLimitInterval failure
RestartSec=30
Restart=always
User=dk
[Install]
WantedBy=multi-user.target
```

**Unit file Fatim Kone**
```source
[Unit]

Description=AutoSSH tunnel service 
Documentation=https://github.com/aucoop/origineOAdiadem/

# This will ensure that all configured network devices are up and have an IP address assigned before the
# service is started. network-online.target will time out after 90s.
# Enabling this might considerably delay your boot even if the timeout is not reached.
After=network-online.target ssh.service
Wants=network-online.target

[Service]

# autossh is a program to start a copy of ssh and monitor it, restarting it as necessary should it die or stop passing traffic.
# Man page: https://linux.die.net/man/1/autossh
# Flags:
# * AUTOSSH_GATETIME=0 Equivale a el flag -f
# * -M : specifies the base monitoring port to use
# * -N : Just establish the tunnel, no command input (no interactive).
# * -o "ExitOnForwardFailure=yes" : f the client cuts the connection to the server (like power goes off), the port may still be considered in use on the server.
# * -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no": Force key exchange authentication, avoiding password auth.
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 -N -o "ExitOnForwardFailure=yes" -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no" -i /home/konte/.ssh/konte_server.key -R 6666:localhost:22 aucoop@147.83.200.187

# Restart every >2 seconds to avoid StartLimitInterval failure
RestartSec=30
Restart=always
User=konte
[Install]
WantedBy=multi-user.target
```

Notar que ara el flag `-M` esta a 0, ja que el monitoratge del tunel passa a carrec del daemon de ssh.

Cal prestar atenció a aquesta línia:

```source
ExecStart=/usr/bin/autossh -M 0 -N -o "ExitOnForwardFailure=yes" -o "PubkeyAuthentication=yes" -o "PasswordAuthentication=no" -i /home/username/.ssh/restricted_server_private.key -R 6666:localhost:22 aucoop@147.83.200.187
```

* `username` és el nom del usuari de la màquina restringida a la qual es vol tenir accés.
* `restricted_server_private.key` clau privada del servidor al qual es vol tenir accés.


## Guia per accedir a tots els ordinadors del centre desde l'exterior

En aquesta seccio es mostren els passos a seguir per poder accedir als ordinadors del centre desde qualsevol maquina a qualsevol lloc del mon. De moment, l'acces nomes es pot fer via ssh, es a dir, nomes acces al terminal.

**Important**
>En aquesta seccio, s'assumeix que hi ha un acces permanent al servidor del centre desde l'exterior via tunel invers ssh desde el servidor de la upc.

A continuacio es llisten els passos a seguir:

Accedir al servidor de la upc:

```source
ssh aucoop@147.83.200.187
```

Accedir al servidor del centre via tunel invers, en el cas de Saint Louis (Senegal):

```source
ssh -p 5555 konte@localhost
```

Un cop dins del servidor del centre, per accedir als ordinadors de la intranet es pot accedir a traves del nom de la maquina: `ssh user@nom_de_la_maquina.local`.

En el cas dels ordinadors amb labdoo es molt facil, perque les maquines tenen la id de labdoo implicita en el seu nom i tots tenen el mateix format. Per tant:

```source
ssh labdoo@labdoo-ID_DE_LABDOO.local
```
Un exemple seria:

```source
ssh labdoo@labdoo-000019931.local
```

A vegades aquest metode dona error. Si es aixi, es pot fer us del programa `nmap` per veure els equips que es troben a la xarxa, aixi com els seus noms i adresses IP actuals. En el cas del centre de Saint Louis, la subxarxa on es troben tots els ordinadors es del rang `192.168.33.0/24`. Per tant l'ordre amb nmap seria:

```source
nmap -sn 192.168.33.0/24
```

TODO
```source
OUTPUT DEL COMANDO
```

Ara simplement cal accedir per ssh a la maquina a traves de  `ssh labdoo@ip`.
