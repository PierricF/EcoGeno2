**CC1 Pierric MATHIS- -FUMEL**
================

#Installation des packages  
*non-exécuté*

``` bash
sudo apt-get update -y
sudo apt-get install -y libglpk-dev 
sudo apt-get install -y liblzma-dev libbz2-dev
```

``` r
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install("BiocStyle")
BiocManager::install("Rhtslib")
```

``` r
library("knitr")
library("BiocStyle")
.cran_packages <- c("ggplot2", "gridExtra", "devtools")
install.packages(.cran_packages) 
.bioc_packages <- c("dada2", "phyloseq", "DECIPHER", "phangorn")
BiocManager::install(.bioc_packages)
sapply(c(.cran_packages, .bioc_packages), require, character.only = TRUE)
```

``` bash
cd ~
wget https://mothur.s3.us-east-2.amazonaws.com/wiki/miseqsopdata.zip
unzip miseqsopdata.zip
```

*exécuté*

#Méthodes ##Bioinformatique des amplicons : des reads bruts aux
tableaux  
Construction du tableau des caractéristiques échantillon par séquence à
partir des reads brutes, attribution de la taxonomie et création d’un
arbre phylogénétique reliant les séquences des échantillons.  
Tout d’abord, nous chargeons les packages nécessaires.

``` r
library("knitr")
library("BiocStyle")
.cran_packages <- c("ggplot2", "gridExtra")
.bioc_packages <- c("dada2", "phyloseq", "DECIPHER", "phangorn")
sapply(c(.cran_packages, .bioc_packages), require, character.only = TRUE)
```

    ##   ggplot2 gridExtra     dada2  phyloseq  DECIPHER  phangorn 
    ##      TRUE      TRUE      TRUE      TRUE      TRUE      TRUE

``` r
set.seed(100)
```

Les données que nous analyserons ici sont des séquences d’amplicons
Illumina Miseq 2x250 hautement chevauchantes de la région V4 du gène 16S
(Kozich et al. 2013). Ces 360 échantillons fécaux ont été prélevés sur
12 souris de manière longitudinale au cours de la première année de vie
; un contrôle communautaire fictif. Ils ont été collectés pour étudier
le développement et la stabilisation du microbiome murin (Schloss et
al. 2012). Ces données sont téléchargées à partir de l’emplacement de
données suivant et dézippées. Pour l’instant, il suffit de les
considérer comme des fichiers fastq paired-end à traiter.  
Définir la variable path afin qu’elle désigne le répertoire extrait sur
la machine :

``` r
miseq_path <- "/home/rstudio/MiSeq_SOP"
list.files(miseq_path)
```

    ##  [1] "F3D0_S188_L001_R1_001.fastq"   "F3D0_S188_L001_R2_001.fastq"  
    ##  [3] "F3D1_S189_L001_R1_001.fastq"   "F3D1_S189_L001_R2_001.fastq"  
    ##  [5] "F3D141_S207_L001_R1_001.fastq" "F3D141_S207_L001_R2_001.fastq"
    ##  [7] "F3D142_S208_L001_R1_001.fastq" "F3D142_S208_L001_R2_001.fastq"
    ##  [9] "F3D143_S209_L001_R1_001.fastq" "F3D143_S209_L001_R2_001.fastq"
    ## [11] "F3D144_S210_L001_R1_001.fastq" "F3D144_S210_L001_R2_001.fastq"
    ## [13] "F3D145_S211_L001_R1_001.fastq" "F3D145_S211_L001_R2_001.fastq"
    ## [15] "F3D146_S212_L001_R1_001.fastq" "F3D146_S212_L001_R2_001.fastq"
    ## [17] "F3D147_S213_L001_R1_001.fastq" "F3D147_S213_L001_R2_001.fastq"
    ## [19] "F3D148_S214_L001_R1_001.fastq" "F3D148_S214_L001_R2_001.fastq"
    ## [21] "F3D149_S215_L001_R1_001.fastq" "F3D149_S215_L001_R2_001.fastq"
    ## [23] "F3D150_S216_L001_R1_001.fastq" "F3D150_S216_L001_R2_001.fastq"
    ## [25] "F3D2_S190_L001_R1_001.fastq"   "F3D2_S190_L001_R2_001.fastq"  
    ## [27] "F3D3_S191_L001_R1_001.fastq"   "F3D3_S191_L001_R2_001.fastq"  
    ## [29] "F3D5_S193_L001_R1_001.fastq"   "F3D5_S193_L001_R2_001.fastq"  
    ## [31] "F3D6_S194_L001_R1_001.fastq"   "F3D6_S194_L001_R2_001.fastq"  
    ## [33] "F3D7_S195_L001_R1_001.fastq"   "F3D7_S195_L001_R2_001.fastq"  
    ## [35] "F3D8_S196_L001_R1_001.fastq"   "F3D8_S196_L001_R2_001.fastq"  
    ## [37] "F3D9_S197_L001_R1_001.fastq"   "F3D9_S197_L001_R2_001.fastq"  
    ## [39] "filtered"                      "HMP_MOCK.v35.fasta"           
    ## [41] "Mock_S280_L001_R1_001.fastq"   "Mock_S280_L001_R2_001.fastq"  
    ## [43] "mouse.dpw.metadata"            "mouse.time.design"            
    ## [45] "stability.batch"               "stability.files"

##Filtres et ajustements Nous commençons par filtrer les lectures de
séquençage de mauvaise qualité et par ajuster les lectures à une
longueur cohérente. Bien que les paramètres de filtrage et d’ajustement
généralement recommandés servent de point de départ, il n’existe pas
deux ensembles de données identiques et il est donc toujours utile
d’inspecter la qualité des données avant de poursuivre.  
Tout d’abord, nous lisons les noms des fichiers fastq, et nous
effectuons quelques manipulations de chaînes pour obtenir des listes de
fichiers fastq forward et reverse dans l’ordre correspondant :

``` r
# Sort garantit que les reads forward/reverse sont dans le même ordre
fnFs <- sort(list.files(miseq_path, pattern = "_R1_001.fastq"))
fnRs <- sort(list.files(miseq_path, pattern = "_R2_001.fastq"))
# Extraire les noms des échantillons, en considérant que les noms de fichiers ont le format : SAMPLENAME_XXX.fastq
sampleNames <- sapply(strsplit(fnFs,"_"),'[',1)    #fnFs transformé, strsplit=on garde juste la première partie d'une liste de caractères séparés par underscore  <=> on transforme le nom fichier en nom échantillon
# Spécifier le chemin complet vers fnFs et fnRs (forward/reverse)
fnFs <- file.path(miseq_path,fnFs)  #file.path = construire une voie indépendante de la plateforme, utilisable par Unix, Windows, etc.
fnRs <- file.path(miseq_path,fnRs)
fnFs[1:3]
```

    ## [1] "/home/rstudio/MiSeq_SOP/F3D0_S188_L001_R1_001.fastq"  
    ## [2] "/home/rstudio/MiSeq_SOP/F3D1_S189_L001_R1_001.fastq"  
    ## [3] "/home/rstudio/MiSeq_SOP/F3D141_S207_L001_R1_001.fastq"

``` r
fnRs[1:3]
```

    ## [1] "/home/rstudio/MiSeq_SOP/F3D0_S188_L001_R2_001.fastq"  
    ## [2] "/home/rstudio/MiSeq_SOP/F3D1_S189_L001_R2_001.fastq"  
    ## [3] "/home/rstudio/MiSeq_SOP/F3D141_S207_L001_R2_001.fastq"

La plupart des données de séquençage Illumina montrent une tendance à la
baisse de la qualité moyenne vers la fin des lectures de séquençage.

``` r
#Les deux premiers reads forward :
plotQualityProfile(fnFs[1:2]) # =fonction de DADA2
```

    ## Warning: `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> =
    ## "none")` instead.

