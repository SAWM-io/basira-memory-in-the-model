# Brancher une mémoire dans le modèle : ce qui est mesuré, ce qui est proposé

*Comment un laboratoire d'IA pourrait relier SAWM & Basira Memory liquide à son modèle, du contexte préparé jusqu'à
l'attention, et ce que lui et ses clients pourraient y gagner.*

[English version](README.md)

## En bref

Trois mots reviennent dans tout l'article. Un **jeton** est un morceau de mot : 1 000 jetons font environ 750 mots.
Un **souvenir** est une information enregistrée par Basira, avec sa date et sa source. Le **rappel** est le moment où
Basira cherche les souvenirs utiles à une question.

- **Les résultats, avec Claude Opus 5.5.** Basira obtient 98,4 % puis 98,6 % sur LongMemEval-S, et 96 % sur LoCoMo :
  ce sont les deux tests publics de référence pour la mémoire à long terme. Les deux pourcentages correspondent à deux
  notations successives par le juge automatique. Opus ne lit que ce que Basira lui rend.
- **Presque le même score avec 3,2 fois moins à lire** (LongMemEval-S). Avec 160 souvenirs, le modèle lit environ
  31 400 jetons au lieu des 101 700 de l'historique complet, et obtient 98,2 % puis 98,4 %.
- **Aucun jeton de modèle de langage pour la mémoire, par défaut.** Écrire et rappeler un souvenir se fait avec un
  petit modèle de vecteurs qui tourne sur processeur.
- **Quatre profondeurs de branchement** : contexte préparé, outil consulté pendant la réflexion, mémoire lue par
  l'attention, mémoire compressée ou apprise.
- **Une mémoire par utilisateur.** Par conception, les rappels de deux utilisateurs ne dépendent pas l'un de l'autre,
  et l'on peut répartir les mémoires sur autant de machines qu'il faut. Notre propre service n'a été mesuré qu'à petite
  échelle (section 7).
- **Le flashback et la mémoire des conséquences (proposés, pas encore mesurés).** Pendant sa réflexion, le modèle
  vérifierait dans sa mémoire ce qui a été dit, quand, et ce qui a changé. Il retrouverait aussi ce qui a suivi ses
  réponses passées, pour mieux prévoir les conséquences de la prochaine.

## 1. Le problème

Un agent qui travaille avec quelqu'un depuis un an accumule des millions de jetons d'historique. Il a trois options :

1. **Tout relire à chaque question.** C'est cher, plus lent, et borné par la fenêtre du modèle (la quantité de texte
   qu'il peut lire d'un coup, environ un million de jetons pour Opus 5.5). Et l'exactitude baisse avec la longueur.
   Dans l'article qui a créé LongMemEval, GPT-4o obtient 60,6 % avec tout l'historique, contre 87,0 % quand on ne lui
   donne que les sessions qui contiennent la réponse, désignées à l'avance par les auteurs. D'autres études montrent
   la même pente : NoLiMa, RULER, Lost in the Middle, Context Rot.
2. **Résumer.** Le résumé perd les détails dont on aura besoin plus tard : une date, un montant, un prénom.
3. **Rechercher des passages (RAG).** Tout dépend alors de la qualité de la recherche, et une recherche ratée ne se
   voit pas : le modèle répond avec assurance, sans le fait qui manquait.

Basira appartient à la troisième famille, mais elle est faite pour la mémoire d'un agent. Elle date chaque souvenir,
garde les anciennes versions d'une information, et élargit sa recherche quand le souvenir utile est ancien.

## 2. Ce que Basira fait aujourd'hui

