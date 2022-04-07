# BreizhCTF 2022 - Extranet Word Worker


# Introduction

Le 1er Avril 2022 a eu lieu de BreizhCTF, et ce, 3 ans après la dernière édition.
Bien sûr, j'ai voulu y participer. Et comme d'habitude il y a eu plein de challenges intéréssant.

Mais je n'ai pas spécialement envie de parler d'une énième XSS qui mènera à une RCE ou de comment retrouver un domaine avec des requêtes modifiées... Ou tout autre truc du genre.
Aujourd'hui, on va parler de programmation !
Et je vais essayer de bien tout détailler

# Le challenge

Il y avait en tout 3 challenges de programmation, je n'ai touché qu'à celui dont je vais vous parler : Word Worker.

Le but est simple : se connecter à un serveur et résoudre les petits challenges qu'il nous propose. En boucle.

Pas de problème, je suis coutumier de ce genre de puzzle, je dégaine mon python et c'est partie pour un petit moment de code.


## Se connecter au serveur

On oublie souvent d'en parler de cette étape, probablement parce qu'elle n'est pas très intéréssante, mais j'ai tout de même un petit point à apporter.

Dans ce genre de Challenge, je vois beaucoup de gens habitués aux CTF en tout genre utiliser [_pwntools_](http://docs.pwntools.com/en/stable/). L'outil est très puissant et embarque directement une gestion des socket. Mais en soit il n'est pas vraiment adapté à ce genre de challenge... C'est l'équivalent d'utiliser un bazooka pour tuer une mouche : Ça fonctionne, mais était-ce vraiment nécessaire, et est-ce qu'on se serait pas un peu compliqué la vie ?

Pour ma part, j'opte pour une solution simple : _socket_, que je couple avec _time_ pour ne pas répondre trop vite.

```python
import socket
import time

s = socket.socket() #On créé notre socket
s.connect(('challenges.ctf.bzh', 27001)) #On se connecte au serveur

time.sleep(1) #On laisse le temps de se connecter au serveur

while True:
    time.sleep(0.1) #Pour laisser le temps de recevoir le message
    output = s.recv(2048).decode() #On récupère le message et on le passe de Byte-string à string
    print(output) #Pour avoir la visibilité sur ce qu'il se passe

    res="Ma réponse"
    s.send(res.encode()+b"\n") #On envoie notre réponse (en Byte-String et on oublie le \n pour qu'elle soit envoyée)
```

Parfait, tout est en place, on va pouvoir commencer vraiment le challenge.

## Trouver l'anagramme

Dès notre connexion, on se retrouve avec un message plutôt simple :
> Which word is mixed up ? XXXXXXX

On retrouve alors un ensemble de lettre avec seulement 1 lettre majuscule. Avec la phrase en plus, on comprend qu'il s'agit d'un anagramme dont il faut retrouver le mot d'origine. La lettre majuscule étant la première lettre du mot.

### Itertools ?

Une première idée était d'utiliser itertools.
Mais, au moment de sa sortie, ce challenge proposait des mots de toute taille, parfois de plus de 10 lettres.

Par exemple, s'il fallait trouvé __Zygomatiques__ depuis un de ses anagrammes, on aurait un total de 11! (soit 39.916.800) possibilités.
Il y a de forte probabilités pour que le mot soit trouvé avant toutes les itérations. Mais même en trouvant au bout de 1.000.000 itérations, le temps passer pour 1 seul mot est plutôt conséquent.

Après une petite maintenance le challenge est revenu en ne proposant que des mots de 5/6 lettres, mais ma solution était déjà écrite.

### Un dictionnaire !



Heureusement, ce sont les anagramme de mots français.
Et par chance, il existe un dictionnaire plutôt complet pour les mots en français : [ods6.txt, le dictionnaire officiel du scrabble](http://www.3zsoftware.com/listes/ods6.zip).

Et on s'empresse de coder une fonction retrouvant l'anagramme.

Tout d'abord, il faut avoir la liste de tous les mots présents dans notre dictionnaire.
```python
dico_words = open("ods6.txt","r").read().split()
```

Maintenant, comment peut-on retrouver notre mot ?

Tout d'abord il faut repérer les critères qu'on peut utiliser :
- La première lettre du mot
- L'ensemble des lettres

Ça fait peut, et le tout doit être formatté comme il faut. Notre dictionnaire a tous ses mots en majuscule, on fait de même avec notre anagramme.

```python
#On récupère la seule lettre en majuscule, première lettre du mot
first_letter = [l for l in anagramme if l.isupper()][0]
#On met l'anagramme en majuscule
anagramme = anagramme.upper()
```

Pour vérifier notre premier critère, c'est plutôt simple : On compare les premières lettres.

Mais comment vérifier que le contenu total soit bien le même ?
On va juste vérifier qu'il y ai exactement les même lettres, et ce, en triant nos mots avec __sorted()__.

```python
possibilities = [w for w in dico_words if w[0]==first_letter and sorted(w)==sorted(anagramme)]
```

Ici on a un dictionnaire avec toutes les possibilités. On va juste considérer qu'il faut sortir la première. Donc on va juste renvoyer le premier mot possible.

```python
word = possibilities[0] #On récupère le premier mot
return word[0] + word[1:].lower() #On le formatte pour le serveur
```

Malheureusement, certains mots ont exactement les même lettres. Par exemple Signe et Singe.
Le challenge a prévu ces éventualité et n'est pas trop punitif. En cas d'erreur, on peut faire d'autre propositions (c'est d'ailleurs pour ça qu'une solution avec itertools est possible).

On va donc rajouter la possibilité de récupérer la possibilité suivante dans notre fonction.
Ce qui une fois entière donne le code qui suit

```python
def get_word(anagramme, ind):
    first_letter = [l for l in anagramme if l.isupper()][0]
    anagramme = anagramme.upper()
    possibilities = [w for w in dico_words if w[0]==first_letter and sorted(w)==sorted(anagramme)]
    word = possibilities[ind]
    return word[0] + word[1:].lower()
```

### On l'ajoute à notre boucle

Bon, on peut trouver un mot à partir de son anagramme, maintenant il faut le faire à partir de ceux qu'on nous envoie.
On va tout simplement faire du parsing sur les messages du serveur dans notre boucle.

```python
while True:
    time.sleep(0.1) 
    output = s.recv(2048).decode() 
    print(output)

    if "mixed up" in output: #Si c'est un nouvel anagramme
        try_nb = 0 #Premier essaie sur cet anagramme
        #On récupère le mot un peu salement
        anagramme = output.split("mixed up ? ")[1].split()[0]
        res = get_word(word,try_nb)

    else: #Si on n'a pas fourni le bon mot précédemment
        try_nb += 1 #On incrémente notre index

    print(res) #Pour avoir une vision sur ce que j'envoie
        
    s.send(res.encode()+b"\n")
```

Super, tout passe parfaitement.

## De nouveaux challenges

Malheureusement, ça n'était pas suffisant.
En effet, après quelques dizaines de mots retrouvés, de nouveaux exercices fleurissaient petit à petit.

Cela dit, il était bien plus simple de les résoudre, donc on va faire ça en vitesse. Et pour être un peu plus propre que pendant le CTF, je ferai une fonction pour chaque exercice.

### A l'envers

On nous demande de mettre à l'envers un mot.

> Put it backwards : 'XXXXXXX'

Rien de plus simple :

```python
def backward(mot):
    return mot[::-1]
```


### En majuscule !

Au final, il y a plus simple que de mettre à l'envers. Il faut renvoyer le mot tout en majuscule.

> Oops, 'XXXXXX' must be given in UPPERCASE

Une méthode en python le fait tout seul.

```python
def majuscule(mot):
    return mot.upper()
```

### Les consonnes

Un peu plus intéréssant, il faut renvoyer le nombre de consonne dans le mot donné.

> How many consonants in the word 'XXXXXX' ?

Mais le tout se fait encore en une seule ligne.
J'ai tout de même considéré qu'il n'y aurais pas de mots composés (présents dans la 1ère version du challenge).

```python
def consonnes(mot):
    return str(len([l for l in mot if l not in 'aeiouy']))
```

### ROT15

Ici, quelque chose qui peut être intéréssant. 

> What is the ROT15 of the word 'XXXXXXX' ?

Mais honnêtement, l'objectif durant le challenge était la rapidité. Je ne me suis pas embêté et j'ai juste cherché sur StackOverflow [un code de rot13](https://stackoverflow.com/questions/3269686/short-rot13-function-python) !

Et je l'ai un peu modifié pour qu'il soit en rot15.

```python
def rot15(s):
    chars = "abcdefghijklmnopqrstuvwxyz"
    trans = chars[15:]+chars[:15] #La ligne à modifier
    rot_char = lambda c: trans[chars.find(c)] if chars.find(c)>-1 else c
    return ''.join( rot_char(c) for c in s )
```

Je suis enfin un vrai développeur, je copie du code sur StackOverflow !

### Une bière ?

Pour finir, on nous demande si on paierais une bière au créateur du challenge.

> Would you offer me a beer ?

Étant extrêmement radin, mon premier reflexe aurait été d'envoyer un grand __NON__. Mais j'imagine qu'il faut être gentil tout plein pour ce challenge et je vais envoyer un petit _oui_.

```python
def beer(): #Clairement la fonction la plus utile, j'aurais pas pu faire sans.
    return "Yes" 
```


## Et le flag ?

Petit piège ici, le flag s'affiche mais le challenge continue. J'ai donc faire un test dans ma boucle While pour tout couper en urgence.

```python
if "BZHCTF{" in output:
    break
```

Et il ne me reste plus qu'à le lire.


> BZHCTF{D4mn\_Th4t\_sCR1pt\_w0rked\_1_Gu3sS}

# Le code complet

Je ne suis pas un monstre, je vous propose mon code complet.

```python
import socket
import time

#Le dictionnaire
dico_words = open("ods6.txt","r").read().split()

#Les fonctions
def get_word(anagramme, ind):
    first_letter = [l for l in anagramme if l.isupper()][0]
    anagramme = anagramme.upper()
    possibilities = [w for w in dico_words if w[0]==first_letter and sorted(w)==sorted(anagramme)]
    word = possibilities[ind]
    return word[0] + word[1:].lower()

def backward(mot):
    return mot[::-1]

def majuscule(mot):
    return mot.upper()

def consonnes(mot):
    return str(len([l for l in mot if l not in 'aeiouy']))

def rot15(s):
    chars = "abcdefghijklmnopqrstuvwxyz"
    trans = chars[15:]+chars[:15] #La ligne à modifier
    rot_char = lambda c: trans[chars.find(c)] if chars.find(c)>-1 else c
    return ''.join( rot_char(c) for c in s )

def beer(): #OMG HARDCORE CODE
    return "Yes"


s = socket.socket()
s.connect(('challenges.ctf.bzh', 27001))

time.sleep(1)

while True:
    time.sleep(0.1)
    output = s.recv(2048).decode()
    print(output)

    if "BZHCTF{" in output:
        break

    elif "beer?" in output:
        res = beer()

    elif "ROT15" in output:
        word = output.split("of the word '")[1].split("'")[0]
        res = rot15(word)

    elif "consonants" in output:
        word = output.split("in the word '")[1].split("'")[0]
        res = consonnes(word)

    elif "UPPERCASE" in output:
        word = output.split("Oops, '")[1].split("'")[0]
        res = majuscule(word)

    elif "backwards" in output:
        word = output.split("backwards : '")[1].split()[0][:-1]
        res = backward(word)

    elif "mixed up" in output:
        try_nb = 0
        anagramme = output.split("mixed up ? ")[1].split()[0]
        res = get_word(word,try_nb)

    else:
        try_nb += 1

    print(res)
    s.send(res.encode()+b"\n")

print("FLAGGED")
```

# Mon avis 

Au final, je n'ai pas trouvé le challenge vraiment compliqué.
J'ai parlé de temps en temps dans ce WU de la première version du challenge. Elle a avait ce petit truc en plus qui la rendait plus complexe sur la première partie : Avoir le bon dictionnaire.

Sur la table d'à côté il y avait [Zeecka](https://twitter.com/Zeecka_) qui s'amusait à faire le challenge en même temps que moi. Nous n'avions pas spécialement le même dictionnaire et nous bloquions sur des mots différents tous les deux.

De mot côté il manquait les noms propres (Italie, Lucifer, Hollywood, ...) et les mots composés, puisqu'ils sont interdits au Scrabble. C'est vraiment ce qui m'a empêché de le flag dans la première version.
Mais la seconde version l'a vraiment énormement simplifié, laissant la porte ouvertes aux solutions avec itertools, qui seraient vraiment inadaptées (sauf si elles servent à créer un dictionnaire à chaque mot trouvé, mais ça reste long et laborieux).