![](test_rstudio_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

``` r
#en noir = score qualité le plus fréquent à chaque position/nucléotide
#score qualité, va de 0 à 50 en général ; à 30 = 1 chance sur 10^3 que ce soit pas la bonne base, etc.
#en vert = moyenne des scores qualité
#rouge = proportion des reads qui vont au moins jusqu'à cette position  

#Les deux premiers reads reverse :
plotQualityProfile(fnRs[1:2])
```

    ## Warning: `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> =
    ## "none")` instead.

![](test_rstudio_files/figure-gfm/unnamed-chunk-9-2.png)<!-- --> Ici,
les reads forward conservent une qualité élevée tout au long du
processus, tandis que la qualité des reads reverse chute de manière
significative à la position 160 environ. Par conséquent, nous
choisissons de tronquer les reads Forward à la position 245, et les
reads Reverse à la position 160. Nous choisissons également de tronquer
les 10 premiers nucléotides de chaque read en nous basant sur des
observations empiriques sur de nombreux ensembles de données Illumina,
selon lesquelles ces positions de base sont particulièrement
susceptibles de contenir des erreurs.  
(On enlève les bases les moins sures, donc on perd le chevauchement,
mais il faut faire attention et prendre en compte la taille de la région
pour quand même avoir un alignement (au moins 10 pb !).)

Nous définissons les noms de fichiers pour les fichiers fastq.gz filtrés
:

``` r
filt_path <- file.path(miseq_path,"filtered") # Placer les fichiers filtrés dans le sous-répertoire filtered/
if(!file_test("-d",filt_path)) dir.create(filt.path)
filtFs <- file.path(filt_path,paste0(sampleNames, "_F_filt.fastq.gz"))
filtRs <- file.path(filt_path,paste0(sampleNames, "_R_filt.fastq.gz"))
```

Nous combinons ces paramètres de sélection avec des paramètres de
filtrage standard, le plus important étant l’application d’un maximum de
2 erreurs attendues par lecture (Edgar et Flyvbjerg 2015). Ces filtrages
sont effectués conjointement sur les reads appariées, c’est-à-dire que
les deux reads doivent passer le filtre pour que la paire passe.

**Filtrer les reads F et R :**

``` r
out <- filterAndTrim(fnFs,filtFs,fnRs,filtRs,truncLen=c(240,160),
                     maxN=0, maxEE=c(2,2), truncQ=2, rm.phix=TRUE,
                     compress=TRUE, multithread=TRUE) #truncLen=1e valeur pour les reads Forward, 2e pour les Reverse ; qd le séquenceur ne sait pas qoi mettre, il met N, donc on veut nb de N=0 ; truncQ=2 = on enlève toutes les bases qd score arrvive à 20 ; on ajoute l'ADN de phiX, qui sert de témoin, mais faut ê sûr de bien l'enlever ensuite ; compress, car fichiers sont compressés ; multithread=on utilise tous les processeurs de la machine
head(out)
```

    ##                               reads.in reads.out
    ## F3D0_S188_L001_R1_001.fastq       7793      7113
    ## F3D1_S189_L001_R1_001.fastq       5869      5299
    ## F3D141_S207_L001_R1_001.fastq     5958      5463
    ## F3D142_S208_L001_R1_001.fastq     3183      2914
    ## F3D143_S209_L001_R1_001.fastq     3178      2941
    ## F3D144_S210_L001_R1_001.fastq     4827      4312

##Déduire les variants de séquences  
Après le filtrage, le flux de travail bioinformatique typique des
amplicons regroupe les lectures de séquençage en unités taxonomiques
opérationnelles (OTU) : des groupes de lectures de séquençage qui
diffèrent par moins d’un seuil de dissimilarité fixe. Ici, nous
utilisons plutôt la méthode DADA2 à haute résolution pour déduire les
variants de séquences d’amplicons (ASV) de manière exacte, sans imposer
de seuil arbitraire, et ainsi résoudre les variantes qui diffèrent d’un
seul nucléotide (Benjamin J Callahan et al. 2016).  
Les données de séquence sont importées dans R à partir de fichiers fastq
démultiplexés (c’est-à-dire un fastq pour chaque échantillon) et
simultanément dérépliquées pour éliminer la redondance. Nous nommons les
objets de la classe derep résultants par leur nom d’échantillon.

###Déréplication  
La déréplication combine tous les reads de séquençage identiques en
“séquences uniques” avec une “abondance” correspondante : le nombre de
lectures avec cette séquence unique. La déréplication réduit
considérablement le temps de calcul en éliminant les comparaisons
redondantes.

``` r
derepFs <- derepFastq(filtFs, verbose=TRUE)  #dérépliquer   #verbose=on lui demande d'expliquer les étapes qu'il fait
derepRs <- derepFastq(filtRs, verbose=TRUE)
# Nommer les objets de la classe Derep avec les noms des échantillons.
names(derepFs) <- sampleNames
names(derepRs) <- sampleNames
```

La méthode DADA2 s’appuie sur un modèle paramétré d’erreurs de
substitution pour distinguer les erreurs de séquençage de la variation
biologique réelle. Comme les taux d’erreur peuvent varier (et varient
souvent) de manière substantielle entre les cycles de séquençage et les
protocoles PCR, les paramètres du modèle peuvent être découverts à
partir des données elles-mêmes en utilisant une forme d’apprentissage
non supervisé dans laquelle l’inférence de l’échantillon est alternée
avec l’estimation des paramètres jusqu’à ce que les deux soient
cohérents.  
L’apprentissage des paramètres est intensif en termes de calcul, car il
nécessite de multiples itérations de l’algorithme d’inférence de
séquence, et il est donc souvent utile d’estimer les taux d’erreur à
partir d’un sous-ensemble (suffisamment grand) de données.

``` r
errF <- learnErrors(filtFs, multithread=TRUE)
```

    ## 33514080 total bases in 139642 reads from 20 samples will be used for learning the error rates.

``` r
errR <- learnErrors(filtRs, multithread=TRUE)  
```

    ## 22342720 total bases in 139642 reads from 20 samples will be used for learning the error rates.

``` r
plotErrors(errF) # moy des scores q en abscisse pour les différentes positions, qu'importe l'ordre ; en ordonnées, la fqce des erreurs obs par DADA2 en log ; "A2C" = remplacement de A par C, etc.    #applique les corrections direct ds les reads
```

    ## Warning: Transformation introduced infinite values in continuous y-axis

![](test_rstudio_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

``` r
plotErrors(errR)
```

    ## Warning: Transformation introduced infinite values in continuous y-axis

![](test_rstudio_files/figure-gfm/unnamed-chunk-13-2.png)<!-- --> Afin
de vérifier que les taux d’erreur ont été raisonnablement bien estimés,
nous inspectons l’ajustement entre les taux d’erreur observés (points
noirs) et les taux d’erreur ajustés (lignes noires) dans la figure 1.
Ces figures montrent les fréquences de chaque type de transition en
fonction de la qualité.  
(La seule raison de faire des OTUs = erreurs de séquençage ; donc DADA2
permet de ne plus avoir à regrouper en OTUs -> ASVs)  
La méthode d’inférence de séquence DADA2 peut fonctionner selon deux
modes différents : Inférence indépendante par échantillon (pool=FALSE),
et inférence à partir des reads de séquençage regroupées de tous les
échantillons (pool=TRUE). L’inférence indépendante présente l’avantage
que le temps de calcul est linéaire par rapport au nombre
d’échantillons, et que les besoins en mémoire sont constants avec le
nombre d’échantillons. Cela permet de mettre à l’échelle des ensembles
de données de taille presque illimitée. L’inférence groupée est plus
exigeante en termes de temps de calcul et peut devenir intraitable pour
des ensembles de données de plusieurs dizaines de millions de reads.
Cependant, la mise en commun améliore la détection de variantes rares
qui n’ont été observées qu’une ou deux fois dans un échantillon
individuel mais plusieurs fois dans l’ensemble des échantillons. Comme
cet ensemble de données n’est pas particulièrement grand, nous
effectuons une inférence groupée. Depuis la version 1.2, le
multithreading peut être activé avec les arguments multithread = TRUE,
ce qui accélère considérablement cette étape.

``` r
dadaFs <- dada(derepFs, err=errF, multithread=TRUE)  
```

    ## Sample 1 - 7113 reads in 1979 unique sequences.
    ## Sample 2 - 5299 reads in 1639 unique sequences.
    ## Sample 3 - 5463 reads in 1477 unique sequences.
    ## Sample 4 - 2914 reads in 904 unique sequences.
    ## Sample 5 - 2941 reads in 939 unique sequences.
    ## Sample 6 - 4312 reads in 1267 unique sequences.
    ## Sample 7 - 6741 reads in 1756 unique sequences.
    ## Sample 8 - 4560 reads in 1438 unique sequences.
    ## Sample 9 - 15637 reads in 3590 unique sequences.
    ## Sample 10 - 11413 reads in 2762 unique sequences.
    ## Sample 11 - 12017 reads in 3021 unique sequences.
    ## Sample 12 - 5032 reads in 1566 unique sequences.
    ## Sample 13 - 18075 reads in 3707 unique sequences.
    ## Sample 14 - 6250 reads in 1479 unique sequences.
    ## Sample 15 - 4052 reads in 1195 unique sequences.
    ## Sample 16 - 7369 reads in 1832 unique sequences.
    ## Sample 17 - 4765 reads in 1183 unique sequences.
    ## Sample 18 - 4871 reads in 1382 unique sequences.
    ## Sample 19 - 6504 reads in 1709 unique sequences.
    ## Sample 20 - 4314 reads in 897 unique sequences.

``` r
dadaRs <- dada(derepRs, err=errR, multithread=TRUE)
```

    ## Sample 1 - 7113 reads in 1660 unique sequences.
    ## Sample 2 - 5299 reads in 1349 unique sequences.
    ## Sample 3 - 5463 reads in 1335 unique sequences.
    ## Sample 4 - 2914 reads in 853 unique sequences.
    ## Sample 5 - 2941 reads in 880 unique sequences.
    ## Sample 6 - 4312 reads in 1286 unique sequences.
    ## Sample 7 - 6741 reads in 1803 unique sequences.
    ## Sample 8 - 4560 reads in 1265 unique sequences.
    ## Sample 9 - 15637 reads in 3414 unique sequences.
    ## Sample 10 - 11413 reads in 2522 unique sequences.
    ## Sample 11 - 12017 reads in 2771 unique sequences.
    ## Sample 12 - 5032 reads in 1415 unique sequences.
    ## Sample 13 - 18075 reads in 3290 unique sequences.
    ## Sample 14 - 6250 reads in 1390 unique sequences.
    ## Sample 15 - 4052 reads in 1134 unique sequences.
    ## Sample 16 - 7369 reads in 1635 unique sequences.
    ## Sample 17 - 4765 reads in 1084 unique sequences.
    ## Sample 18 - 4871 reads in 1161 unique sequences.
    ## Sample 19 - 6504 reads in 1502 unique sequences.
    ## Sample 20 - 4314 reads in 732 unique sequences.

Inspection de l’objet de classe dada renvoyé par dada :

``` r
dadaFs[[1]] #premier élmt d'une liste, faisant elle-même partie d'une liste
```

    ## dada-class: object describing DADA2 denoising results
    ## 128 sequence variants were inferred from 1979 input unique sequences.
    ## Key parameters: OMEGA_A = 1e-40, OMEGA_C = 1e-40, BAND_SIZE = 16

L’algorithme DADA2 a inféré 128 variants de séquence réels à partir des
1979 séquences uniques du premier échantillon. L’objet dada-class
contient de multiples diagnostics sur la qualité de chaque variante de
séquence inférée (voir help(“dada-class”) pour plus d’informations).  
L’étape d’inférence de séquence DADA2 a éliminé (presque) toutes les
erreurs de substitution et d’indel des données (Benjamin J Callahan et
al. 2016). Nous fusionnons maintenant ensemble les séquences F et R
inférées, en supprimant les séquences appariées qui ne se chevauchent
pas parfaitement, comme un contrôle final contre les erreurs
résiduelles.

##Construire le tableau des séquences et supprimer les chimères La
méthode DADA2 produit un tableau de séquences qui est un analogue à plus
haute résolution du “tableau OTU” commun, c’est-à-dire un tableau de
caractéristiques échantillon par séquence évalué par le nombre de fois
que chaque séquence a été observée dans chaque échantillon.

``` r
mergers <- mergePairs(dadaFs, derepFs, dadaRs, derepRs)   #on aligne les reads F et R, appli sur jeu dadaFs, dérépliqué en derepFs, et idem R
```

``` r
seqtabAll <- makeSequenceTable(mergers[!grepl("Mock", names(mergers))])  #nb de fois que chaq ASVs apparaît ds chaq échantillon -> table d'observations   #exclut les éch contenant "Mock" = commu artificielle pour évaluer notre pipeline
table(nchar(getSequences(seqtabAll)))  #cherche les seq dans la table qu'on vient de construire -> lecture nb de caractères -> table de fréquence
```

    ## 
    ## 251 252 253 254 255 
    ##   1  85 186   5   2

Notamment, les chimères n’ont pas encore été supprimées. Le modèle
d’erreur de l’algorithme d’inférence des séquences n’inclut pas de
composante chimérique, et nous nous attendons donc à ce que ce tableau
de séquences comprenne de nombreuses séquences chimériques. Nous
éliminons maintenant les séquences chimériques en comparant chaque
séquence inférée aux autres séquences du tableau, et en éliminant celles
qui peuvent être reproduites en assemblant deux séquences plus
abondantes.

``` r
seqtabNoC <- removeBimeraDenovo(seqtabAll)  # chimères pdt PCR = fragments incomplets, p une raison x l'élongation s'est arrêtée avt fin du V4 -> fragment s'hybride à autre 16S amplifié car zones conservées, donc on a un mélange entre deux gènes de 16S = début d une bactos + fin d une autre bactos ; amplifié au cours des cycles restants ; mais on arrive bien à les enlever.
```

Bien que les nombres exacts varient considérablement selon les
conditions expérimentales, il est typique que les chimères constituent
une fraction substantielle des variants de séquence inférés, mais
seulement une petite fraction de toutes les lectures. C’est ce que l’on
observe ici : les chimères représentent environ 22 % des variants de
séquence déduits, mais ces variants ne représentent qu’environ 4 % du
total des reads.

##Assigner la taxonomie  
L’un des avantages de l’utilisation de loci marqueurs bien classés comme
le gène de l’ARNr 16S est la possibilité de classer les variants de
séquence de manière taxonomique. Le paquet dada2 met en œuvre la méthode
du classificateur bayésien naïf à cette fin (Wang et al. 2007). Ce
classificateur compare les variants de séquence à un ensemble
d’entraînement de séquences classées, et nous utilisons ici l’ensemble
d’entraînement RDP v16 (Cole et al. 2009).  
Le site web du tutoriel dada2 contient des fastas d’entraînement
formatés pour l’ensemble d’entraînement RDP, GreenGenes clusterisé à 97%
d’identité, et la base de données de référence Silva disponible. Pour la
taxonomie fongique, les fichiers de libération General Fasta de la base
de données UNITE ITS peuvent être utilisés tels quels.  
Pour suivre ce workflow, télécharger le fichier rdp_train_set_16.fa.gz,
et le placer dans le répertoire avec les fichiers fastq.

``` bash
cd home/rstudio
wget https://zenodo.org/record/4587955/files/silva_nr99_v138.1_train_set.fa.gz   #acquisition du jeu d'entraînement Silva, *non-exécuté*
```

``` r
fastaRef <- "/home/rstudio/silva_nr99_v138.1_train_set.fa.gz"
taxTab <- assignTaxonomy(seqtabNoC, refFasta=fastaRef, multithread=TRUE)    #assigne taxonomie à chacune de nos séquences
unname(head(taxTab))
```

    ##      [,1]       [,2]           [,3]          [,4]            [,5]            
    ## [1,] "Bacteria" "Bacteroidota" "Bacteroidia" "Bacteroidales" "Muribaculaceae"
    ## [2,] "Bacteria" "Bacteroidota" "Bacteroidia" "Bacteroidales" "Muribaculaceae"
    ## [3,] "Bacteria" "Bacteroidota" "Bacteroidia" "Bacteroidales" "Muribaculaceae"
    ## [4,] "Bacteria" "Bacteroidota" "Bacteroidia" "Bacteroidales" "Muribaculaceae"
    ## [5,] "Bacteria" "Bacteroidota" "Bacteroidia" "Bacteroidales" "Bacteroidaceae"
    ## [6,] "Bacteria" "Bacteroidota" "Bacteroidia" "Bacteroidales" "Muribaculaceae"
    ##      [,6]         
    ## [1,] NA           
    ## [2,] NA           
    ## [3,] NA           
    ## [4,] NA           
    ## [5,] "Bacteroides"
    ## [6,] NA

##Construire l’arbre phylogénétique  
La parenté phylogénétique est couramment utilisée pour informer les
analyses en aval, en particulier le calcul des distances phylogéniques
entre les communautés microbiennes. La méthode d’inférence de séquence
DADA2 est sans référence, nous devons donc construire de novo l’arbre
phylogénétique reliant les variantes de séquence inférées. Nous
commençons par effectuer un alignement multiple à l’aide du paquet R
DECIPHER (Wright 2015).

``` r
seqs <- getSequences(seqtabNoC)   #prend le nom de chaq colonne (=les séqces), donc = liste de séquences
names(seqs) <- seqs #Cela se propage aux étiquettes des branches de l'arbre
alignment <- AlignSeqs(DNAStringSet(seqs), anchor=NA, verbose=FALSE)
```

Le package R phangorn est ensuite utilisé pour construire un arbre
phylogénétique. Ici, nous construisons d’abord un arbre
neighbor-joining, puis nous ajustons un arbre de maximum de
vraisemblance GTR+G+I (Generalized time-reversible with Gamma rate
variation) en utilisant l’arbre neighbor-joining comme point de départ.

``` r
phangAlign <- phyDat(as(alignment,"matrix"),type="DNA")
dm <- dist.ml(phangAlign)
treeNJ <- NJ(dm)  # Note, tip order != sequence order
fit = pml(treeNJ, data=phangAlign)
fitGTR <- update(fit, k=4, inv=0.2)
fitGTR <- optim.pml(fitGTR, model="GTR", optInv=TRUE, optGamma=TRUE,
                    rearrangement="stochastic", control=pml.control(trace=0))
plot(fitGTR)
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-22-1.png)<!-- -->

``` r
detach("package:phangorn", unload=TRUE)
```

##Combiner les données dans un objet phyloseq  
Le paquet phyloseq organise et synthétise les différents types de
données d’une expérience typique de séquençage d’amplicons en un seul
objet de données qui peut être facilement manipulé. La dernière
information nécessaire est l’échantillon de données contenu dans un
fichier .csv. Celui-ci peut être téléchargé sur github :

``` r
samdf <- read.csv("https://raw.githubusercontent.com/spholmes/F1000_workflow/master/data/MIMARKS_Data_combined.csv",header=TRUE)  #métadonnées associées à nos souris
samdf$SampleID <- paste0(gsub("00", "", samdf$host_subject_id), "D", samdf$age-21)    #corriger des erreurs
samdf <- samdf[!duplicated(samdf$SampleID),] # Supprimer les entrées en double pour les lectures inversées      (reçoit les infos de sampleID)  
rownames(seqtabAll) <- gsub("124", "125", rownames(seqtabAll)) # Corriger l'écart
all(rownames(seqtabAll) %in% samdf$SampleID) # TRUE
```

    ## [1] TRUE

``` r
rownames(samdf) <- samdf$SampleID
keep.cols <- c("collection_date", "biome", "target_gene", "target_subfragment",
"host_common_name", "host_subject_id", "age", "sex", "body_product", "tot_mass",
"diet", "family_relationship", "genotype", "SampleID")    #ne garder que ces colonnes
samdf <- samdf[rownames(seqtabAll), keep.cols]   #reçoit les lignes de seqtaball et qui sont dans le vecteur keepcols
```

L’ensemble des données de cette étude - le tableau des caractéristiques
de l’échantillon par séquence, les métadonnées de l’échantillon, les
taxonomies des séquences et l’arbre phylogénétique - peut maintenant
être combiné en un seul objet.

``` r
ps <- phyloseq(otu_table(seqtabNoC, taxa_are_rows=FALSE),   #ps reçoit la commande phyloseq, appliquée à seqtabnoc=table d'observations, pas de lignes qui f=definisent la taxonomie, et on spécifie que ce=est une otu_table
               sample_data(samdf),       #sur table de metadonnees
               tax_table(taxTab),phy_tree(fitGTR$tree))      #sur taxtab ; phytree cest larbre phylogenetique, et cherche larbre ds fitGR
ps <- prune_samples(sample_names(ps) != "Mock", ps) # Retirer l'échantillon fictif (mock)   #prune=enlever, dc on garde les éch qui ne cpntiennent pas Mock
ps
```

    ## phyloseq-class experiment-level object
    ## otu_table()   OTU Table:         [ 218 taxa and 19 samples ]
    ## sample_data() Sample Data:       [ 19 samples by 14 sample variables ]
    ## tax_table()   Taxonomy Table:    [ 218 taxa by 6 taxonomic ranks ]
    ## phy_tree()    Phylogenetic Tree: [ 218 tips and 216 internal nodes ]

#Utilisation de phyloseq  
phyloseq(McMurdie and Holmes 2013) est un package R permettant
d’importer, de stocker, d’analyser et d’afficher graphiquement des
données complexes de séquençage phylogénétique qui ont déjà été
regroupées en unités taxonomiques opérationnelles (OTU) ou, de manière
plus appropriée, débruitées. Il est particulièrement utile lorsqu’il
existe également des données d’échantillon associées, une phylogénie
et/ou une affectation taxonomique de chaque taxon. phyloseq exploite et
s’appuie sur de nombreux outils disponibles dans R pour l’écologie et
l’analyse phylogénétique (ape, vegan, ade4), tout en utilisant des
systèmes graphiques avancés/flexibles (ggplot2) pour produire facilement
des graphiques de qualité de publication de données phylogénétiques
complexes. Le paquet phyloseq utilise un système spécialisé de classes
de données S4 pour stocker toutes les données de séquençage
phylogénétique connexes sous la forme d’un objet de niveau expérimental
unique, autoconsistant et autodescriptif, ce qui facilite le partage des
données et la reproduction des analyses. En général, phyloseq cherche à
faciliter l’utilisation de R pour une analyse efficace, interactive et
reproductible des données de comptage d’amplicons conjointement avec des
covariables importantes de l’échantillon.

##Chargement de données De nombreux cas d’utilisation entraînent le
besoin d’importer et de combiner différentes données dans un objet de
classe phyloseq, ceci peut être fait en utilisant la fonction
import_biom pour lire les fichiers récents au format QIIME, les fichiers
plus anciens peuvent toujours être importés avec import_qiime. Des
détails plus complets peuvent être trouvés sur la page FAQ de
phyloseq.  
Dans la section précédente, les résultats du traitement des séquences de
dada2 ont été organisés dans un objet phyloseq. Nous avons en fait
exécuté dada2 sur un ensemble plus important d’échantillons provenant de
la même source de données. Cet objet a également été sauvegardé au
format RDS sérialisé natif de R. Nous le rechargerons ici pour être plus
complet. Nous allons le recharger ici par souci d’exhaustivité comme
l’objet initial ps. Si vous n’avez pas téléchargé l’ensemble du dépôt,
vous pouvez accéder au fichier ps via github :

``` r
#remplace le ps établi avant, car erreur à la fin ! 
ps_connect <-url("https://raw.githubusercontent.com/spholmes/F1000_workflow/master/data/ps.rds")
ps = readRDS(ps_connect)
ps
```

    ## phyloseq-class experiment-level object
    ## otu_table()   OTU Table:         [ 389 taxa and 360 samples ]
    ## sample_data() Sample Data:       [ 360 samples by 14 sample variables ]
    ## tax_table()   Taxonomy Table:    [ 389 taxa by 6 taxonomic ranks ]
    ## phy_tree()    Phylogenetic Tree: [ 389 tips and 387 internal nodes ]

##Shiny-phyloseq  
Il peut être bénéfique de commencer le processus d’exploration des
données de manière interactive, cela permet souvent de gagner du temps
dans la détection des valeurs aberrantes et des caractéristiques
spécifiques des données. Shiny-phyloseq (McMurdie et Holmes 2015) est
une application web interactive qui fournit une interface utilisateur
graphique au paquet phyloseq. L’objet qui vient d’être chargé dans la
session R dans ce flux de travail convient à l’exploration graphique
avec Shiny-phyloseq.

##Filtrage  
phyloseq fournit des outils utiles pour le filtrage, le sous-ensemble et
l’agglomération des taxons - une tâche qui est souvent appropriée ou
même nécessaire pour une analyse efficace des données de comptage du
microbiome. Dans cette sous-section, nous explorons graphiquement la
prévalence des taxa dans l’ensemble de données de l’exemple, et nous
démontrons comment cela peut être utilisé comme critère de filtrage.
L’une des raisons de filtrer de cette manière est d’éviter de passer
beaucoup de temps à analyser des taxons qui n’ont été vus que rarement
parmi les échantillons. Cette méthode s’avère également utile pour
filtrer le bruit (les taxons qui ne sont en fait que des artefacts du
processus de collecte des données), une étape qui devrait probablement
être considérée comme essentielle pour les ensembles de données
construits à l’aide de méthodes heuristiques de regroupement des OTU,
qui sont notoirement enclines à générer des taxons parasites.

###Filtrage taxonomique  
Dans de nombreux contextes biologiques, l’ensemble des organismes de
tous les échantillons est bien représenté dans la base de données
taxonomique de référence disponible. Si ( et seulement si ) c’est le
cas, il est raisonnable ou même conseillé de filtrer les
caractéristiques taxonomiques pour lesquelles une taxonomie de haut rang
n’a pu être attribuée. Dans ce cas, les caractéristiques ambiguës sont
presque toujours des artefacts de séquence qui n’existent pas dans la
nature. Il devrait être évident qu’un tel filtre n’est pas approprié
pour les échantillons provenant de spécimens mal caractérisés ou
nouveaux, du moins jusqu’à ce que la possibilité de nouveauté
taxonomique puisse être rejetée de manière satisfaisante. Phylum est un
rang taxonomique utile à utiliser à cette fin, mais d’autres peuvent
fonctionner efficacement pour vos données.  
Pour commencer, créer un tableau des comptes de lecture pour chaque
phylum présent dans l’ensemble de données.

``` r
# Afficher les rangs disponibles dans l'ensemble de données
rank_names(ps)
```

    ## [1] "Kingdom" "Phylum"  "Class"   "Order"   "Family"  "Genus"

``` r
# Créer un tableau, nombre de caractéristiques pour chaque phyla
table(tax_table(ps)[, "Phylum"], exclude = NULL)  #on exclut quand le champ Phylum est nul
```

    ## 
    ##              Actinobacteria               Bacteroidetes 
    ##                          13                          23 
    ## Candidatus_Saccharibacteria   Cyanobacteria/Chloroplast 
    ##                           1                           4 
    ##         Deinococcus-Thermus                  Firmicutes 
    ##                           1                         327 
    ##                Fusobacteria              Proteobacteria 
    ##                           1                          11 
    ##                 Tenericutes             Verrucomicrobia 
    ##                           1                           1 
    ##                        <NA> 
    ##                           6

Ceci montre quelques phyla pour lesquels une seule caractéristique a été
observée. Ceux-ci peuvent valoir la peine d’être filtrés, et nous allons
vérifier cela ensuite. D’abord, remarquez que dans ce cas, six
caractéristiques ont été annotées avec un phylum de NA. Ces
caractéristiques sont probablement des artefacts dans un ensemble de
données comme celui-ci, et doivent être supprimées.  
Ce qui suit assure que les caractéristiques avec une annotation de
phylum ambiguë sont également supprimées. Notez la flexibilité dans la
définition des chaînes qui devraient être considérées comme une
annotation ambiguë.

``` r
ps <- subset_taxa(ps, !is.na(Phylum) & !Phylum %in% c("", "uncharacterized")) #on enlève aussi qd phylum est marqué NA
```

Une étape suivante utile consiste à explorer la prévalence des
caractéristiques dans l’ensemble de données, que nous définirons ici
comme le nombre d’échantillons dans lesquels un taxon apparaît au moins
une fois.

``` r
# Calculer la prévalence de chaque caractéristique, la stocker dans une dataframe.                #(Pas nécessaire.)
prevdf = apply(X = otu_table(ps),
               MARGIN = ifelse(taxa_are_rows(ps), yes = 1, no = 2),
               FUN = function(x){sum(x > 0)})
# Ajouter la taxonomie et le nombre total de reads à cette dataframe
prevdf = data.frame(Prevalence = prevdf,
                    TotalAbundance = taxa_sums(ps),
                    tax_table(ps))
```

Existe-t-il des embranchements composés principalement de
caractéristiques à faible prévalence ? Calculer les prévalences totales
et moyennes des caractéristiques dans chaque phylum.

``` r
plyr::ddply(prevdf, "Phylum", function(df1){cbind(mean(df1$Prevalence),sum(df1$Prevalence))})
```

    ##                         Phylum         1     2
    ## 1               Actinobacteria 120.15385  1562
    ## 2                Bacteroidetes 265.52174  6107
    ## 3  Candidatus_Saccharibacteria 280.00000   280
    ## 4    Cyanobacteria/Chloroplast  64.25000   257
    ## 5          Deinococcus-Thermus  52.00000    52
    ## 6                   Firmicutes 179.24771 58614
    ## 7                 Fusobacteria   2.00000     2
    ## 8               Proteobacteria  59.09091   650
    ## 9                  Tenericutes 234.00000   234
    ## 10             Verrucomicrobia 104.00000   104

*Deinococcus-Thermus* apparaît dans un peu plus d’un pour cent des
échantillons, et *Fusobacteria* dans seulement 2 échantillons au total.
Dans certains cas, il pourrait être utile d’explorer ces deux phyla plus
en détail malgré cela (mais probablement pas les deux échantillons de
*Fusobacteria*). Pour les besoins de cet exemple, cependant, ils seront
filtrés de l’ensemble de données.

``` r
# Définir les phyla à filtrer
filterPhyla = c("Fusobacteria", "Deinococcus-Thermus")
# Filtrer les entrées dont le phylum n'est pas identifié.
ps1 = subset_taxa(ps, !Phylum %in% filterPhyla)
ps1
```

    ## phyloseq-class experiment-level object
    ## otu_table()   OTU Table:         [ 381 taxa and 360 samples ]
    ## sample_data() Sample Data:       [ 360 samples by 14 sample variables ]
    ## tax_table()   Taxonomy Table:    [ 381 taxa by 6 taxonomic ranks ]
    ## phy_tree()    Phylogenetic Tree: [ 381 tips and 379 internal nodes ]

###Filtrage de la prévalence  
Les étapes de filtrage précédentes sont considérées comme supervisées,
car elles reposent sur des informations préalables externes à cette
expérience (une base de données de référence taxonomique). L’étape de
filtrage suivante est complètement non supervisée, car elle repose
uniquement sur les données de cette expérience et sur un paramètre que
nous choisirons après avoir exploré les données. Ainsi, cette étape de
filtrage peut être appliquée même dans des contextes où l’annotation
taxonomique est indisponible ou peu fiable.  
D’abord, explorer la relation entre la prévalence et le nombre total de
reads pour chaque caractéristique. Parfois, cela révèle des valeurs
aberrantes qui devraient probablement être supprimées, et donne
également un aperçu des plages de chaque caractéristique qui pourraient
être utiles. Cet aspect dépend beaucoup du plan expérimental et des
objectifs de l’inférence en aval, il faut donc les garder à l’esprit. Il
se peut même que différents types d’inférences en aval nécessitent des
choix différents. Il n’y a aucune raison de s’attendre à l’avance à ce
qu’un seul flux de travail de filtrage soit approprié pour toutes les
analyses.

``` r
# Sous-ensemble des phyla restants
prevdf1 = subset(prevdf, Phylum %in% get_taxa_unique(ps1, "Phylum"))
ggplot(prevdf1, aes(TotalAbundance, Prevalence / nsamples(ps),color=Phylum)) +
  # Inclure une estimation pour le paramètre
  geom_hline(yintercept = 0.05, alpha = 0.5, linetype = 2) +  geom_point(size = 2, alpha = 0.7) +
  scale_x_log10() +  xlab("Total Abundance") + ylab("Prevalence [Frac. Samples]") +
  facet_wrap(~Phylum) + theme(legend.position="none")
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-31-1.png)<!-- --> Chaque
point de la figure 2 représente un taxon différent. L’exploration des
données de cette manière est souvent utile pour sélectionner des
paramètres de filtrage, comme le critère de prévalence minimale que nous
utiliserons pour filtrer les données ci-dessus.  
Parfois, une séparation naturelle dans l’ensemble de données se révèle,
ou du moins, un choix conservateur qui se trouve dans une région stable
pour laquelle de petites modifications du choix auraient un effet mineur
ou nul sur l’interprétation biologique (stabilité). Ici, aucune
séparation naturelle n’est immédiatement évidente, mais il semble que
nous pourrions raisonnablement définir un seuil de prévalence dans une
fourchette de zéro à dix pour cent environ. Il faut veiller à ce que ce
choix n’introduise pas de biais dans une analyse en aval de
l’association de l’abondance différentielle.  
L’exemple suivant utilise 5 % de tous les échantillons comme seuil de
prévalence.

``` r
# Définir le seuil de prévalence comme 5% du total des échantillons
prevalenceThreshold = 0.05 * nsamples(ps)
prevalenceThreshold  
```

    ## [1] 18

``` r
# Exécuter le filtre de prévalence, en utilisant la fonction `prune_taxa()`
keepTaxa = rownames(prevdf1)[(prevdf1$Prevalence >= prevalenceThreshold)]
ps2 = prune_taxa(keepTaxa, ps)
```

##Agglomérer les taxa.  
Lorsque l’on sait qu’il y a beaucoup de redondance fonctionnelle entre
espèces ou sous-espèces dans une communauté microbienne, il peut être
utile d’agglomérer les caractéristiques des données correspondant à des
taxa étroitement liés. L’idéal serait de connaître parfaitement les
redondances fonctionnelles à l’avance, auquel cas on agglomérerait les
taxa en utilisant ces relations définies et la fonction dans phyloseq.
Ce type de données fonctionnelles précises n’est généralement pas
disponible, et différentes associations de microorganismes auront des
ensembles différents de fonctions qui se chevauchent, ce qui complique
la définition de critères de regroupement appropriés.  
Bien que ce ne soit pas nécessairement le critère le plus utile ou le
plus précis du point de vue fonctionnel pour regrouper les
caractéristiques microbiennes (il est parfois loin d’être précis),
l’agglomération taxonomique a l’avantage d’être beaucoup plus facile à
définir à l’avance. En effet, les taxonomies sont généralement définies
à l’aide d’une structure graphique arborescente relativement simple,
comportant un nombre fixe de nœuds internes, appelés “rangs”. Cette
structure est suffisamment simple pour que le paquetage phyloseq
représente les taxonomies sous forme de tableau d’étiquettes de
taxonomie. L’agglomération taxonomique regroupe toutes les “feuilles” de
la hiérarchie qui descendent du rang d’agglomération prescrit par
l’utilisateur, ceci est parfois appelé “glomming”.  
L’exemple de code suivant montre comment on pourrait combiner toutes les
caractéristiques qui descendent du même genre.

``` r
# Combien de genres seraient présents après le filtrage ?
length(get_taxa_unique(ps2, taxonomic.rank = "Genus"))  
```

    ## [1] 49

``` r
ps3 = tax_glom(ps2, "Genus", NArm = TRUE)
```

Si la taxonomie n’est pas disponible ou n’est pas fiable,
l’agglomération par arbre est une alternative “sans taxonomie” pour
combiner des caractéristiques de données correspondant à des taxons
étroitement liés. Dans ce cas, plutôt que le rang taxonomique,
l’utilisateur spécifie une hauteur d’arbre correspondant à la distance
phylogénétique entre les caractéristiques qui devrait définir leur
regroupement. Cette méthode est très similaire au “clustering d’OTU”,
sauf que dans de nombreux algorithmes de clustering d’OTU, la distance
de séquence utilisée n’a pas la même (ou aucune) définition évolutive.

``` r
h1 = 0.4
ps4 = tip_glom(ps2, h = h1)
```

La fonction plot_tree() de phyloseq compare ici les données originales
non filtrées, l’arbre après agglomération taxonomique, et l’arbre après
agglomération phylogénétique. Ils sont stockés comme des objets de tracé
séparés, puis rendus ensemble dans un graphique combiné en utilisant
gridExtra::grid.arrange.

``` r
multiPlotTitleTextSize = 15
p2tree = plot_tree(ps2, method = "treeonly",
                   ladderize = "left",
                   title = "Before Agglomeration") +
  theme(plot.title = element_text(size = multiPlotTitleTextSize))
p3tree = plot_tree(ps3, method = "treeonly",
                   ladderize = "left", title = "By Genus") +
  theme(plot.title = element_text(size = multiPlotTitleTextSize))
p4tree = plot_tree(ps4, method = "treeonly",
                   ladderize = "left", title = "By Height") +
  theme(plot.title = element_text(size = multiPlotTitleTextSize))
```

``` r
#Regrouper les plots ensemble
grid.arrange(nrow = 1, p2tree, p3tree, p4tree)
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-36-1.png)<!-- --> La
figure 3 montre l’arbre original à gauche, l’agglomération taxonomique
au rang de Genus au milieu et l’agglomération phylogénétique à une
distance fixe de 0,4 à droite.

##Transformation de la valeur de l’abondance Il est généralement
nécessaire de transformer les données de comptage du microbiome pour
tenir compte des différences de taille de la bibliothèque, de la
variance, de l’échelle, etc. Le paquet phyloseq fournit une interface
flexible pour définir de nouvelles fonctions pour accomplir ces
transformations des valeurs d’abondance via la fonction
transform_sample_counts(). Le premier argument de cette fonction est
l’objet phyloseq que vous voulez transformer, et le deuxième argument
est une fonction R qui définit la transformation. La fonction R est
appliquée par échantillon, en s’attendant à ce que le premier argument
non nommé soit un vecteur de comptes de taxons dans le même ordre que
l’objet phyloseq. Des arguments supplémentaires sont transmis à la
fonction spécifiée dans le second argument, fournissant un moyen
explicite d’inclure des valeurs pré-calculées, des paramètres/seuils
précédemment définis, ou tout autre objet qui pourrait être approprié
pour calculer les valeurs transformées d’intérêt.  
Cet exemple commence par définir une fonction de tracé personnalisée,
plot_abundance(), qui utilise la fonction de phyloseq pour définir un
graphique d’abondance relative. Nous allons l’utiliser pour comparer
plus facilement les différences d’échelle et de distribution des valeurs
d’abondance dans notre objet phyloseq avant et après transformation.

``` r
plot_abundance = function(physeq,title = "",
                          Facet = "Order", Color = "Phylum"){
  # Sous-ensemble arbitraire, basé sur le phylum, pour le traçage.
  p1f = subset_taxa(physeq, Phylum %in% c("Firmicutes"))
  mphyseq = psmelt(p1f)
  mphyseq <- subset(mphyseq, Abundance > 0)
  ggplot(data = mphyseq, mapping = aes_string(x = "sex",y = "Abundance",
                              color = Color, fill = Color)) +
    geom_violin(fill = NA) +
    geom_point(size = 1, alpha = 0.3,
               position = position_jitter(width = 0.3)) +
    facet_wrap(facets = Facet) + scale_y_log10()+
    theme(legend.position="none")
}
```

Dans ce cas, la transformation convertit les comptages de chaque
échantillon en leurs fréquences, souvent appelées proportions ou
abondances relatives. Cette fonction est si simple qu’il est plus facile
de la définir dans l’appel à la fonction transform_sample_counts().

``` r
# Transformer en abondance relative. Sauvegarde en tant que nouvel objet.
ps3ra = transform_sample_counts(ps3, function(x){x / sum(x)})
```

Nous traçons maintenant les valeurs d’abondance avant et après la
transformation.

``` r
plotBefore = plot_abundance(ps3,"")
plotAfter = plot_abundance(ps3ra,"")
# Combiner chaque graphe en un seul graphique.
grid.arrange(nrow = 2,  plotBefore, plotAfter)
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-39-1.png)<!-- --> La
figure 4 montre la comparaison des abondances originales (panneau
supérieur) et des abondances relatives (panneau inférieur).