Ces résultats sont publiés réponse par réponse, avec un script pour les recompter :
[basira-longmemeval](https://github.com/SAWM-io/basira-longmemeval) et
[basira-locomo](https://github.com/SAWM-io/basira-locomo).

| Test | Rappel de Basira | Ce que lit le modèle (médiane) | Exactitude, deux passes du juge |
|---|---|---|---|
| LongMemEval-S, 500 questions, juge officiel GPT-4o | jusqu'à 320 souvenirs | ≈ 62 600 jetons | **98,4 % / 98,6 %** |
| LongMemEval-S | jusqu'à 160 souvenirs | ≈ 31 400 jetons, pour un historique de ≈ 101 700 | **98,2 % / 98,4 %** |
| LoCoMo, 1 540 questions, juge de référence de Mem0 | jusqu'à 320 souvenirs | ≈ 18 400 jetons, pour une conversation de ≈ 21 800 | **95,97 % / 95,84 %** |

Les jetons sont comptés avec le tokenizer de GPT-4o ; l'historique LongMemEval est compté en texte brut (l'article
LongMemEval annonce environ 115 000 jetons avec un autre comptage). Avec le juge strict de LongMemEval, les réponses
LoCoMo obtiennent 92,08 % puis 91,95 %.

Sur LoCoMo, les conversations sont courtes : à 320 souvenirs, le modèle lit encore 84 % de la conversation en jetons.
La réduction de lecture y est donc faible ; elle se joue sur les longs historiques. Pour 95 % des questions dont les
passages de preuve sont annotés (1 455 sur 1 527), Basira rend tous ces passages. Sur un échantillon fixe de 310
questions, Opus obtient 96,13 % en lisant toute la conversation et 94,84 % avec le rappel de 320 souvenirs : l'écart
de 1,3 point n'est pas statistiquement significatif (p = 0,29), ce qui ne prouve pas pour autant l'équivalence.

Sur LongMemEval, le détail par capacité (rappel 320, passe 1) :

| Capacité | Justes |
|---|---|
| Retrouver ce que l'utilisateur a dit | 70/70 |
| Retrouver ce que l'assistant lui-même a dit | 56/56 |
| Suivre une information qui a changé | 77/78 |
| Raisonner sur les dates et l'ordre des événements | 130/133 |
| Rassembler plusieurs conversations | 129/133 |
| Dire qu'une information n'a jamais été donnée | 29/30 |

Quatre propriétés comptent pour un laboratoire :

- **Une mémoire par utilisateur.** Rien ne se mélange entre clients.
- **Chaque souvenir est daté et sourcé.** Les anciennes versions d'une information restent visibles, donc le modèle
  sait laquelle est la plus récente.
- **Un souvenir s'efface à la demande**, ce qui répond au droit à l'effacement du RGPD, sans réentraîner quoi que ce
  soit.
- **Aucun modèle de langage dans l'écriture ni le rappel, par défaut.** Un petit modèle de vecteurs sur processeur
  suffit. Seule une extraction facultative de relations en utilise un ; elle est éteinte par défaut et facturée à part.

Depuis le 7 novembre 2026, l'API Basira propose aussi un **rappel adaptatif** : sa profondeur s'ajuste à la question,
de 20 à 320 souvenirs. Sur LongMemEval, le contexte médian tombe à environ 26 700 jetons. Son score n'est pas encore
jugé ; nous le publierons dès qu'il le sera.

## 3. Quatre façons de brancher la mémoire

De l'extérieur du modèle vers l'intérieur :

| Niveau | Où vit la mémoire | Ce que le modèle voit | Ce qu'il faut | État |
|---|---|---|---|---|
| **1. Contexte préparé** | à côté du modèle | les souvenirs choisis, en texte, avant la question | rien | en production ; c'est ce que mesurent nos scores |
| **2. Outil pendant la réflexion** | à côté du modèle | les réponses de la mémoire au milieu de son raisonnement | des appels d'outils pendant la réflexion, qui existent déjà | possible aujourd'hui ; pas encore mesuré par nous |
| **3. Mémoire lue par l'attention** | dans le moteur d'inférence | les états internes, calculés à l'avance, des souvenirs choisis | l'accès au moteur d'inférence ; avec ou sans entraînement selon la technique | proposé |
| **4. Mémoire compressée ou apprise** | dans quelques vecteurs, ou dans les poids | des souvenirs résumés en vecteurs, ou une mémoire fondue dans le modèle | un pont à entraîner, ou un réentraînement | déconseillé pour la mémoire personnelle |

### Niveau 1 : contexte préparé

C'est ce qui produit nos scores. Avant chaque réponse, Basira choisit les souvenirs utiles et les donne au modèle
avec leurs dates. Le modèle reste en bout de chaîne : il lit et tranche. Ce niveau marche avec n'importe quel modèle,
et changer de modèle ne coûte rien.

### Niveau 2 : la mémoire consultée pendant la réflexion

Les modèles de raisonnement d'Anthropic et d'OpenAI savent déjà appeler des outils au milieu de leur réflexion. La
mémoire devient alors une source que le modèle interroge autant de fois que nécessaire, avant d'avoir fini de
penser : une première question pour situer, une deuxième pour vérifier une date, une troisième pour savoir si
l'information a changé. Ce niveau existe déjà par un connecteur standard (MCP). C'est la base du flashback décrit en
section 4. Chaque consultation ajoute un délai, environ 3 s aujourd'hui pour un rappel profond.

### Niveau 3 : la mémoire lue par l'attention

C'est le branchement le plus profond. Quand un modèle lit un texte, ses couches d'attention calculent pour chaque
jeton un état interne, gardé dans la mémoire du GPU : le cache KV (des clés et des valeurs). L'idée :

1. **À l'écriture**, le moteur d'inférence calcule l'état interne de chaque souvenir et le range avec lui.
2. **À la question**, Basira choisit les souvenirs autorisés et utiles : ceux de cet utilisateur, encore valables,
   non effacés.
3. **Le modèle reçoit leurs états internes** au lieu de relire leur texte. Deux familles de techniques publiées le
   permettent. CacheBlend fusionne dans l'attention des états précalculés pour des passages retrouvés, sans
   entraînement. Memorizing Transformers et LongMem laissent le modèle chercher lui-même, dans quelques couches, les
   plus proches voisins parmi des paires de clés et de valeurs mémorisées ; ils demandent un entraînement.

**Ce qu'on y gagnerait.** Du calcul : le modèle ne refait plus toute la lecture des souvenirs à chaque question. Le
cache de prompt classique aide peu ici, puisque les souvenirs rappelés changent d'une question à l'autre, alors qu'un
état rangé avec son souvenir se réutilise d'une question à l'autre.

**Ce que ça ne règle pas.** Ces états ont été calculés sans voir la question ni les autres souvenirs, avec des
positions fixées d'avance. Il faut corriger les positions et recalculer une petite partie des jetons (CacheBlend en
recalcule environ 15 % selon ses auteurs). L'attention entre souvenirs n'est qu'approchée, ce qui peut peser sur les
dates et sur les questions qui croisent plusieurs conversations. La qualité face à la relecture du texte n'est pas
mesurée. Une fois chargés, les états occupent la mémoire du GPU comme le texte l'aurait fait : on économise du calcul,
pas de place pendant la réponse.

