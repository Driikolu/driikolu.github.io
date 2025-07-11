# Installation D'Archlinux en UEFI & Chiffré



# Introduction

Cet article peut servir de support pour installer ArchLinux avec des partitions root ( / ) et home ( /home ) chiffrées et le tout en UEFI.

## Pourquoi ArchLinux

Il y a 3 grandes raisons qui me poussent à utiliser ArchLinux depuis maintenant plusieurs années.

J'apprécie dans un premier temps la philosophie de cette distribution : 

- Elle est développée pour être légère et "simple" (sans superflu).
- Elle se base sur l'idée que l'utilisateur comprend ce qu'il fait, elle laisse donc une totale liberté dans sa configuration.
- Elle est basée sur le libre & communautaire, tout le monde peut donc participer à l'amélioration de la distribution et des paquets

Ensuite, en lien avec le dernier point, il existe sur ArchLinux le AUR (ArchLinux User Repository) : Des depots utilisateurs moins vérifiés mais permettant à n'importe qui de créer un paquet. On trouve facilement la majorité des outils disponible sous linux via l'AUR, ce qui est un plus dans la sécurité.

Pour finir, le wiki ArchLinux est l'un des plus complets (si ce n'est le plus complet). On peut facilement s'y référer en cas de problèmes et pour en apprendre un peu plus sur nos outils ou sur le fonctionnement de Linux.

## Pourquoi chiffrer les partitions ?

On peut considérer 2 contextes :

- Le contexte professionnel
- Le contexte privé

Dans le contexte professionnel, on est souvent ammené à chiffrer nos données. En général ce travail est fait automatiquement par des services informatiques et des outils. On doit parfois configurer et utiliser un PC sous Linux, notamment pour les pentesteurs, et dans ce cas il faut chiffrer nous même nos données.

Chiffrer les partitions permet à ce moment de ne pas avoir à gérer les fichiers à l'unité tout en apportant le minimum en terme de protection des données.



Dans le contexte privé, tout le monde a son propre rapport à la vie privée. Lorsque l'on veut la protéger au maximum il est normal de vouloir chiffrer ses données. En comparaison au chiffrement fichier par fichier, le chiffrement des partitions permet de ne pas limiter son confort d'utilisation en déchiffrant celles-ci uniquement au démarrage du PC.

# L'installation

Vous pourrez voir quelques captures d'écrans faites durant une installation. Pour l'exemple j'ai utilisé une machine virtuelle de 30Go, j'adapterais donc les valeurs des espaces disques pour coller mieux à des disque durs plus grands.

## Préparation

