# TP Hadoop sous Ubuntu — Clusters, HDFS, YARN, MapReduce et Python

**Auteur :** Mohamed
**Environnement :** Ubuntu Server, OpenJDK 11, Hadoop 3.3.6 (mode pseudo-distribué), Python 3

## Contenu du dépôt

```
.
├── README.md          Compte rendu (ce fichier)
├── scripts/           Mappers, reducers et combiner Python (exercices 2 à 6)
├── data/              Fichiers de données du TP
├── images/            Captures d'écran
└── docs/plan_video.md Plan de la vidéo de démonstration
```

---

## Partie 1 — Les clusters : définition et présentation

Un **cluster** est un ensemble d'ordinateurs indépendants (les nœuds), reliés par un réseau rapide et coordonnés par un logiciel, de façon à fonctionner comme un seul système. **Hadoop** est un framework libre en Java pour stocker (HDFS) et traiter (MapReduce sur YARN) de très gros volumes de données sur un cluster de machines ordinaires.

![Architecture maître / esclaves](images/00_architecture.png)

---

## Partie 2 — Installation d'Hadoop en mode pseudo-distribué

### 2.1 Mise à jour du système et installation de Java

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openjdk-11-jdk ssh pdsh wget python3
java -version
```

![java -version](images/01_java_version.png)

### 2.2 Création d'un utilisateur dédié

```bash
sudo adduser hadoop
sudo usermod -aG sudo hadoop
su - hadoop
```

### 2.3 Configuration de SSH sans mot de passe

```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
ssh localhost
```

### 2.4 Téléchargement et installation d'Hadoop

```bash
cd ~
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xzf hadoop-3.3.6.tar.gz
mv hadoop-3.3.6 hadoop
```

### 2.5 Variables d'environnement (`~/.bashrc`)

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_HOME=$HOME/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export PDSH_RCMD_TYPE=ssh
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

```bash
source ~/.bashrc
hadoop version
```

![hadoop version](images/02_hadoop_version.png)

### 2.6 Fichiers de configuration

Dans `$HADOOP_HOME/etc/hadoop/` : `hadoop-env.sh` (JAVA_HOME), `core-site.xml` (`fs.defaultFS = hdfs://localhost:9000`), `hdfs-site.xml` (réplication = 1), `mapred-site.xml` (framework YARN) et `yarn-site.xml` (service `mapreduce_shuffle`).

| Fichier | Rôle |
|---|---|
| `hadoop-env.sh` | Définit `JAVA_HOME` pour les démons |
| `core-site.xml` | Adresse du système de fichiers et dossier temporaire |
| `hdfs-site.xml` | Réplication (1 en pseudo-distribué) et dossiers NameNode/DataNode |
| `mapred-site.xml` | Exécute MapReduce sur YARN, variables `HADOOP_MAPRED_HOME` |
| `yarn-site.xml` | Service de shuffle et mémoire du NodeManager |

### 2.7 Formatage de HDFS et démarrage du cluster

```bash
mkdir -p ~/hadoopdata/{namenode,datanode,tmp}
hdfs namenode -format
start-dfs.sh
start-yarn.sh
jps
```

![Sortie de jps](images/03_jps.png)

Les cinq démons attendus : `NameNode`, `DataNode`, `SecondaryNameNode`, `ResourceManager`, `NodeManager`.

### 2.8 Interfaces web

| Interface | Adresse |
|---|---|
| NameNode (HDFS) | http://localhost:9870 |
| ResourceManager (YARN) | http://localhost:8088 |

![Interface NameNode](images/04_namenode_9870.png)
![Interface YARN](images/05_yarn_8088.png)

> Sur Ubuntu Server (sans navigateur), on accède à ces pages depuis Windows grâce à un tunnel SSH :
> `ssh -L 9870:localhost:9870 -L 8088:localhost:8088 hadoop@IP_DU_SERVEUR`

---

## Partie 3 — Manipulation de HDFS

### Exercice 1 — Premiers pas avec HDFS