**Ce que ça coûte.** De la place de stockage. Sur un modèle public de 70 milliards de paramètres sans compression du
cache, un jeton pèse environ 320 Ko en 16 bits (2 pour les clés et les valeurs × 80 couches × 8 têtes × 128
dimensions × 2 octets). L'historique de 101 700 jetons d'un seul utilisateur pèserait donc environ 33 Go. Il faut les
garder sur disque ou en mémoire rapide, et ne charger que les souvenirs rappelés (environ 10 Go pour 160 souvenirs).
Ce chargement peut coûter autant de temps que relire le texte : c'est à mesurer. Les modèles conçus avec un cache plus
compact réduisent fortement ces chiffres : moins de têtes de clés et de valeurs (GQA), représentations latentes (MLA ;
DeepSeek-V2 réduit son cache de 93,3 % par rapport à DeepSeek 67B). Ces choix se font à l'entraînement du modèle. Enfin,
les états sont propres à une version précise du modèle : changer de modèle, mettre à jour ses poids ou changer sa
quantification oblige à tout recalculer.

**Ce qu'on garderait.** Les états seraient rangés souvenir par souvenir, sous le contrôle de Basira. L'isolement par
utilisateur, l'effacement et la source de chaque souvenir pourraient donc être conservés, à condition que le
laboratoire efface aussi l'état interne d'un souvenir quand le souvenir est effacé.