##Sous-ensemble par taxonomie Remarque : sur le graphique précédent que
*Lactobacillales* semble être un ordre taxonomique avec un profil
d’abondance bimodal dans les données. Nous pouvons vérifier
l’explication taxonomique de ce modèle en traçant uniquement ce
sous-ensemble taxonomique des données. Pour cela, nous faisons un
sous-ensemble avec la fonction, puis nous spécifions un rang taxonomique
plus précis à l’argument de la fonction que nous avons défini ci-dessus.

``` r
psOrd = subset_taxa(ps3ra, Order == "Lactobacillales")
plot_abundance(psOrd, Facet = "Genus", Color = NULL)
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-40-1.png)<!-- --> La
figure 5 montre les abondances relatives de l’ordre taxonomique des
Lactobacillales, regroupées par sexe de l’hôte et par genre. Ici, il est
clair que la distribution biomodale apparente des Lactobacillales sur le
graphique précédent était le résultat d’un mélange de deux genres
différents, avec l’abondance relative typique de *Lactobacillus*
beaucoup plus grande que celle de *Streptococcus*.  
À ce stade du workflow, après avoir converti les reads bruts en
abondances d’espèces interprétables, et après avoir filtré et transformé
ces abondances pour concentrer l’attention sur des quantités
scientifiquement significatives, nous sommes en mesure d’envisager une
analyse statistique plus attentive. R est un environnement idéal pour
effectuer ces analyses, car il dispose d’une communauté active de
développeurs de paquets qui créent des interfaces simples pour des
techniques sophistiquées. Comme de nombreuses méthodes sont disponibles,
il n’est pas nécessaire de s’engager a priori dans une stratégie
d’analyse rigide. De plus, la possibilité d’appeler facilement des
paquets sans réimplémenter les méthodes permet aux chercheurs d’itérer
rapidement à travers des idées d’analyse alternatives. L’avantage
d’exécuter ce flux de travail complet dans R est que cette transition de
la bioinformatique aux statistiques se fait sans effort.  
Commençons par installer quelques packages qui sont disponibles pour ces
analyses complémentaires :

``` r
.cran_packages <- c( "shiny","miniUI", "caret", "pls", "e1071", "ggplot2", "randomForest", "dplyr", "ggrepel", "nlme", "devtools",
                  "reshape2", "PMA", "structSSI", "ade4",
                  "ggnetwork", "intergraph", "scales")