Dans un premier temps, vous devez [télécharger un liveboot](https://www.archlinux.org/download/) et le mettre sur le support de votre choix (USB ou CD).

Avant de vous lancer dans l'installation, vérifiez bien que vous avez votre BIOS en UEFI et non en Legacy.

Je vais aussi considérer que vous êtes en filaire. Si vous voulez vous connecter en wifi, vous pouvez utiliser **wifi-menu**.

Une fois le liveboot ArchLinux lancé, on va passer notre clavier en AZERTY. Si vous avez une autre disposition de clavier  je vous invite à [regarder sur le wiki spécifique](https://www.archlinux.org/download/).

```bash
loadkeys fr
```

On va ensuite régler notre horloge. Ce n'est pas toujours nécessaire mais cela peut parfois poser problème pour la vérification de paquets. Ici, je considère que vous utilisez l'heure française.

```bash
timedatectl set-timezone Europe/Paris
timedatectl set-ntp true
```



## Le partionnement

Dans un tout premier temps, nous allons lister nos disques

```bash
fdisk -l
```



![Retour fdisk -l](/img/fdisk_install.png)

Ici, on peut voir que le disque que nous voulons utiliser est **/dev/sda**. Le disque /dev/loop0 correspondant à mon liveboot Arch. 

Pour le partitionnement j'ai décider de faire 4 partitions :

- Une partition EFI (pour l'UEFI)
- Une partition boot (pour le bootloader)
- Une partition root ( / )
- Une partition home ( /home )

Dans le cadre de partitions chiffrées, il faut obligatoirement au moins une partition EFI en dehors. Pour des raisons de confort et de facilité j'ai décidé de garder un /boot non chiffré aussi (Il est selon moi, moins important de le chiffrer).

Les partitions root et home seront chiffrées. Vous pouvez cependant ne faire que 3 partitions en ne faisant pas de partition home.

Pour commencer le partitionnement, on ouvre notre disque avec *gparted*.

```bash
parted /dev/sda
```

Respectez les valeurs pour les 2 premières partitions qui feront respectivement 100Mib et 250Mib.

Pour la partition *root* je vous conseille au moins 20GiB, mais au vu de la taille des disques durs actuel vous pouvez facilement pousser à 50GiB voir plus.

Le reste ira dans le *home* qui acceuille généralement les gros fichiers & les jeux.

```parted
mklabel gpt
mkpart primary fat32 1MiB 101MiB  #La partition EFI
set 1 esp on
mkpart primary ext4 101MiB 361MiB #La parition boot
set 2 boot on
mkpart primary ext4 361MiB 20.5GiB #La partition root d'environ 20GiB
mkpart primary ext4 20.5GiB 100% #La partition home prenant le reste de place
q
```

Une fois nos partitions créées, nous allons chiffrer nos 2 partitions *root* et *home*. J'ai choisi d'utiliser de l'AES avec une clef de 512 bits. Nous allons utiliser un temps d'itération de 5000 millisecondes ce qui permet un peu plus de sécurité que la configuration par défaut (2000) sans prendre trop de temps.

On nous demandera ensuite pour chaque chiffrement une **passphrase** c'est à dire un mot de passe assez long. Une bonne clef de chiffrement, dans notre cas, sera une clef longue (et non une clef avec des caractères spéciaux).

Ici, nous avons 2 disques donc 2 **passphrases** différentes.

```bash
cryptsetup -v --type luks -c aes-xts-plain64 -s 512 -h sha512 -i 5000 -y luksFormat /dev/sda3
cryptsetup -v --type luks -c aes-xts-plain64 -s 512 -h sha512 -i 5000 -y luksFormat /dev/sda4
```

Une fois les disques chiffrés, nous allons les déchiffrer en leur donnant un nom qui nous servira ensuite pour faire appel à eux. En tant que fan de [bitoduc](https://bitoduc.fr/), j'ai décidé d'appeler mon disque root *racine* et mon disque home *maison*.

```bash
cryptsetup open /dev/sda3 racine
cryptsetup open /dev/sda4 maison
```

On créé ensuite les systèmes des fichiers sur nos partitions. Il faut que la partition EFI soit en fat32, pour le reste j'utilise le format ext4.

Pour faire appel à nos disques déchiffrés, nous devons utiliser **/dev/mapper/\<NOM DU DISQUE>**

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkfs.ext4 /dev/mapper/racine
mkfs.ext4 /dev/mapper/maison
```

Nos partitions sont enfin prêtes, on va commencer à vraiment installer Linux.

## Préparation de Linux

Pour commencer, il faut monter nos partitions afin qu'elles forment dans **/mnt** notre système de fichier pour Linux. Pour les partitions home, boot et EFI, il faudra créer nous-même les points de montage à l'intérieur du root.

```bash
mount /dev/mapper/racine /mnt
mkdir /mnt/home && mount /dev/mapper/maison /mnt/home
mkdir /mnt/boot && mount /dev/sda2 /mnt/boot
mkdir /mnt/boot/efi && mount /dev/sda1 /mnt/boot/efi
```

Une fois notre système de fichier en place, on y installe les paquets de base ainsi que linux et des firmwares. (Depuis Octobre 2019, le paquet *linux* n'est plus compris dans l'ensemble de paquet *base*)

Pour plus de vitesse à l'installation vous pouvez ordonner les miroirs de **/etc/pacman.d/mirrorlist** afin de mettre ceux de votre pays et des pays voisins en premier.

Il y a beaucoup de paquets à installer, ça peut prendre quelques minutes.

```bash
pacstrap /mnt base base-devel linux linux-firmware
```

Une fois tous les paquets installés, on va générer le fstab, c'est à dire le fichier qui indique les points de montage des partitions.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Notre ArchLinux est installé, mais pas fonctionnel. On va donc le configurer !



## Configuration de Linux

Dans un premier temps on va *chrooter*, c'est à dire utiliser nos partitions montées comme la racine de notre système.

```bash
arch-chroot /mnt
```

On va ensuite définir un nom pour notre machine. Remplacez dans les commandes suivantes **NomMachine** par celui voulu en évitant les caractères spéciaux et les espaces 

```bash
echo NomMachine > /etc/hostname
echo '127.0.0.1 NomMachine.localdomain NomMachine' >> /etc/hosts
```

On va configurer l'horloge de notre nouveau système, là encore pour éviter les problèmes de NTP et de vérifications de paquets. Que ça soit à l'installation ou lors de l'utilisation futur de notre Linux.

```bash
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

Nous allons ensuite générer nos locales. Elles vont définir la langue du système (et non pas le clavier).

J'aime avoir mon système en français, même si c'est parfois plus compliqué pour les erreurs, je génère donc ma locale française.

```bash
sed -i 's/#fr_FR.UTF-8 UTF-8/fr_FR.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo LANG="fr_FR.UTF-8" > /etc/locale.conf
export LANG=fr_FR.UTF-8
```

On configure ensuite notre clavier en AZERTY.

```bash
echo KEYMAP=fr > /etc/vconsole.conf
```

Puis on met un mot de passe au compte root.

```bash
passwd
```

On va ensuite passer à la partie la plus "délicate", qui est généralement celle qui rate lors des installations : Le bootloader !

## Gestion boot & chiffrement

Pour commencer il faut installer un bootloader, dans mon cas j'utilise **grub**. On installe aussi **efibootmgr** afin de gérer l'UEFI.

Pour ma part j'en ai profiter pour installer *vim* afin de configurer les fichiers par la suite, je vous invite à installer l'éditeur de texte de votre choix.

```bash
pacman -S grub efibootmgr <EDITEUR DE TEXTE>
```

Pour gérer le déchiffrement de notre partie *racine*, il nous faut l'UUID de sa partition chiffrée (/dev/sda3). On va donc le récupérer avec la commande **lsblk -f /dev/sda3**.

Il faut bien lire la ligne correspondant à notre partition chiffrée (/dev/sda3) et non déchiffrée (/dev/mapper/racine).

On va ensuite utiliser cet UUID pour modifier la ligne **GRUB_CMDLINE_LINUX** du fichier **/etc/default/grub** afin quelle soit sous la forme

__GRUB_CMDLINE_LINUX="cryptdevice=UUID=*\<VOTRE UUID>*:racine root=/dev/mapper/racine"__

Il faut aussi dé-commenter la ligne **GRUB_ENABLE_CRYPTODISK=y**

 Astuce : vous pouvez rediriger la sortie de lsblk avec *>>* vers votre fichier afin de n'avoir qu'à copier l'UUID via votre éditeur. 

```
lsblk -f /dev/sda3 >> /etc/default/grub
```

![contenu /etc/default/grub](/img/grub_install.png)

Ensuite, on va modifier le fichier **/etc/mkinitcpio.conf** afin d'ajouter les hooks *keymap* et *encrypt* (dans cet ordre). Ils nous permettront d'avoir la bonne configuration de clavier et de déchiffer nos disques au démarrage de la machine.

![Contenu /etc/mkinitcpio.conf](/img/mkinitcpio_install.png)

On modifie le fichier **/etc/fstab** afin de ne pas utiliser l'UUID de */dev/mapper/maison* pour le monter. On commente donc la ligne correpondante et on rajoute une dernière ligne contenant

__/dev/mapper/maison /home ext4 rw,relatime,data=ordered 0 2__

![Contenu /etc/fstab](/img/fstab_install.png)

Contenu /etc/crypttabPuis on modifier le fichier **/etc/crypttab** pour déchiffrer notre partition *maison* au démarrage.

Pour cela il suffit de rajouter le nom de la partition et sa parittion chiffrée correspondante.

__maison /dev/sda4__

![Contenu /etc/crypttab](/img/crypttab_install.png)

Pour finir, on va générer ce qui nous permettra de charger les modules kernels au démarrage et on va finaliser l'installation de grub.

```bash
mkinitcpio -p linux
grub-install --target=x86_64-efi --efi-directory=/boot/efi --recheck /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```



Vous pouvez en théorie démarrer et vous aurez un ArchLinux en UEFI avec des partitions chiffrées !



# Aller plus loin



Par défaut, vous n'aurez quasiment aucun paquet sur votre ArchLinux, même pour la connexion réseau. Vous pouvez très bien relancer le liveboot et répeter les phases de déchiffrement & montage des partitions afin d'installer des nouveaux paquets.

Je vous conseille aussi, pour profiter un maximum de la distribution, d'utiliser un *AUR helper* qui permettra d'installer faciement les paquets utilisateur.

Pour cela, regardez dans [la liste fournie sur le wiki](https://wiki.archlinux.org/index.php/AUR_helpers), vous trouverez probablement votre bonheur. Pour ma part à l'heure où j'écris cet article j'utilise *pikaur*.

Attention cepandant, les *AUR helpers* sont développés et maintenus par la communauté. Il se peut au fil du temps que le projet soit abandonné et donc plus mis à jour. Certains grands projets comme *aurman* et *yaourt* on ainsi été dépréciés.



---

> Auteur: [Driikolu](https://x.com/driikolu)  
> URL: https://driikolu.fr/posts/install_arch_uefi/  

