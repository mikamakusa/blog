---
title: CLI et UI en Python
created: 2025-12-07
author:
  - mikamakusa
category:
  - Dev
tags:
  - Python
  - CLI
  - UI
  - NiceGUI
---
# CLI et UI en Python

Il y a quelques temps, je suis arrivé chez un nouveau client...et qui dit nouveau client dit également nouvel environnement à appréhender et, parfois, nouveaux outils sur lesquels se former.  
Et bien que j'avais une vague idée de type d'outils avant d'arriver, en jetant un oeil à l'appel d'offre et en lisant la stack technique, il faut avouer que j'étais relativement loin du compte.  
Parmi la fameuse stack technique, j'ai pu voir :  
- Dataiku : les certifications sont gratuites, il y en a 6 assortie d'un période d'essai au service,  
- Concourse : j'avoue avoir été curieux, et ça me rappelait Jenkins (en beaucoup moins *eye-candy*),  
- XLDeploy/XLRelease  
- Digdash
Ca, c'était pour les technologies que je ne connaissais pas encore, pour le reste...du très classique :  
- Gitlab
- Jenkins
- Confluence / Jira

Et il arrive souvent que l'on tombe sur des outils développés en interne...et je n'y ai pas échappé.

Cependant là n'est pas le sujet de cet article...enfin presque parce que la CLI et l'UI ont eu pour thème principal le monitoring d'une application développée en interne.

## Concernant la CLI
Sur ce sujet, j'ai eu plusieurs point technique à traiter :  
- La connexion SSH au serveur,  
- La sélection des logs,  
- Le formatage des chaînes de retour,  
- La création des rapports.

Mon prédécesseur, lors de ma formation, me disait de check les logs de l'application chaque jour au cas ou un des batchs fonctionnait mal...sans pour autant préciser le nombre exact de logs à vérifier (parce qu'il y en a une quinzaine, sur deux serveurs différents).  
Dans ma grande naïveté, j'ai cru qu'un seul suffisait...

Ce fut lorsque la première erreur survint que j'ai compris...l'erreur n'était pas dans le log que je venais de contrôler.  
Par conséquent, j'ai cherché le moyen "le plus simple" de contrôler plusieurs logs...une CLI.

## Le choix des armes
Une CLI, j'en avais déjà fait, je connaissais Click et Argparse...et je préférais le second pour sa simplicité.

### ARGPARSE
```python
import argparse

parser = argparse.ArgumentParser(description='Process some integers.')
subparsers = parser.add_subparsers(dest="cmd1", help="command 1 or 2")
parser.add_argument('arg1', type=str, help='The first number to add.')
parser.add_argument('arg2', type=str, help='The second number to add.')
subparsers = parser.add_subparsers(dest="cmd2", help="command 1 or 2")
parser.add_argument('arg3', type=str, help='The first number to add.')
parser.add_argument('arg4', type=str, help='The second number to add.')

[...] ## La logique de la CLI

args = parser.parse_args()

if args.commande == "cmd1":
    report_backup(args)
elif args.commande == "cmd2":
    restore_backup(args)
else:
    parser.print_help()
```

### TYPER ou CLICK
Les libraires sont relativement similaires, il suffit de faire un tour sur la documentation officielle.  
```python
import typer
app = typer.Typer()

@app.command()
def hello():
	print("Hello !")

@app.command()
def goodbye():
	print("Goodbye !")

if __name__ == "__main__":
    app()
```
C'est la version simple, mais évidement si vous souhaitez ajouter des options telles que *user*, *password* ou des options facultatives, c'est toujours faisable :  
```python
import typer

app = typer.typer()

@app.command()
def connect(user: str = typer.Option(default=...),
            password: str = typer.Option(prompt=True, confirmation_prompt=True, hide_input=True),
            url: str = typer.Option(default="")
            ):
    print(f"connect to {url} with login {user} and password {password}")

if __name__ == "__main__":
    app()
```

## Concernant les UI
Les framework GUI pour Python sont légions et plus ou moins accessibles, mais il y en a un que j'ai découvert récemment et qui m'a totalement séduit, il s'agit de **NiceGUI**.  
Tellement simple à utiliser qu'il ferait presque passer **MkDocs** pour un meuble IKEA à monter sans notice explicative.  
Je m'emballe peut-être un peu, mais c'est plus ou moins l'effet que ça m'a fait quand j'ai commencé un projet chez un client.  

Par exemple, vous souhaitez ajouter un élément texte ?
```python
from nicegui import ui

ui.label("test")
```

Formater en gras l'élément texte ?  
```python
ui.label("test").mark("strong")
```

Transformer complétement l'élément question avec du **CSS** ?
```python
from nicegui import ui

ui.add_head_html('''
    <style type="text/tailwindcss">
        @layer components {
            .green-box {
                @apply bg-green-500 p-12 text-center shadow-lg rounded-lg text-white;
            }
        }
    </style>
''')

ui.label("test").classes("green-box")
```

Vous pouvez également créer des tableaux, les formater de la manière que vous souhaitez.  
Par exemple, quatres colonnes dont une non affichée et un formatage conditionnel d'une colonne en fonction de sa valeur :  
```python
from nicegui import ui

columns = [
    {
        'name': 'server',
        'label': 'Server',
        'field': 'server'
    },
    {
        'name': 'url',
        'label': 'URL',
        'field': 'url'
    },
    {
        'name': 'status',
        'label': 'Status',
        'field': 'status'
    },
    {
        'name': 'purpose',
        'label': 'Purpose',
        'field': 'purpose',
        'classes': 'hide-col',
        'headerClasses': 'hide-col'
    }
]
table = ui.table(columns=columns,rows=[{
    'server': 'xbp10n',
    'url': 'xbp10n.info.intra',
    'status': 'active',
    'purpose': 'web'
    }],
    row_key='server',
    column_default={
        'align': 'center',
        'headerclasses': 'uppercase text-primary'
    }
                 )
table.add_slot('body-cell-status', '''
    <q-td key="status" :props="props">
        <q-badge :color="props.value != active ? 'red' : 'green'">
            {{ props.value }}
        </q-badge>
    </q-td>
''')
```

Je vous invite à jeter un oeil sur le [site officiel](https://nicegui.io/) afin de vous faire une idée de la simplicité et de la puissance de cette librairie.