C'est pourquoi le texte des souvenirs reste la source de vérité, et leurs états un accélérateur.

### Niveau 4 : une mémoire compressée ou apprise

Deux familles de travaux vont encore plus loin.

- **Compresser chaque souvenir en quelques vecteurs** que le modèle lit à la place du texte : un seul jeton par
  document pour xRAG, des jetons résumés pour ICAE ou les gist tokens. Seul un petit pont est entraîné, le modèle
  reste tel quel. Mais la compression perd les détails exacts (dates, chiffres, prénoms), justement là où se jouent
  les tests de mémoire.
- **Fondre la mémoire dans les poids** : couches de mémoire à clés (Memory Layers, Product Keys), mémoire apprise
  pendant l'usage (Titans, MemoryLLM). C'est prometteur pour la connaissance générale. Pour la mémoire d'une
  personne, on y perd ce qui compte : effacer un fait précis de façon sûre, séparer deux utilisateurs, dire d'où vient
  une réponse.

**Notre recommandation** : garder le niveau 1 comme base, et y ajouter les niveaux 2 et 3. Basira garde la vérité,
avec ses preuves. Le modèle la consulte pendant sa réflexion et, quand c'est mesuré comme utile, lit les souvenirs
choisis par leurs états internes.

## 4. Le flashback : vérifier le passé pendant la réflexion (proposé)

Avant de répondre à une question importante, un humain revient sur ce qu'il sait : qu'est-ce qui a été dit, quand, et
est-ce encore vrai ? Le flashback donnerait ce réflexe au modèle :

1. **Le modèle commence à raisonner** et repère ce qui, dans sa réponse, dépend du passé.
2. **Il interroge la mémoire.** Par exemple : « Qu'a dit l'utilisateur sur son budget, et quand ? Est-ce encore
   valable ? »
3. **Basira répond avec des preuves datées.** Elle rend l'information la plus récente et ses anciennes versions, ou
   rien du tout si l'information n'a jamais été donnée.
4. **Le modèle confronte son brouillon aux preuves.** Il corrige, précise, ou dit qu'il ne sait pas.

Dans ce schéma, le modèle ne reconstruit plus le passé de tête : il répond à partir de pièces datées. Ce que nous
mesurons déjà, avec la mémoire lue avant la réponse (niveau 1), va dans ce sens : 77 sur 78 pour une information qui a
changé, 29 sur 30 pour dire qu'une information n'a jamais été donnée. Le flashback lui-même, avec plusieurs
consultations pendant la réflexion, n'est pas encore mesuré : chaque consultation ajoute du délai, et nous ne savons
pas encore de combien elle améliore les réponses.

La recherche va dans le même sens. Faire vérifier au modèle ses propres affirmations avant de répondre réduit ses
erreurs : sur des listes de faits, la part de réponses correctes d'un modèle de 65 milliards de paramètres passe de
0,17 à 0,36 (Chain-of-Verification, une tâche différente de la nôtre). Mais il vérifie alors avec ce qu'il sait déjà,
et ses auteurs notent que ce savoir fixe le plafond. Une mémoire lui apporte justement les preuves qu'il n'a pas. Et
lancer une recherche au milieu du raisonnement, quand le modèle doute, aide sur plusieurs tâches difficiles
(Search-o1, avec une recherche web).

Le flashback sert aussi à mieux répondre pour la suite. Une bonne réponse tient compte de ce qui est déjà annoncé :
une date limite, un rendez-vous déplacé, une préférence qui a changé. En revérifiant le passé, le modèle peut en tenir
compte et proposer une réponse qui a plus de chances de rester juste. Il ne devine pas ce qui n'a jamais été dit.

## 5. Les conséquences : apprendre de ce qui a suivi (proposé)

Une mémoire peut garder plus que ce qui a été dit. Elle peut garder **ce qui s'est passé ensuite** : la réponse
donnée, l'action faite, la réaction de l'utilisateur, le résultat. Avant de répondre, le modèle retrouverait des
situations proches et leur suite. Par exemple : « La dernière fois que j'ai proposé ce fournisseur, l'utilisateur a
signalé un retard de livraison. »