.github_packages <- c("jfukuyama/phyloseqGraphTest")
.bioc_packages <- c("genefilter", "impute")
# Installer les packages CRAN (s'ils ne sont pas déjà installés)
.inst <- .cran_packages %in% installed.packages()
if (any(!.inst)){
  install.packages(.cran_packages[!.inst],repos = "http://cran.rstudio.com/")
}
```

    ## Installing package into '/usr/local/lib/R/site-library'
    ## (as 'lib' is unspecified)

    ## Warning: package 'structSSI' is not available for this version of R
    ## 
    ## A version of this package for your version of R might be available elsewhere,
    ## see the ideas at
    ## https://cran.r-project.org/doc/manuals/r-patched/R-admin.html#Installing-packages

``` r
.inst <- .github_packages %in% installed.packages()
if (any(!.inst)){
  devtools::install_github(.github_packages[!.inst])
}  
```

    ## Skipping install of 'phyloseqGraphTest' from a github remote, the SHA1 (3fb6c274) has not changed since last install.
    ##   Use `force = TRUE` to force installation

``` r
.inst <- .bioc_packages %in% installed.packages()
if(any(!.inst)){BiocManager::install(.bioc_packages[!.inst])
}
```

Nous étayons ces affirmations en illustrant plusieurs analyses sur les
données de souris préparées ci-dessus. Nous expérimentons plusieurs
variantes de l’ordination exploratoire avant de passer à des tests et à
une modélisation plus formels, en expliquant les contextes dans lesquels
les différents points de vue sont les plus appropriés. Enfin, nous
fournissons des exemples d’analyses de données multitables, en utilisant
une étude dans laquelle les mesures métabolomiques et d’abondance
microbienne ont été collectées sur les mêmes échantillons, afin de
démontrer que le workflow général présenté ici peut être adapté au
multitable.

###Preprocessing Avant d’effectuer les projections multivariées, nous
allons ajouter quelques colonnes à notre échantillon de données, qui
pourront ensuite être utilisées pour annoter les graphiques. Sur la
Figure 6, nous voyons que les âges des souris se répartissent en deux
groupes, et nous créons donc une variable catégorielle correspondant aux
souris jeunes, d’âge moyen et âgées. Nous enregistrons également le
nombre total de comptes observés dans chaque échantillon et transformons
les données en logarithme pour stabiliser approximativement la variance.

``` r
qplot(sample_data(ps)$age, geom = "histogram",binwidth=20) + xlab("age")
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-42-1.png)<!-- --> La
figure 6 montre que la covariable âge appartient à trois groupes
distincts.

