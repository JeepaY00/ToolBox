# Accéder partage NFS sous Windows Server 2012 R2 | Windows Server | IT-Connect

URL: https://www.it-connect.fr/acceder-partage-nfs-sous-windows-server-2012-r2/?utm_source=ReviveOldPost&utm_medium=social&utm_campaign=ReviveOldPost

![https://www.it-connect.fr/wp-content-itc/uploads/2014/04/tuto-monter-partage-nfs-windows.jpg](https://www.it-connect.fr/wp-content-itc/uploads/2014/04/tuto-monter-partage-nfs-windows.jpg)

---

![tuto-monter-partage-nfs-windows.jpg](tuto-monter-partage-nfs-windows.jpg)

Sommaire [[-](https://www.it-connect.fr/acceder-partage-nfs-sous-windows-server-2012-r2/#)]

## I. Présentation

**Dans ce tutoriel, nous allons voir comment monter un partage NFS sur Windows en installant un client NFS via PowerShell ou l'interface graphique.**

Nativement, il n'est pas possible d'accéder à un partage NFS depuis Windows, que ce soit les éditions Desktop (Windows 10 ou Windows 11), ou les éditions Server comme comme Windows Server 2019. Ce n'est pas surprenant, car ce type de partage réseau étant créé dans la plupart des cas sur un hôte [Linux](https://www.it-connect.fr/cours-tutoriels/administration-systemes/linux/).

Heureusement, Microsoft a tout prévu pour permettre l'[accès à un partage NFS depuis Windows Server](https://www.it-connect.fr/creation-dun-espace-de-stockage-reseau-nfs-avec-freenas-et-ajout-dans-vmware-esxesxi/). Il suffit de passer par l'installation de fonctionnalités facultatives.

Pour cela, j'utilise un serveur **Windows Server 2019** en tant que client NFS, concernant le serveur **NFS** il s'agit d'un serveur **Debian** avec le partage **/srv/partagenfs**.

Si vous souhaitez découvrir le protocole NFS plus en détails, voici mon tutoriel d'introduction :

- [Le protocole NFS pour les débutants](https://www.it-connect.fr/le-protocole-nfs-pour-les-debutants/)

## II. Installation du client NFS sous Windows

Deux méthodes d'installation seront détaillées : méthode graphique / méthode PowerShell (beaucoup plus rapide).

### A. Installer le client NFS avec l'interface graphique

Ouvrez le Gestionnaire de serveur, cliquez sur "**Gérer**" et "**Ajouter des rôles et fonctionnalités**". Lorsque l'étape "**Avant de commencer**" apparaît, cliquez sur "**Suivant**".

Sélectionnez "**Installation basée sur un rôle ou une fonctionnalité**" et poursuivez.

Ensuite, sélectionnez le serveur sur lequel vous souhaitez installer la fonctionnalité, continuez. Concernant l'étape "**Rôles de serveurs**" vous pouvez la passer, car dans ce cas c'est l'installation de fonctionnalités qui nous intéressent.

Désormais, nous arrivons à l'étape la plus importante de l'installation : le choix des éléments à installer. Vous devez cocher deux fonctionnalités :

- **Client pour NFS**
- Outils d'administration de serveur distant > Outils d'administration de rôles > Outils de [services](https://www.it-connect.fr/cours-tutoriels/administration-systemes/linux/services/) de fichiers > **Services des outils de gestion du [système](https://www.it-connect.fr/cours-tutoriels/administration-systemes/windows-server/systeme/) de gestion de fichiers en réseau** (ceci étant un outil RSAT).

Une fois les deux choix effectués, cliquez sur "**Suivant**".

Enfin, cliquez sur "**Installer**" et patientez un instant.

### B. Installer le client NFS avec PowerShell

Ouvrez une console PowerShell, et, saisissez la commande suivante sur votre serveur Windows :

```
Install-WindowsFeature NFS-Client,RSAT-NFS-Admin
```

Patientez pendant l'installation, vraiment très simple et efficace en PowerShell une fois le nom des fonctionnalités repéré.

Si vous souhaitez installer le client NFS sur une édition Desktop de Windows, comme Windows 10 ou Windows 11, la commande est différente :

```
Enable-WindowsOptionalFeature -FeatureName ServicesForNFS-ClientOnly, ClientForNFS-Infrastructure -Online
```

## III. Monter un partage NFS sur Windows

L'installation étant effectuée, on peut désormais **monter le partage NFS sur le serveur Windows**. Ouvrez une **Invite de commande**, et, utilisez la commande *mount* comme ceci pour un montage anonyme :

```
mount -o anon <serveur>:<partage> <lettre-montage>:
```

N'utilisez pas une console PowerShell pour monter le partage NFS avec la syntaxe ci-dessus, car cela ne fonctionnera pas : il y a un alias entre "*mount*" et "*New-PSDrive*".

```
mount -o anon 192.168.100.121:/srv/partagenfs N:
```

Partage NFS monté sur Windows avec mount

Si vous utilisez une console PowerShell, vous pouvez utiliser ceci :

```
New-PSDrive -Name N -Root "\\192.168.100.121\srv\partagenfs" -PSProvider FileSystem
```

Néanmoins, je trouve que les résultats sont assez aléatoires pour monter un partage NFS avec New-PSDrive. Préférez plutôt la commande "mount" dans l'Invite de commande.

Le partage NFS se retrouve bien sous la lettre N sur le serveur :

Nous pourrions également monter ce partage NFS à partir de l'Explorateur de fichiers Windows, en précisant le chemin vers le partage au format UNC. Cela donnerait : *\\192.168.100.121\srv\partagenfs*

## IV. Registre Windows : AnonymousGid et AnonymousUid

Lorsque le partage NFS est monté comme nous venons de le faire, il y a de fortes chances pour qu'il soit monté en lecture seule. Si vous avez besoin d'être en mode anonyme tout en ayant les droits de lecture et d'écriture, il va falloir modifier la base de Registre de Windows.

Ouvrez l'éditeur de Registre en tant qu'administrateur :

```
regedit.exe
```

Parcourez l'arborescence comme ceci :

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ClientForNFS\CurrentVersion\Default
```

Ici, il va falloir créer deux valeurs "**DWORD (32 bits)**" nommées : **AnonymousGid** (ID du groupe) et **AnonymousUid** (ID de l'utilisateur).

Pour ces valeurs, la **valeur décimale** doit correspondre à l'ID déclaré dans la configuration du partage NFS, sur le serveur distant. Pour ma part, mon partage s'appuie sur l'utilisateur *nobody* et le groupe *nogroup*. Ces deux éléments ont le même ID : 65534.

Sur mon serveur Linux, dans le fichier "/etc/exports", j'ai la configuration suivante :

```
/srv/partagenfs 192.168.100.0/24(rw,sync,anonuid=65534,anongid=65534,no_subtree_check)
```

Ce qui me donne dans le Registre :

Registre Windows : les valeurs AnonymousGid et AnonymousUid

Suite à cette modification, il faut redémarrer le serveur. Ensuite, vous devriez pouvoir accéder au partage NFS en lecture et écriture !

## V. Paramétrage du client NFS

En cas de nécessité, sachez qu'il est possible de paramétrer le client NFS que nous avons installé. Pour cela, accédez aux Outils d'administration et ouvrez "**Services pour NFS**". Lorsque la console est ouverte, effectuez un clic droit sur "**Client pour NFS**" et cliquez sur "**Propriétés**".

Divers paramètres sont accessibles, comme le(s) protocole(s) de transport à utiliser par le client NFS (UDP, TCP ou les deux) ou encore les autorisations UNIX à attribuer par défaut aux fichiers créés sur le partage.

Enfin, l'onglet "**Sécurité**" sert à spécifier les stratégies de sécurité autorisées. Tout ce qui concerne Kerberos, à savoir "*krb5*", "*krb5i*" et "*krb5p*" nécessite d'utiliser NFS v4 au minimum.