# Notes article

## Différentes méthodes pour la factorisation QR

Le coût de la facto QR dépend du nombre de non-zéros introduits pendant la factorisation.

* Householder: Très efficace sur les matrices denses

* Given Rotations: Méthode utilisée historiquement sur les matrices creuses pour réduire le fill-in, mais est peu efficace (structure de données, consommation mémoire)

* Multifrontal method: Découpe la matrice en sous-matrices denses(**frontals**) sur lesquels seront appliqués un Householder.
  * Permet une meilleure utilisation de la mémoire
  * Le découpage sous forme d'arbre permet d'extraire une première forme de parallèlisme
  * Problème de passage à l'échelle sur les archis modernes (beaucoup de parallèlisme). Les auteurs proposeront ici une nouvelle stratégie de parallélisation)
  * Utilise un arbre d'élimination
  * Arbre d'élimination:
    * $n$ noeuds, $n$ étant le nombre de colonnes de $A$
    * Chaque noeud représente une étape de pivotage de la factorisation QR et est associé à une matrice dense (appelée $H$) qui contient les coefficients affectés par l'élimination du pivot
    * Le parcours de cet arbre se fait de bas en haut (implique un parallèlisme décroissant)
    * Les noeuds dont les lignes ont la même structure sont regroupés
    * L'arbre de ces noeuds regroupés est appelé *assembly tree*

> Au niveau de l'aspect mathématique de l'article, c'est tout ce que j'ai saisi, autant dire que c'est juste les grandes lignes. Je serais incapable de décrire le fonctionnement du *householder* ou des *given rotations*.

## Solutions pour augmenter le parallèlisme de la *multifrontal method*

### **Différentes formes de parallèlisme**

Deux formes de parallélisme existent:

* *Tree parallelism*: Utiliser les branches de l'arbre comme source de parallélisation.
* *Node parallelism*: Sur des **frontales** suffisament grandes, les opérations BLAS (multiplication, produit scalaire, factorisations, etc) peuvent être réalisées de manière parallèle
  * Traditionellement, ce parallèlisme est géré par les routines BLAS, ce qui pose un problème de *scalabilité*:
    * Le nombre de threads dédiés à BLAS doit être choisi à l'avance
        > Ça n'est plus forcément vrai sur les dernières versions. Le contrôle du placement des threads reste malgré tout moins précis. 
    * L'arbre d'élimination offrant de moins en moins de parallélisme, il y a un manque à gagner
    * Ajoute de la synchronisation: les coefficients de toutes les frontales enfants doivent être calculés avant de factoriser la frontale courante.
        > Je suis pas sûr de comprendre ce point. Le fait de calculer seulement une partie des coefficients des frontales enfants permet de commencer à calculer la frontale courante ?

L'objectif de ce papier est de réunir la gestion de ces deux formes de paralléliseme au sein d'un même *framework*.

### **Différents types de taches**

Une frontale est calculée en 5 étapes:

| Lettre | Nom      | Description |
| ------ | -------- | --- |
| a      | Activate | Calculs d'indices, allocation de la mémoir, etc |
| p      | Panel    | Calcule la factorisation QR d'une colonne-bloc |
| u      | Update   | Applique les réflecteurs trouvés précédemments |
| s      | Assemble | Pour une colonne-bloc, assemble les différentes contributions dans le noeud parent |
| c      | Clean    | Enregistre les coefficients trouvés et les réfélecteurs $H$ et nettoie la mémoire |

Le DAG permet d'exprimer un meilleur parallélisme que l'*assembly tree* en exposant les deux niveaux de parallélisme. Il permet aussi de commencer certaines tâches plus tôt.

### **Ordonancement**

L'ordonancement se base sur deux critères:

* La localité des données
* Le *Fan-out*, c'est à dire le nombre des tâches libérées lorsqu'une tâche s'achève

Pas mal d'opérations sont *memory bounded*, ce qui implique un gros impacte de la localité des données. Le système d'*ownership* est mis en place: Le thread qui active une frontale en devient propriétaire et est privilégié pour exécuter les autres tâches liées à cette frontale

Chaque thread possède une *queue* qui contient les tâches prêtes à être exécutées liées à la **frontale** qu'il possède. Si le nombre de tâche dans cette *queue* devient trop faible, la fonction *fill_queue()* est appelée pour la repeupler.