``` r
qplot(log10(rowSums(otu_table(ps))),binwidth=0.2) +
  xlab("Logged counts-per-sample")
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-43-1.png)<!-- --> Ces
graphiques préliminaires suggèrent certaines étapes de prétraitement.
L’histogramme de la Figure 6 motive la création d’une nouvelle variable
catégorielle, classant l’âge dans l’un des trois pics.  
L’histogramme de la Figure 7 suggère qu’une transformation log(1+x)
pourrait être suffisante pour normaliser les données d’abondance pour
les analyses exploratoires.  
En fait, cette transformation n’est pas suffisante pour les tests et
lorsque nous effectuons des abondances différentielles, nous
recommandons les transformations de stabilisation de la variance
disponibles dans DESeq2 via la fonction phyloseq_to_deseq2, voir le
tutoriel phyloseq_to_deseq2 ici.  
Comme première étape, nous examinons l’analyse des coordonnées
principales (PCoA) avec la dissimilarité de Bray-Curtis ou la distance
Unifrac pondérée.

``` r
sample_data(ps)$age_binned <- cut(sample_data(ps)$age,
                          breaks = c(0, 100, 200, 400))
levels(sample_data(ps)$age_binned) <- list(Young100="(0,100]", Mid100to200="(100,200]", Old200="(200,400]")
sample_data(ps)$family_relationship=gsub(" ","",sample_data(ps)$family_relationship)
pslog <- transform_sample_counts(ps, function(x) log(1 + x))
out.wuf.log <- ordinate(pslog, method = "MDS", distance = "wunifrac")
```

    ## Warning in UniFrac(physeq, weighted = TRUE, ...): Randomly assigning root as --
    ## GCAAGCGTTATCCGGAATGACTGGGCGTAAAGGGTGCGTAGGTGGTTTGGCAAGTTGGTAGCGTAATTCCGGGGCTCAACCTCGGCGCTACTACCAAAACTGCTGGACTTGAGTGCAGGAGGGGTGAATGGAATTCCTAGTGTAGCGGTGGAATGCGTAGATATTAGGAAGAACACCAGCGGCGAAGGCGATTCACTGGACTGTAACTGACACTGAGGCACGAAAGCGTGGGGAG
    ## -- in the phylogenetic tree in the data you provided.

``` r
evals <- out.wuf.log$values$Eigenvalues
plot_ordination(pslog, out.wuf.log, color = "age_binned") +
  labs(col = "Binned Age") +
  coord_fixed(sqrt(evals[2] / evals[1]))
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-44-1.png)<!-- --> La
figure 8, qui montre l’ordination des données d’abondance enregistrées,
révèle quelques valeurs aberrantes.  
Il s’agit des échantillons des femelles 5 et 6 au jour 165 et des
échantillons des mâles 3, 4, 5 et 6 au jour 175. Nous allons les
supprimer, puisque nous sommes principalement intéressés par les
relations entre les points non aberrants.  
Avant de continuer, nous devons vérifier les deux points aberrants des
femelles - ils ont été pris en charge par le même OTU/ASV, qui a une
abondance relative de plus de 90% dans chacun d’eux. C’est la seule fois
dans l’ensemble des données que cette ASV a une abondance relative aussi
élevée - le reste du temps, elle est inférieure à 20%. En particulier,
sa diversité est de loin la plus faible de tous les échantillons.