```bash
hdfs dfs -mkdir -p /user/hadoop/tp/input
mkdir -p ~/tp && cd ~/tp
cat > poeme.txt << 'EOF'
le chat dort sur le tapis
le chien court dans le jardin
le chat court apres le chien
EOF
hdfs dfs -put poeme.txt /user/hadoop/tp/input/
hdfs dfs -ls /user/hadoop/tp/input
hdfs dfs -cat /user/hadoop/tp/input/poeme.txt
hdfs fsck /user/hadoop/tp/input/poeme.txt -files -blocks -locations
```

![Exercice 1 — HDFS](images/06_exercice1_hdfs.png)
![fsck](images/07_exercice1_fsck.png)

**Réponses aux questions**

1) **Combien de blocs `poeme.txt` occupe-t-il ? Pourquoi ?**
Un seul bloc. Le fichier fait 85 octets, bien moins que la taille d'un bloc HDFS (128 Mo par défaut). Un fichier plus petit qu'un bloc occupe un seul bloc, et ce bloc n'utilise sur le disque que la taille réelle des données (HDFS ne réserve pas 128 Mo). Avec `dfs.replication = 1`, il n'existe qu'une seule copie de ce bloc.

2) **Retrouver le fichier dans l'interface web du NameNode.**
Dans http://localhost:9870 : *Utilities → Browse the file system*, puis `/user/hadoop/tp/input/poeme.txt` (capture ci-dessous).

![Browse the file system](images/08_exercice1_browse.png)

3) **Nombre de blocs d'un fichier de 1 Go et espace utilisé avec une réplication de 3 ?**
- 1 Go = 1024 Mo ; 1024 / 128 = **8 blocs**.
- Avec une réplication de 3 : 8 × 3 = **24 blocs stockés**, soit **3 Go** d'espace disque.

---

## Partie 4 — Le modèle MapReduce

MapReduce est un modèle de programmation en deux fonctions écrites par le développeur : `map(clé, valeur) → liste de (clé', valeur')` et `reduce(clé', liste de valeurs') → (clé'', valeur'')`. Hadoop gère le découpage, la distribution, le shuffle & sort et la reprise sur panne.

Phases : Input/Split → **Map** → *Combine* (facultatif) → Partition → Shuffle & Sort → **Reduce** → Output. Les phases écrites par le développeur sont en gras (Map, Reduce, et Combine si utilisé).

Exemple déroulé (WordCount sur « le chat dort », « le chien court », « le chat court ») :

| Étape | Données |
|---|---|
| Map (ligne 1) | (le,1) (chat,1) (dort,1) |
| Map (ligne 2) | (le,1) (chien,1) (court,1) |
| Map (ligne 3) | (le,1) (chat,1) (court,1) |
| Shuffle & Sort | chat→[1,1] chien→[1] court→[1,1] dort→[1] le→[1,1,1] |
| Reduce | (chat,2) (chien,1) (court,2) (dort,1) (le,3) |

---

## Partie 5 — Pratique : MapReduce en Python

Principe d'Hadoop Streaming : le mapper lit stdin et écrit `clé<TAB>valeur` sur stdout ; Hadoop trie par clé ; le reducer lit les lignes triées sur stdin. Test local d'un job complet : `cat fichier | ./mapper.py | sort | ./reducer.py`.

Récupérer les scripts et les données directement sur le serveur :

```bash
sudo apt install -y git
git clone https://github.com/<TON_COMPTE>/<TON_DEPOT>.git ~/tp-hadoop
cd ~/tp-hadoop && chmod +x scripts/*.py
```

### Exercice 2 — WordCount en Python

Scripts : [`scripts/mapper.py`](scripts/mapper.py) et [`scripts/reducer.py`](scripts/reducer.py).

Test local :

```bash
cd ~/tp-hadoop/scripts
cat ../data/poeme.txt | ./mapper.py | sort | ./reducer.py
```

Résultat obtenu :

```
apres	1
chat	2
chien	2
court	2
dans	1
dort	1
jardin	1
le	6
sur	1
tapis	1
```

Exécution sur Hadoop :

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -files mapper.py,reducer.py \
  -mapper "python3 mapper.py" \
  -reducer "python3 reducer.py" \
  -input /user/hadoop/tp/input \
  -output /user/hadoop/tp/output_wordcount