Il pourrait ainsi s'appuyer sur des cas réels pour prévoir les conséquences de sa réponse, et ajuster sa proposition
avant de la faire. Ce sont des analogies, pas des causes : un cas isolé peut avoir d'autres raisons, et le modèle ne
doit pas s'y accrocher à tort. Plusieurs travaux montrent que l'idée porte, chacun sur une partie :

- garder par écrit ses échecs et les retours reçus fait progresser un agent d'essai en essai (Reflexion) ;
- retrouver des expériences réussies proches avant d'agir l'améliore (ExpeL, Agent Workflow Memory) ;
- simuler l'état suivant avant de choisir une action aide à planifier (RAP).

Nous n'avons pas trouvé de travail qui relie ces trois idées à une mémoire datée et propre à chaque utilisateur. Si
vous en connaissez un, nous le citerons.

Basira retrouve déjà ce que l'assistant a dit lui-même : 56 sur 56 sur LongMemEval. Relier chaque réponse à ses
conséquences, puis mesurer ce que cela change sur la réponse suivante, est notre prochaine étape.

## 6. Ce que le laboratoire et ses clients pourraient y gagner

- **Une exactitude élevée sur de longs historiques.** 98 % sur LongMemEval avec Opus 5.5. Les scores publiés d'un
  modèle qui relit tout l'historique sont de 60 à 64 % pour GPT-4o et de 76 % pour o3, avec d'autres juges et d'autres
  consignes : ils ne se comparent pas directement aux nôtres. Nous n'avons pas mesuré Opus 5.5 avec l'historique
  complet, donc nous ne disons pas que la mémoire est plus juste que la relecture avec ce modèle. Nos réglages ont été
  établis sur LongMemEval ; LoCoMo (96 %) est le test sur lequel ils n'avaient pas été conçus.