``` r
rel_abund <- t(apply(otu_table(ps), 1, function(x) x / sum(x)))
qplot(rel_abund[, 12], geom = "histogram",binwidth=0.05) +
  xlab("Relative abundance")
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-45-1.png)<!-- -->
#Projections d’ordinations différentes  
Comme nous l’avons vu, une première étape importante dans l’analyse des
données du microbiome est de faire une analyse exploratoire non
supervisée. Ceci est simple à faire dans phyloseq, qui fournit de
nombreuses distances et méthodes d’ordination.  
Après avoir documenté les valeurs aberrantes, nous allons calculer des
ordinations avec ces valeurs aberrantes enlevées et étudier plus
attentivement le résultat.

``` r
outliers <- c("F5D165", "F6D165", "M3D175", "M4D175", "M5D175", "M6D175")
ps <- prune_samples(!(sample_names(ps) %in% outliers), ps)
```

Nous allons également supprimer les échantillons ayant moins de 1000
reads :

``` r
which(!rowSums(otu_table(ps)) > 1000)  
```

    ## F5D145 M1D149   M1D9 M2D125  M2D19 M3D148 M3D149   M3D3   M3D5   M3D8 
    ##     69    185    200    204    218    243    244    252    256    260

``` r
ps <- prune_samples(rowSums(otu_table(ps)) > 1000, ps)
pslog <- transform_sample_counts(ps, function(x) log(1 + x))
```

Nous allons d’abord effectuer une PCoA en utilisant la dissimilarité de
Bray-Curtis.

``` r
out.pcoa.log <- ordinate(pslog,  method = "MDS", distance = "bray")
evals <- out.pcoa.log$values[,1]
plot_ordination(pslog, out.pcoa.log, color = "age_binned",
                  shape = "family_relationship") +
  labs(col = "Binned Age", shape = "Litter")+
  coord_fixed(sqrt(evals[2] / evals[1]))
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-48-1.png)<!-- --> Nous
constatons qu’il existe un effet d’âge assez important qui est cohérent
entre toutes les souris, mâles et femelles, et provenant de différentes
portées.  
Ensuite, nous examinons l’analyse des coordonnées principales doubles
(DPCoA) (Pavoine, Dufour et Chessel 2004 ; Purdom 2010 ; Fukuyama et
al. 2012), qui est une méthode d’ordination phylogénétique et qui
fournit une représentation biplot des échantillons et des catégories
taxonomiques. Nous voyons à nouveau que le deuxième axe correspond aux
jeunes souris par rapport aux vieilles souris, et le biplot suggère une
interprétation du deuxième axe : les échantillons qui ont des scores
plus élevés sur le deuxième axe ont plus de taxa de *Bacteroidetes* et
un sous-ensemble de *Firmicutes*.