hdfs dfs -ls /user/hadoop/tp/output_wordcount
hdfs dfs -cat /user/hadoop/tp/output_wordcount/part-00000
```

![WordCount sur Hadoop](images/09_exercice2_wordcount.png)
![Job dans YARN](images/10_exercice2_yarn.png)

**Test sur un vrai texte (Les Misérables)** avec comparaison du combiner :

```bash
wget -O ~/tp/miserables.txt https://www.gutenberg.org/cache/epub/17489/pg17489.txt
hdfs dfs -mkdir -p /user/hadoop/tp/livre
hdfs dfs -put ~/tp/miserables.txt /user/hadoop/tp/livre/
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -files mapper.py,reducer.py -mapper "python3 mapper.py" -reducer "python3 reducer.py" \
  -input /user/hadoop/tp/livre -output /user/hadoop/tp/output_livre
hdfs dfs -cat /user/hadoop/tp/output_livre/part-* | sort -k2 -n -r | head -20
```

Avec combiner (ajouter `-combiner "python3 reducer.py"`, sortie `output_livre_comb`) : comparer le compteur *Reduce input records* avec et sans combiner dans l'interface YARN. Le combiner fait une mini-réduction locale à la sortie de chaque Map, donc beaucoup moins de paires circulent sur le réseau vers les reducers.

### Exercice 3 — Chiffre d'affaires par ville

Scripts : [`scripts/mapper_ventes.py`](scripts/mapper_ventes.py) et [`scripts/reducer_ventes.py`](scripts/reducer_ventes.py). Données : [`data/ventes.csv`](data/ventes.csv).

```bash
cat ../data/ventes.csv | ./mapper_ventes.py | sort | ./reducer_ventes.py
hdfs dfs -mkdir -p /user/hadoop/tp/ventes
hdfs dfs -put ../data/ventes.csv /user/hadoop/tp/ventes/
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -files mapper_ventes.py,reducer_ventes.py \
  -mapper "python3 mapper_ventes.py" -reducer "python3 reducer_ventes.py" \
  -input /user/hadoop/tp/ventes -output /user/hadoop/tp/output_ventes
hdfs dfs -cat /user/hadoop/tp/output_ventes/part-00000
```

Résultat (ville, CA total, nombre de ventes) :

```
Abidjan	910000	3
Bouake	445000	2
Korhogo	40000	1
San-Pedro	260000	2
Yamoussoukro	540000	2
```

![Résultat ventes](images/11_exercice3_ventes.png)

### Exercice 4 — Température maximale par année

Scripts : [`scripts/mapper_meteo.py`](scripts/mapper_meteo.py) et [`scripts/reducer_meteo.py`](scripts/reducer_meteo.py). Données : [`data/meteo.txt`](data/meteo.txt).
Le mapper émet `annee<TAB>temperature` ; le reducer conserve le **maximum** au lieu de faire une somme.

```bash
cat ../data/meteo.txt | ./mapper_meteo.py | sort | ./reducer_meteo.py
hdfs dfs -mkdir -p /user/hadoop/tp/meteo
hdfs dfs -put ../data/meteo.txt /user/hadoop/tp/meteo/
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -files mapper_meteo.py,reducer_meteo.py \
  -mapper "python3 mapper_meteo.py" -reducer "python3 reducer_meteo.py" \
  -input /user/hadoop/tp/meteo -output /user/hadoop/tp/output_meteo
hdfs dfs -cat /user/hadoop/tp/output_meteo/part-00000
```

Résultat : `2024 → 35.0` et `2025 → 36.1`.

![Résultat météo](images/12_exercice4_meteo.png)

### Exercice 5 — Moyenne des notes par matière

Scripts : [`scripts/mapper_notes.py`](scripts/mapper_notes.py), [`scripts/combiner_notes.py`](scripts/combiner_notes.py) et [`scripts/reducer_notes.py`](scripts/reducer_notes.py). Données : [`data/notes.csv`](data/notes.csv).

```bash
cat ../data/notes.csv | ./mapper_notes.py | sort | ./reducer_notes.py
hdfs dfs -mkdir -p /user/hadoop/tp/notes
hdfs dfs -put ../data/notes.csv /user/hadoop/tp/notes/
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -files mapper_notes.py,combiner_notes.py,reducer_notes.py \
  -mapper "python3 mapper_notes.py" \
  -combiner "python3 combiner_notes.py" \
  -reducer "python3 reducer_notes.py" \
  -input /user/hadoop/tp/notes -output /user/hadoop/tp/output_notes