- **De la capacité GPU.** Le cache KV d'une conversation grandit avec la longueur du contexte, et c'est lui qui limite
  le nombre de conversations servies en même temps par GPU. Avec 3,2 fois moins de jetons à lire, la part du cache due
  au contexte est 3,2 fois plus petite pendant le traitement (un peu moins au total, car la consigne, la réflexion et la
  réponse s'y ajoutent). Sur le modèle public de 70 milliards de paramètres de la section 3, le contexte passe d'environ
  33 Go à 10 Go. C'est un ordre de grandeur sur le papier, non mesuré sur un moteur de service et non transposable à
  Opus. Moins de jetons, c'est aussi moins de revenu par appel : l'argument vaut pour un fournisseur dont le calcul est
  le facteur limitant.
- **Moins cher quand le cache a expiré ou que l'historique est très long.** Au tarif d'Opus 5.5 (4 $ le million de
  jetons d'entrée), le coût d'entrée d'une question est d'environ 0,41 $ en relisant tout l'historique sans cache,
  et d'environ 0,13 $ avec Basira. Mais avec un cache de prompt chaud, relire l'historique ne coûte qu'environ 0,02 $.
  Basira est moins cher pour des conversations espacées (le cache, valable 5 minutes par défaut, a expiré), pour des
  sessions de 4 questions au plus après un démarrage à froid, et, même contre un cache chaud, pour des historiques de
  plus d'environ 535 000 jetons. Ces coûts ne comptent que les jetons d'entrée.
- **Au-delà de la fenêtre.** Au-delà d'un million de jetons, l'historique complet ne rentre plus dans le modèle. La
  mémoire peut encore choisir des souvenirs ; nous n'avons pas encore mesuré sa justesse sur des historiques plus longs
  que ceux de LongMemEval-S.
- **La confiance.** Chaque réponse s'appuie sur des souvenirs datés et sourcés, et un souvenir s'efface à la demande.

## 7. Des rappels en parallèle, par conception

La mémoire de chaque utilisateur est séparée des autres, et un rappel ne lit que la mémoire de l'utilisateur
concerné. Le travail d'un rappel ne dépend donc pas de celui des autres utilisateurs, et l'on augmente la capacité en
ajoutant des machines et en répartissant les mémoires par utilisateur. Par conception, rien ne fixe de limite au
nombre de rappels en parallèle. Au niveau 3, les états des souvenirs vivraient à côté de l'inférence, sur les machines
du laboratoire.

Ce n'est pas encore démontré à grande échelle. Notre service a été mesuré jusqu'à 4 rappels profonds simultanés, avec
environ 3,4 s en médiane par rappel ; il partage encore des ressources communes (encodeur, stockage), et notre moteur
natif n'est pas encore entièrement parallèle. Nous y travaillons, et nous publierons un banc de charge avec plusieurs
comptes et une montée progressive.

## 8. Ce qui est mesuré, ce qui est proposé

| Affirmation | Statut |
|---|---|
| 98,4–98,6 % sur LongMemEval-S, 98,2–98,4 % avec 160 souvenirs (juge officiel GPT-4o, Opus 5.5) | mesuré, publié réponse par réponse |
| 96 % sur LoCoMo (juge de référence de Mem0 ; 92 % avec le juge strict de LongMemEval) | mesuré, publié réponse par réponse |
| 3,2 fois moins de jetons lus que l'historique complet, pour 98,2 % de bonnes réponses | mesuré sur LongMemEval-S ; score d'Opus 5.5 avec l'historique complet non mesuré |
| Aucun jeton de modèle de langage à l'écriture ni au rappel | configuration par défaut ; vérifié dans notre code |
| Rappel adaptatif (20 à 320 souvenirs) | en production ; score pas encore jugé |
| Niveau 2 (mémoire consultée pendant la réflexion) | possible aujourd'hui ; pas encore mesuré par nous |
| Niveau 3 (mémoire lue par l'attention) | proposé ; s'appuie sur des techniques publiées ; qualité et coût à mesurer |
| Flashback (vérification du passé pendant la réflexion) | proposé ; pas encore mesuré |
| Mémoire des conséquences | proposé ; pas encore mesuré |
| Rappels en parallèle | mesuré jusqu'à 4 rappels profonds simultanés ; au-delà, banc de charge à venir |
| Gain de capacité GPU | calcul sur un modèle public ; non mesuré sur un moteur de service |

## 9. Références

**Tests de mémoire et longueur du contexte**
- Wu et al., *LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory*, ICLR 2025. [arXiv:2410.10813](https://arxiv.org/abs/2410.10813)
- Maharana et al., *Evaluating Very Long-Term Conversational Memory of LLM Agents* (LoCoMo), ACL 2024. [arXiv:2402.17753](https://arxiv.org/abs/2402.17753)
- Modarressi et al., *NoLiMa: Long-Context Evaluation Beyond Literal Matching*, 2025. [arXiv:2502.05167](https://arxiv.org/abs/2502.05167)
- Hsieh et al., *RULER: What's the Real Context Size of Your Long-Context Language Models?*, 2024. [arXiv:2404.06654](https://arxiv.org/abs/2404.06654)
- Liu et al., *Lost in the Middle: How Language Models Use Long Contexts*, 2023. [arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- Chroma, *Context Rot*, 2025. [trychroma.com/research/context-rot](https://www.trychroma.com/research/context-rot)
- Scores d'un modèle qui relit tout l'historique sur LongMemEval-S : [Emergence](https://www.emergence.ai/blog/sota-on-longmemeval-with-rag) (GPT-4o 63,8 %, o3 76,0 %), [Mastra](https://mastra.ai/research/observational-memory) (GPT-4o 60,2 %).

**Mémoire dans l'attention ou dans les poids**
- Yao et al., *CacheBlend: Fast Large Language Model Serving for RAG with Cached Knowledge Fusion*, 2024. [arXiv:2405.16444](https://arxiv.org/abs/2405.16444)
- Wu, Rabe, Hutchins, Szegedy, *Memorizing Transformers*, ICLR 2022. [arXiv:2203.08913](https://arxiv.org/abs/2203.08913)
- Wang et al., *Augmenting Language Models with Long-Term Memory* (LongMem), 2023. [arXiv:2306.07174](https://arxiv.org/abs/2306.07174)
- Khandelwal et al., *Generalization through Memorization: Nearest Neighbor Language Models*, ICLR 2020. [arXiv:1911.00172](https://arxiv.org/abs/1911.00172)
- Borgeaud et al., *Improving language models by retrieving from trillions of tokens* (RETRO), ICML 2022. [arXiv:2112.04426](https://arxiv.org/abs/2112.04426)
- Cheng et al., *xRAG: Extreme Context Compression for Retrieval-augmented Generation with One Token*, NeurIPS 2024. [arXiv:2405.13792](https://arxiv.org/abs/2405.13792)
- Ge et al., *In-context Autoencoder for Context Compression in a Large Language Model*, ICLR 2024. [arXiv:2307.06945](https://arxiv.org/abs/2307.06945)
- Mu, Li, Goodman, *Learning to Compress Prompts with Gist Tokens*, NeurIPS 2023. [arXiv:2304.08467](https://arxiv.org/abs/2304.08467)
- Berges et al., *Memory Layers at Scale*, 2024. [arXiv:2412.09764](https://arxiv.org/abs/2412.09764)
- Lample et al., *Large Memory Layers with Product Keys*, NeurIPS 2019. [arXiv:1907.05242](https://arxiv.org/abs/1907.05242)
- Behrouz, Zhong, Mirrokni, *Titans: Learning to Memorize at Test Time*, fin 2024. [arXiv:2501.00663](https://arxiv.org/abs/2501.00663)
- Wang et al., *MEMORYLLM: Towards Self-Updatable Large Language Models*, ICML 2024. [arXiv:2402.04624](https://arxiv.org/abs/2402.04624)

**Taille du cache KV**
- Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention* (vLLM), SOSP 2023. [arXiv:2309.06180](https://arxiv.org/abs/2309.06180)
- Ainslie et al., *GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints*, EMNLP 2023. [arXiv:2305.13245](https://arxiv.org/abs/2305.13245)
- DeepSeek-AI, *DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model*, 2024. [arXiv:2405.04434](https://arxiv.org/abs/2405.04434)

**Mémoire pendant la réflexion, vérification, expérience**
- Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*, ICLR 2023. [arXiv:2210.03629](https://arxiv.org/abs/2210.03629)
- Asai et al., *Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection*, ICLR 2024. [arXiv:2310.11511](https://arxiv.org/abs/2310.11511)
- Dhuliawala et al., *Chain-of-Verification Reduces Hallucination in Large Language Models*, Findings ACL 2024. [arXiv:2309.11495](https://arxiv.org/abs/2309.11495)
- Li et al., *Search-o1: Agentic Search-Enhanced Large Reasoning Models*, 2025. [arXiv:2501.05366](https://arxiv.org/abs/2501.05366)
- Shinn et al., *Reflexion: Language Agents with Verbal Reinforcement Learning*, NeurIPS 2023. [arXiv:2303.11366](https://arxiv.org/abs/2303.11366)
- Zhao et al., *ExpeL: LLM Agents Are Experiential Learners*, AAAI 2024. [arXiv:2308.10144](https://arxiv.org/abs/2308.10144)
- Wang et al., *Agent Workflow Memory*, 2024. [arXiv:2409.07429](https://arxiv.org/abs/2409.07429)
- Hao et al., *Reasoning with Language Model is Planning with World Model* (RAP), EMNLP 2023. [arXiv:2305.14992](https://arxiv.org/abs/2305.14992)
- Park et al., *Generative Agents: Interactive Simulacra of Human Behavior*, UIST 2023. [arXiv:2304.03442](https://arxiv.org/abs/2304.03442)
- Packer et al., *MemGPT: Towards LLMs as Operating Systems*, 2023. [arXiv:2310.08560](https://arxiv.org/abs/2310.08560)
- Anthropic, documentation de la réflexion entre les appels d'outils (*extended thinking*). [platform.claude.com](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)
- OpenAI, guide d'appel d'outils des modèles o3 et o4-mini. [developers.openai.com](https://developers.openai.com/cookbook/examples/o-series/o3o4-mini_prompting_guide)

---

[SAWM](https://sawm.io) · SAWM & Basira Memory liquide, la mémoire des agents IA. Pour en parler :
[sawm.io](https://sawm.io).