> *fill_queue()* cherche les tâches disponibles parmi les frontales activées et les pousse dans la *queue* du thread propriétaire (càd le thread qui a appelé *fill_queue()*). Si une frontale peut être traitée mais n'est pas activé, une tâche d'activation est enfilée.

> L'insertion dans la *queue* peut se faire via la *head* ou la *tail*, en fonction de la priorité de la tâche (càd son *fan-out*).

Si une *queue* devient vide, le vol de tâche s'applique et se fait dans les *queues* de localité mémoire la plus proche.

### **Optimisations de l'*assembly tree***

Si l'*assembly tree* est très large ou trop de petits noeuds, cette phase de recherche de tâches disponibles est très coûteuse. Deux solutions sont proposées:

* *Logical pruning*: Si l'arbre est trop large (càd qui expose plus de parallèlisme que nécessaire), certaines tâches sont regroupées en une seule grosse tâche, ce qui permet de réduire un sous-arbre en un simple noeud.
* *Tree reordering*: $P_i$ est le nombre de frontales actives dans le sous-arbre de racine $i$. $P_i$ est le maximum entre:
  * $nc_i + 1$, $nc_i$ étant le nombre de noeuds enfants du noeud $i$
  * $max_{j=1,...,nc_i}(j - 1 + P_j)$
  
  Les enfants de $i$ sont ensuite triés par ordre décroissant de leur $P_j$.
  > Pour le *tree reordering*, je suis pas sûr à 100%, faudrait que j'essaie de faire tourner l'algo avec leur exemple.

Ces deux optimisations sont réalisées en amont, lors de la phase d'analyse, qui n'est pas traitée dans leur article.

### ***Blocking***

Les techniques de *blocking* rajoute du *fill-in* dans les réflecteurs (exploitation partielle de la structure en escalier). Le choix de la taille des blocs $ib$ se fait en fonction de la largeur des colonnes-bloc $nb$. Une condition à respecter est que $nb$ doit être un multiple de $ib$. La taille du bloc nécessite donc de trouver un compromis entre *fill-in* et localité des données.

## Expérimentations

### **Conditions de tests**

* 50 matrices du *UF Sparse Matrix Collection*
  > Sont exclues les matrices trop petites (pas de *scalabilité*), et les trop grandes (impossible à calculer pour la machine de test)
* 1 matrice d'HIRLAM (pour application météorologique)
* Tests des couples $(ip,nb)$ suivants:
  * $(120, 120)$
  * $(120, 60)$
  * $(60, 60)$
  > Ces valeurs ont été trouvées expérimentalement, et semble plutôt efficaces quelque soit le nombre de coeurs

### **Résultats**

Trois variantes sont testées:

* *Single queue*: Chaque thread pop une tâche dès qu'il est prêt
* *locality*: Chaque thread à une file
  > Utilise l'ordonancement présenté précédemment
* *Round-robin*: Utilise la variante *locality* mais les allocations se font par cycle sur tous les contrôlleurs de DRAM. L'intérêt est de casser la localité des données

De manière générale, il se dégage que *round-robin* > *locality* > *single queue*.

Si la variante *locality* permet de réduire le nombre de mots transférés sur l'*hyper transport link*, elle augmente néammoins légérement le nombre de conflits au niveau des contrôlleurs de DRAM.

Quant à la variante *round-robin*, elle augmente légérement le nombre de mots transférés sur l'*HTL* mais réduit le nombre de conflits des contrôlleurs de DRAM.

> Il semble donc intéressant de privilégier une réduction des conflits au niveau des contrôlleurs de DRAM plutôt qu'une réduction des transferts sur l'HTL. Cependant, l'idéal serait de réussir à combiner les deux.

Les temps obtenus sont jusqu'à 3 fois inférieurs à ceux obtenus avec la méthode "classique".

Le *speedup* stage aux alentours de 24 threads
> Les tâches *panel* sont inefficaces (chemin critique du DAG). Cela se remarque encore plus quand les matrices ont beaucoup plus de lignes que de colonnes.

## Améliorations

* Couper le problème en deux dimensions (au lieu d'une actuellement) pour augmenter le *node parallelism*.
* *Tree preprocessing*: Découper encore plus l'arbre (virtuellement) pour augmenter le parallélisme.