hdfs dfs -cat /user/hadoop/tp/output_notes/part-00000
```

Résultat :

```
Informatique	14.50
Maths	13.00
Physique	12.20
```

![Résultat notes](images/13_exercice5_notes.png)

**Question : peut-on utiliser le reducer de moyenne comme combiner ? Justifiez.**
**Non.** La moyenne n'est pas une opération associative sur les moyennes : la moyenne des moyennes n'est pas égale à la moyenne globale, car chaque sous-groupe a un effectif différent.

Contre-exemple pour les Maths : si un Map reçoit les notes 14 et 10 (moyenne 12) et un autre les notes 18, 8 et 15 (moyenne 13,67), la moyenne des moyennes vaut (12 + 13,67) / 2 = 12,83, alors que la vraie moyenne vaut 65 / 5 = **13,00**.

**Solution :** le mapper émet `matiere<TAB>note<TAB>1`. Le combiner additionne les notes et les effectifs (`matiere<TAB>somme<TAB>effectif`) sans calculer de moyenne, avec le même format que celui du mapper ; seul le reducer divise la somme totale par l'effectif total. Résultat identique avec et sans combiner (vérifié).

### Exercice 6 — Index inversé

Scripts : [`scripts/mapper_index.py`](scripts/mapper_index.py) et [`scripts/reducer_index.py`](scripts/reducer_index.py). Données : [`data/index/`](data/index/) (doc1, doc2, doc3).
Le mapper lit le nom du fichier courant dans `mapreduce_map_input_file` et émet `mot<TAB>fichier` ; le reducer dresse la liste, sans doublon, des fichiers contenant chaque mot.

```bash
hdfs dfs -mkdir -p /user/hadoop/tp/index
hdfs dfs -put ../data/index/*.txt /user/hadoop/tp/index/
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -files mapper_index.py,reducer_index.py \
  -mapper "python3 mapper_index.py" -reducer "python3 reducer_index.py" \
  -input /user/hadoop/tp/index -output /user/hadoop/tp/output_index
hdfs dfs -cat /user/hadoop/tp/output_index/part-00000
```

Extrait du résultat attendu :

```
cluster	doc1.txt, doc2.txt
donnees	doc1.txt, doc2.txt
hadoop	doc1.txt, doc3.txt
le	doc1.txt, doc2.txt, doc3.txt
python	doc3.txt
```

![Résultat index inversé](images/14_exercice6_index.png)

### 5.5 Plusieurs reducers

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -D mapreduce.job.reduces=3 \
  -files mapper.py,reducer.py \
  -mapper "python3 mapper.py" -reducer "python3 reducer.py" \
  -input /user/hadoop/tp/livre -output /user/hadoop/tp/output_3red
hdfs dfs -ls /user/hadoop/tp/output_3red
```

![3 reducers](images/15_reducers_3.png)

On observe trois fichiers `part-00000`, `part-00001` et `part-00002` : chaque reducer reçoit une partie des clés (partitionnement par hachage de la clé modulo le nombre de reducers), donc chaque mot n'apparaît que dans un seul fichier.

---

## Partie 6 — Dépannage

Problèmes rencontrés et solutions (à compléter avec ton expérience) :

| Symptôme | Cause | Solution |
|---|---|---|
| *(à compléter)* | | |

---

## Questions de synthèse

**1. Définition d'un cluster et trois avantages.**
Un cluster est un ensemble d'ordinateurs indépendants (nœuds) reliés par un réseau rapide et pilotés par un logiciel, de façon à se comporter comme un seul système. Avantages : (1) stocker de très gros volumes en répartissant les fichiers sur de nombreux disques ; (2) traiter les données en parallèle, donc beaucoup plus vite ; (3) haute disponibilité : si un nœud tombe, les autres prennent le relais. On peut aussi citer l'évolutivité (ajout de nœuds) et le faible coût (matériel standard).

**2. Scalabilité verticale et horizontale ; laquelle Hadoop exploite-t-il ?**
La scalabilité verticale (scale-up) consiste à rendre une machine plus puissante (plus de CPU, de RAM) : elle atteint vite un plafond matériel, coûte cher et laisse un point unique de défaillance. La scalabilité horizontale (scale-out) consiste à ajouter des machines au cluster : elle est quasi illimitée, avec un coût linéaire. Hadoop exploite la **scalabilité horizontale**.

**3. Rôle du NameNode ; pourquoi est-il un point sensible ?**
Le NameNode gère l'espace de noms HDFS (arborescence, droits) et sait sur quels DataNodes se trouve chaque bloc ; il ne stocke aucune donnée utilisateur. Il est sensible car il est unique dans une installation simple : s'il tombe ou si ses métadonnées sont perdues, HDFS devient inaccessible (les blocs ne sont plus localisables). C'est un point unique de défaillance, que l'on atténue en production avec un NameNode actif/standby (haute disponibilité).

**4. Le Secondary NameNode est-il une copie de secours ?**
**Non.** Il fusionne périodiquement le journal des modifications (`edits`) avec l'image du système de fichiers (`fsimage`) pour éviter que le journal ne grossisse indéfiniment. Ce n'est pas un NameNode de secours et il ne prend pas le relais en cas de panne.

**5. Localité des données et importance.**
C'est le principe « déplacer le calcul vers les données » : on envoie le programme (petit) sur le nœud qui stocke le bloc (volumineux) plutôt que de transférer les données par le réseau. Cela réduit fortement le trafic réseau, qui est le goulot d'étranglement, et accélère les traitements.

**6. Phases d'un job MapReduce, dans l'ordre, et celles écrites par le développeur.**
1) Input / Split (Hadoop) ; 2) **Map** (développeur) ; 3) **Combine**, facultatif (développeur) ; 4) Partition (Hadoop) ; 5) Shuffle & Sort (Hadoop) ; 6) **Reduce** (développeur) ; 7) Output (Hadoop).

**7. Pourquoi le reducer Python peut-il supposer que les lignes d'une même clé sont consécutives ?**
Parce qu'entre le Map et le Reduce, Hadoop effectue le Shuffle & Sort : les paires sont triées par clé avant d'être envoyées au reducer. Toutes les lignes d'une même clé arrivent donc à la suite, ce qui permet d'agréger avec une simple variable « clé courante » sans garder toute la table en mémoire.

**8. Quand un combiner est-il utilisable ? Exemple où il ne l'est pas directement.**
Un combiner est utilisable quand l'opération est associative et commutative (somme, comptage, maximum, minimum) : appliquer la réduction par morceaux puis sur les résultats partiels donne le même résultat. Il ne l'est pas directement pour la **moyenne** : la moyenne des moyennes partielles diffère de la moyenne globale ; il faut faire circuler (somme, effectif).

**9. Que se passe-t-il si un nœud tombe pendant une tâche Map ?**
Le NodeManager n'envoie plus de heartbeat : le ResourceManager et l'ApplicationMaster détectent la panne et relancent automatiquement la tâche Map sur un autre nœud. Grâce à la réplication, une copie du bloc d'entrée existe sur un autre DataNode. Les sorties intermédiaires perdues sont recalculées et le job continue sans intervention.

**10. Deux limites de MapReduce et un outil de l'écosystème qui y répond.**
(1) Lourd pour les traitements itératifs, car chaque job écrit sur disque entre deux étapes ; (2) latence élevée, donc inadapté au temps réel. **Spark** y répond en calculant en mémoire. On peut aussi citer : tous les problèmes ne s'expriment pas facilement en map/reduce (Hive permet alors des requêtes de type SQL).

---

## Ce que j'ai appris

*(à compléter en quelques lignes avec ton ressenti personnel sur le TP)*

## Vidéo de démonstration

*(lien de la vidéo à ajouter ici : YouTube non répertorié, Google Drive, etc.)*