``` r
out.dpcoa.log <- ordinate(pslog, method = "DPCoA")
evals <- out.dpcoa.log$eig
plot_ordination(pslog, out.dpcoa.log, color = "age_binned", label= "SampleID",
                  shape = "family_relationship") +
  labs(col = "Binned Age", shape = "Litter")+
  coord_fixed(sqrt(evals[2] / evals[1]))
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-49-1.png)<!-- --> Dans
la figure 11, le premier axe explique 75 % de la variabilité, soit
environ 9 fois celle du deuxième axe ; cela se traduit par la forme
allongée du graphique d’ordination.

``` r
plot_ordination(pslog, out.dpcoa.log, type = "species", color = "Phylum") +
  coord_fixed(sqrt(evals[2] / evals[1]))
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-50-1.png)<!-- --> Enfin,
nous pouvons examiner les résultats de la PCoA avec Unifrac pondéré.
Comme précédemment, nous constatons que le deuxième axe est associé à un
effet d’âge, ce qui est assez similaire au DPCoA. Cela n’est pas
surprenant, car les deux méthodes d’ordination phylogénétique prennent
en compte l’abondance. Cependant, lorsque nous comparons les biplots,
nous constatons que la DPCoA donne une interprétation beaucoup plus
propre du deuxième axe, par rapport à Unifrac pondéré.

``` r
out.wuf.log <- ordinate(pslog, method = "PCoA", distance ="wunifrac")
```

    ## Warning in UniFrac(physeq, weighted = TRUE, ...): Randomly assigning root as --
    ## GCAAGCGTTATCCGGATTTACTGGGTGTAAAGGGAGCGTAGACGGCTTGATAAGTCTGAAGTGAAAGGCCAAGGCTTAACCATGGAACTGCTTTGGAAACTATGAGGCTAGAGTGCTGGAGAGGTAAGCGGAATTCCTAGTGTAGCGGTGAAATGCGTAGATATTAGGAGGAACACCAGTGGCGAAGGCGGCTTACTGGACAGAAACTGACGTTGAGGCTCGAAAGCGTGGGGAG
    ## -- in the phylogenetic tree in the data you provided.

``` r
evals <- out.wuf.log$values$Eigenvalues
plot_ordination(pslog, out.wuf.log, color = "age_binned",
                  shape = "family_relationship") +
  coord_fixed(sqrt(evals[2] / evals[1])) +
  labs(col = "Binned Age", shape = "Litter") 
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-51-1.png)<!-- -->
##Pourquoi les courbes d’ordination sont-elles si éloignées du carré ?  
###Rapport d’aspect des diagrammes d’ordination  
Dans les diagrammes d’ordination de la Figure 8-Figure 14, les cartes ne
sont pas présentées sous forme de représentations carrées, comme c’est
souvent le cas dans les diagrammes PCoA et PCA standard de la
littérature.  
La raison en est que, comme nous essayons de représenter les distances
entre les échantillons aussi fidèlement que possible, nous devons tenir
compte du fait que la deuxième valeur propre est toujours plus petite
que la première, parfois considérablement, et nous normalisons donc les
rapports de la norme de l’axe aux rapports des valeurs propres
pertinentes. Cela permet de garantir que la variabilité représentée dans
les graphiques l’est de manière fidèle.

###PCA on ranks  
Les données sur l’abondance microbienne sont souvent fortement
inclinées, et il est parfois difficile d’identifier une transformation
qui ramène les données à la normalité. Dans ces cas, il peut être plus
sûr d’ignorer complètement les abondances brutes et de travailler plutôt
avec des rangs. Nous démontrons cette idée en utilisant une version des
données transformées en rangs pour effectuer une ACP. Tout d’abord, nous
créons une nouvelle matrice, représentant les abondances par leurs
rangs, où le micro-organisme le plus petit dans un échantillon est
affecté au rang 1, le deuxième plus petit au rang 2, etc.

``` r
abund <- otu_table(pslog)
abund_ranks <- t(apply(abund, 1, rank))
```

L’utilisation naïve de ces rangs pourrait rendre comparables les
différences entre les ensembles de micro-organismes à faible et à forte
abondance. Dans le cas où de nombreuses bactéries sont absentes ou
présentes à l’état de traces, une différence de rang artificiellement
importante pourrait se produire (Holmes et al. 2011) pour les taxa peu
abondants. Pour éviter cela, tous les organismes dont le rang est
inférieur à un certain seuil sont mis à égalité à 1. Les rangs des
autres organismes sont décalés vers le bas, afin qu’il n’y ait pas de
grand écart entre les rangs.

``` r
abund_ranks <- abund_ranks - 329
abund_ranks[abund_ranks < 1] <- 1
```

``` r
library(dplyr)
library(reshape2)
abund_df <- melt(abund, value.name = "abund") %>%
  left_join(melt(abund_ranks, value.name = "rank"))
```

    ## Joining, by = c("Var1", "Var2")

``` r
colnames(abund_df) <- c("sample", "seq", "abund", "rank")  

abund_df <- melt(abund, value.name = "abund") %>%
  left_join(melt(abund_ranks, value.name = "rank"))
```

    ## Joining, by = c("Var1", "Var2")

``` r
colnames(abund_df) <- c("sample", "seq", "abund", "rank")  

sample_ix <- sample(1:nrow(abund_df), 8)
ggplot(abund_df %>%
         filter(sample %in% abund_df$sample[sample_ix])) +
  geom_point(aes(x = abund, y = rank, col = sample),
             position = position_jitter(width = 0.2), size = 1.5) +
  labs(x = "Abundance", y = "Thresholded rank") +
  scale_color_brewer(palette = "Set2")
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-54-1.png)<!-- --> Cette
transformation est illustrée dans la figure 14.  
L’association entre l’abondance et le rang, pour quelques échantillons
choisis au hasard. Les chiffres de l’axe des y sont ceux fournis à
l’ACP.  
Nous pouvons maintenant effectuer l’ACP et étudier le biplot résultant,
donné dans la figure ci-dessous. Pour produire une annotation pour cette
figure, nous avons utilisé le bloc suivant.

``` r
library(ade4)
ranks_pca <- dudi.pca(abund_ranks, scannf = F, nf = 3) 
row_scores <- data.frame(li = ranks_pca$li,
                         SampleID = rownames(abund_ranks))
col_scores <- data.frame(co = ranks_pca$co,
                         seq = colnames(abund_ranks))
tax <- tax_table(ps) %>%
  data.frame(stringsAsFactors = FALSE)
tax$seq <- rownames(tax)
main_orders <- c("Clostridiales", "Bacteroidales", "Lactobacillales",
                 "Coriobacteriales")
tax$Order[!(tax$Order %in% main_orders)] <- "Other"
tax$Order <- factor(tax$Order, levels = c(main_orders, "Other"))
tax$otu_id <- seq_len(ncol(otu_table(ps)))
row_scores <- row_scores %>%
  left_join(sample_data(pslog))
```

    ## Joining, by = "SampleID"

``` r
col_scores <- col_scores %>%
  left_join(tax)
```

    ## Joining, by = "seq"

``` r
evals_prop <- 100 * (ranks_pca$eig / sum(ranks_pca$eig))
ggplot() +
  geom_point(data = row_scores, aes(x = li.Axis1, y = li.Axis2), shape = 2) +
  geom_point(data = col_scores, aes(x = 25 * co.Comp1, y = 25 * co.Comp2, col = Order),
             size = .3, alpha = 0.6) +
  scale_color_brewer(palette = "Set2") +
  facet_grid(~ age_binned) +
  guides(col = guide_legend(override.aes = list(size = 3))) +
  labs(x = sprintf("Axis1 [%s%% variance]", round(evals_prop[1], 2)),
       y = sprintf("Axis2 [%s%% variance]", round(evals_prop[2], 2))) +
  coord_fixed(sqrt(ranks_pca$eig[2] / ranks_pca$eig[1])) +
  theme(panel.border = element_rect(color = "#787878", fill = alpha("white", 0)))
```

![](test_rstudio_files/figure-gfm/unnamed-chunk-56-1.png)<!-- --> Les
résultats sont similaires aux analyses PCoA calculées sans appliquer de
transformation de rang tronqué, ce qui renforce notre confiance dans
l’analyse des données originales.

###Correspondance canonique L’analyse des correspondances canonique
(CCpnA) est une approche de l’ordination d’un tableau d’espèces par
échantillon qui incorpore des informations supplémentaires sur les
échantillons. Comme précédemment, le but de la création de biplots est
de déterminer quels types de communautés bactériennes sont les plus
importants dans les différents types d’échantillons de souris. Il peut
être plus facile d’interpréter ces biplots lorsque l’ordre entre les
échantillons reflète les caractéristiques de l’échantillon - les
variations d’âge ou de statut de la litière dans les données sur les
souris, par exemple - et ceci est au cœur de la conception de CCpnA.  
La fonction nous permet de créer des biplots où les positions des
échantillons sont déterminées par la similarité des signatures d’espèces
et des caractéristiques environnementales ; en revanche, l’analyse en
composantes principales ou l’analyse des correspondances ne
s’intéressent qu’aux signatures d’espèces. Plus formellement, elle
garantit que les directions CCpnA résultantes se situent dans l’étendue
des variables environnementales ; des traitements approfondis sont
disponibles dans (Braak 1985 ; Greenacre 2007). Comme la PCoA et la
DPCoA, cette méthode peut être exécutée à l’aide d’ordinate du paquet
phyloseq. Afin d’utiliser des données d’échantillon supplémentaires, il
est nécessaire de fournir un argument supplémentaire, spécifiant quelles
caractéristiques doivent être prises en compte - sinon, la méthode
utilise par défaut toutes les mesures lors de la production de
l’ordination.

``` r
ps_ccpna <- ordinate(pslog, "CCA", formula = pslog ~ age_binned + family_relationship)
```

Pour accéder aux positions pour le biplot, nous pouvons utiliser la
fonction ordinate dans phyloseq. De plus, pour faciliter l’annotation
des figures, nous joignons également les scores des sites avec les
données environnementales dans le slot. Sur les 23 ordres taxonomiques
totaux, nous n’annotons explicitement que les quatre plus abondants -
cela rend le biplot plus facile à lire.

``` r
library(ggrepel)
ps_scores <- vegan::scores(ps_ccpna)
sites <- data.frame(ps_scores$sites)
sites$SampleID <- rownames(sites)
sites <- sites %>%
  left_join(sample_data(ps))  
```

    ## Joining, by = "SampleID"

``` r
species <- data.frame(ps_scores$species)
species$otu_id <- seq_along(colnames(otu_table(ps)))
species <- species %>%
  left_join(tax)
```

    ## Joining, by = "otu_id"

``` r
evals_prop <- 100 * ps_ccpna$CCA$eig[1:2] / sum(ps_ccpna$CA$eig)
ggplot() +
  geom_point(data = sites, aes(x = CCA1, y = CCA2), shape = 2, alpha = 0.5) +
  geom_point(data = species, aes(x = CCA1, y = CCA2, col = Order), size = 0.5) +
  geom_text_repel(data = species %>% filter(CCA2 < -2),
                    aes(x = CCA1, y = CCA2, label = otu_id),
            size = 1.5, segment.size = 0.1) +
  facet_grid(. ~ family_relationship) +
  guides(col = guide_legend(override.aes = list(size = 3))) +
  labs(x = sprintf("Axis1 [%s%% variance]", round(evals_prop[1], 2)),
        y = sprintf("Axis2 [%s%% variance]", round(evals_prop[2], 2))) +
  scale_color_brewer(palette = "Set2") +
  coord_fixed(sqrt(ps_ccpna$CCA$eig[2] / ps_ccpna$CCA$eig[1])*0.45   ) +
  theme(panel.border = element_rect(color = "#787878", fill = alpha("white", 0)))  
```

    ## Warning: ggrepel: 9 unlabeled data points (too many overlaps). Consider
    ## increasing max.overlaps

    ## Warning: ggrepel: 9 unlabeled data points (too many overlaps). Consider
    ## increasing max.overlaps

![](test_rstudio_files/figure-gfm/unnamed-chunk-58-1.png)<!-- -->
