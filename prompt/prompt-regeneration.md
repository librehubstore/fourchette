# Prompt de régénération — application de suivi alimentaire et d'activité physique

Ce document décrit, sans aucune référence à un dépôt de code, une application complète.
Il doit suffire à en reconstruire une version fonctionnellement équivalente.

---

## 1. INTENTION

L'application permet à un particulier de suivre ce qu'il mange et ce qu'il dépense, pour
rester dans un budget calorique qu'il ne calcule pas lui-même. Elle scanne le code-barres
d'un produit, en récupère les valeurs nutritionnelles dans une base ouverte de produits
alimentaires, les proratise à la quantité consommée et les confronte à un objectif déduit
du profil (sexe, âge, taille, poids, niveau d'activité, but). Elle s'adresse à une personne
seule ou à un couple qui mange souvent la même chose, sur téléphone, debout dans une cuisine.

Elle remplace les applications de suivi nutritionnel par abonnement — de type MyFitnessPal,
Yazio ou Lifesum — dont les fonctions essentielles (scan, journal, objectifs, tendances)
sont payantes ou noyées dans la publicité. Le problème résolu est double : saisir un repas
doit prendre quelques secondes, et le chiffre affiché doit rester juste dans le temps, sans
que l'historique se réécrive à chaque pesée.

Deux partis pris la distinguent : un **parcours d'urgence anti-fringale**, ouvert d'un
bouton central quand l'envie arrive, et un **suivi de poids théorique** confronté à la
balance, qui dit quel réglage du profil est faux plutôt que de reprocher un écart.

---

## 2. PÉRIMÈTRE FONCTIONNEL

### 2.1 Compte et accueil

- **Créer un compte.** L'utilisateur donne une adresse e-mail, un mot de passe et un nom
  affiché. Le système crée le compte, ouvre la session et le conduit immédiatement au
  questionnaire d'accueil. Une adresse déjà utilisée est refusée avec un message explicite.
- **Se connecter.** L'utilisateur donne adresse et mot de passe. Le système ouvre la
  session. Une adresse inconnue et un mot de passe faux produisent exactement le même
  message et le même délai de réponse.
- **Rester connecté.** La session se prolonge d'elle-même tant que l'utilisateur revient
  régulièrement. Une coupure réseau ne déconnecte jamais : l'application affiche un
  bandeau « hors ligne », réessaie seule, et reprend là où elle en était.
- **Se déconnecter.** La session est close sur le serveur et localement. L'action aboutit
  même si le serveur ne répond pas.
- **Questionnaire d'accueil, en quatre écrans** : qui êtes-vous (nom, sexe biologique,
  âge), vos mesures (taille, poids), votre rythme (niveau d'activité, jour et heure de
  pesée hebdomadaire), vos objectifs (but, poids visé facultatif). Le dernier écran affiche
  les objectifs caloriques et de macronutriments que ce profil produirait, **avant**
  validation. Tant que le questionnaire n'est pas terminé, toute autre page renvoie vers
  lui : un profil non renseigné produirait des chiffres faux.
- **Renvoyer le questionnaire une seconde fois** (double clic, retour arrière) ne change
  rien et ne produit pas d'erreur.

### 2.2 Journal alimentaire

- **Voir sa journée.** L'utilisateur ouvre le journal. Le système affiche, pour la journée
  choisie : un anneau de progression calorique (consommé / budget du jour), trois barres de
  macronutriments (protéines, glucides, lipides) avec valeur et pourcentage de l'objectif,
  puis quatre cartes de repas — petit-déjeuner, déjeuner, dîner, collation — chacune avec
  ses lignes et son total.
- **Changer de journée.** Flèches précédent / suivant, ou sélecteur de date. Le futur est
  interdit : on ne consigne que le passé et le jour même.
- **Ajouter un aliment à un repas.** Trois chemins : scanner un code-barres, chercher par
  nom, ou reprendre un aliment déjà consommé. L'utilisateur choisit le produit, ajuste la
  quantité en grammes (raccourcis 20/30/50/100/150/200 et saisie libre), valide. Le système
  ajoute la ligne et recalcule immédiatement les totaux du repas et du jour.
- **Scanner un code-barres.** L'utilisateur ouvre la caméra, vise l'emballage. Le système
  reconnaît le code, interroge la base de produits et affiche la fiche (nom, marque, photo,
  valeurs pour 100 g, Nutri-Score, degré de transformation). Si la caméra est indisponible
  ou le code illisible, une saisie manuelle du numéro prend le relais. Si le produit est
  inconnu de la base, le système le dit et propose de le créer à la main.
- **Chercher un aliment par nom.** L'utilisateur tape au moins deux caractères. Le système
  renvoie ses propres aliments personnalisés d'abord, puis les produits de la base ouverte,
  classés par proximité avec la requête (voir § 4.9) — l'aliment brut avant la préparation
  qui le contient.
- **Créer un aliment personnalisé.** Pour une préparation maison : nom, marque facultative,
  quantité de référence facultative, et les valeurs pour 100 g. L'aliment reste privé, et
  reste proposé à la recherche du créateur.
- **Corriger une ligne.** L'utilisateur modifie la quantité, ou change le repas de
  rattachement. Le système met la ligne à jour et recalcule.
- **Déplacer une ligne d'un repas à l'autre** par glisser-déposer, à la poignée en début de
  ligne. L'écran de modification propose la même bascule : le geste est un raccourci, jamais
  le seul chemin.
- **Supprimer une ligne.** Elle disparaît, les totaux suivent.
- **Reprendre.** Un seul bouton par repas, trois onglets :
  - *Fréquents* : aliments les plus souvent consommés à ce repas sur les trois derniers
    mois, avec le nombre de fois et la dernière quantité, qui pré-remplit le formulaire.
  - *Récents* : les mêmes, classés par date de dernière saisie.
  - *Journées* : les journées passées (un mois) où ce repas a été renseigné, avec le nombre
    de lignes, les calories et les premiers noms. L'utilisateur choisit une journée : tous
    ses aliments arrivent **cochés**, il décoche ceux qu'il ne reprend pas, et choisit le
    repas de destination (souvent le même, pas toujours — les restes du déjeuner finissent
    au dîner). La copie s'ajoute à l'existant, ne l'écrase jamais, et s'annule d'un geste
    depuis le message de confirmation.

### 2.3 Activité physique

- **Consulter le catalogue.** 36 activités réparties en sept familles (marche, course,
  vélo, salle, sports, eau, quotidien), chacune avec une intensité de référence.
- **Enregistrer une séance.** L'utilisateur choisit une activité, saisit une durée en
  minutes, et éventuellement une dépense relevée (montre, appareil de salle) et une note.
  Le système calcule la dépense à partir de l'intensité et du poids actuel, ou retient la
  valeur relevée si elle est fournie. Il affiche une estimation en temps réel pendant la
  saisie.
- **Déclarer une activité hors catalogue** en fournissant soit une intensité, soit une
  dépense en calories.
- **Voir les séances du jour**, avec durée et dépense, et le total.
- **Corriger une séance** : durée, dépense, note. Changer la durée recalcule la dépense —
  sauf si elle avait été relevée à la main, auquel cas elle est préservée.
- **Supprimer une séance.**

### 2.4 Poids

- **Enregistrer une pesée** : une date, un poids, une note facultative. Une seconde pesée
  le même jour corrige la première au lieu de s'y ajouter.
- **Voir sa courbe de poids**, avec l'écart entre pesées successives et la variation totale.
- **Supprimer une pesée** erronée.
- Le poids de départ est enregistré automatiquement à la fin du questionnaire d'accueil :
  la courbe n'attend pas une seconde saisie pour exister.
- Passé la première pesée, le poids ne se modifie plus depuis le profil : il suit la balance.
- **Rappel hebdomadaire de pesée** : jour et heure choisis par l'utilisateur, notification
  sur ses appareils. Le rappel ne part pas s'il s'est déjà pesé dans les sept jours.

### 2.5 Tendances et bilans

- **Choisir une période** : 7, 14 ou 30 jours.
- **Lire un graphique journalier** : apports en colonnes empilées par macronutriment,
  ligne d'objectif, et dépenses tracées **sous l'axe**, en miroir, pour leur seule part
  recréditée au budget. La dépense brute reste lisible au survol.
- **Réordonner les colonnes empilées** en cliquant sur une légende de macronutriment :
  celui qui est sélectionné passe au pied de la colonne, ce qui rend sa variation lisible.
- **Lire les mêmes données sous forme de tableau**, dans un bloc dépliable — alternative
  textuelle au graphique, accessible aux lecteurs d'écran.
- **Voir les moyennes de la période** : apports, dépense brute, dépense recréditée, bilan
  net, et nombre de journées effectivement renseignées.
- **Voir un calendrier mensuel** où chaque journée porte une pastille : vide (rien saisi),
  verte (bilan net sous l'objectif du jour ou jusqu'à 5 % au-dessus), orange (jusqu'à 20 %
  au-dessus), rouge (au-delà). Rester **sous** l'objectif reste vert : pour une perte de
  poids, ce n'est pas un écart.
- **Comparer deux périodes** (la période affichée face à la précédente).
- **Voir le poids théorique** : seconde courbe superposée aux pesées, partant de la
  première pesée de l'année écoulée. Chaque journée renseignée pèse sur le lendemain.
- **Voir l'analyse des écarts** : l'application confronte la balance au poids théorique et,
  si l'écart dépasse le bruit d'une pesée, annonce si la perte est plus lente ou plus rapide
  que prévu, chiffre l'écart quotidien en calories, et propose jusqu'à trois réglages
  concurrents qui le refermeraient (voir § 4.14).
- **Lire une lecture hebdomadaire rédigée** : trois à cinq observations qui croisent les
  dimensions (écart semaine / week-end, macronutriment qui s'effondre les jours de séance,
  repas qui concentre la journée, aliment qui pèse une part démesurée), chacune avec les
  chiffres qui la fondent, plus une action unique à tenir la semaine suivante. Rédigée
  automatiquement une fois par semaine, ou à la demande (au plus une fois par heure).
  Cette fonction peut être absente d'une installation : l'écran le dit alors clairement au
  lieu de proposer un bouton sans effet.

### 2.6 Alerte fringale

Bouton central de la barre de navigation, en saillie et aux couleurs d'alerte. Parcours en
trois temps, conçu pour être suivi debout devant un placard, avec une porte de sortie à
chaque écran.

1. **Souffler.** Deux ou trois arguments tirés des derniers jours de suivi : budget
   restant, séance récente et ce qu'elle a brûlé, avance calorique de la semaine, série de
   journées tenues, poids en baisse, régularité du suivi. Choisis parmi dix règles selon ce
   que disent réellement les données, du plus fort au plus faible. Un compte neuf reçoit un
   argument de repli plutôt qu'une page vide.
2. **Chiffrer.** L'utilisateur cherche ou scanne ce qui lui fait envie, ajuste la portion
   (50 g par défaut — un craquage se mesure en carrés de chocolat, pas en portions de
   repas). Le système annonce ce que ça coûte, ce qu'il resterait au budget du jour, et
   l'équivalent en minutes de marche rapide, vélo soutenu, course et natation.
3. **Décider.** Deux issues :
   - *Craquer* : l'aliment part en collation, sans détour, sans jugement. Ou bien, au
     dernier moment, « finalement je ne l'ai pas mangé ».
   - *Jouer* : une roue à huit parts, un seul tour. Une part accorde tout, deux en accordent
     la moitié, cinq imposent un détour : vingt minutes d'attente, dix minutes de marche
     quel que soit le temps, trente squats, un brossage de dents, une chanson dansée en
     entier, une minute de gainage. Chaque détour dure assez pour que l'envie retombe —
     c'est le vrai mécanisme, le jeu n'est que l'emballage. **Aucun résultat n'interdit
     quoi que ce soit** : au terme du détour, c'est l'utilisateur qui tranche.

Fermer l'application à n'importe quel moment n'inscrit rien au journal.

### 2.7 Proches et partage

- **Inviter un proche** par l'adresse e-mail d'un compte existant. Aucun e-mail n'est
  envoyé : l'invitation attend dans l'application du destinataire, signalée par une
  notification s'il les a activées.
- **Accepter, refuser, annuler, retirer** : un seul geste côté destinataire ou demandeur,
  qui supprime le lien. Retirer un proche emporte les envois en attente entre les deux.
- **Envoyer tout ou partie d'un repas** à un proche depuis la carte du repas. L'envoi
  attend dans la boîte de réception du destinataire.
- **Recevoir un envoi.** Le destinataire voit l'expéditeur, le jour et le repas d'origine,
  et chaque aliment avec la portion de l'expéditeur. Il coche ce qu'il garde, applique un
  facteur global (50 %, 75 %, 100 %, 125 %) ou ajuste au gramme ligne à ligne, choisit son
  jour et son repas, et valide — ou refuse. **Rien n'entre dans un journal sans l'accord de
  son titulaire.**
- **Ouvrir ses repas à un proche.** Réglage distinct du lien, propre à chaque côté,
  **fermé par défaut**. Une fois ouvert, le proche peut aller chercher lui-même dans les
  repas de l'autre, depuis l'onglet « Journées » de « Reprendre », et ajuster ses portions.
- Les envois sans réponse disparaissent au bout de quatorze jours.

### 2.8 Services d'activité externes

- **Connecter un service de santé ou de sport** depuis le profil : l'utilisateur est renvoyé
  chez le fournisseur, autorise l'accès, revient dans l'application avec un message de
  réussite, de refus ou d'erreur. Deux fournisseurs sont pris en charge : un service de
  données de santé généraliste et un service de suivi sportif.
- **Import automatique** toutes les deux heures, et à la demande depuis le profil. La
  dépense importée est une **mesure** : elle n'est jamais recalculée.
- **Voir l'état de chaque connexion** : compte lié, date de dernière synchronisation,
  nombre de séances importées, et le cas échéant la dernière erreur — une connexion
  silencieusement cassée est pire qu'une absente.
- **Déconnecter** : la connexion locale est supprimée, l'autorisation révoquée chez le
  fournisseur si possible.
- Une séance présente dans les deux services n'est comptée qu'une fois. Une séance importée
  puis supprimée à la main ne revient pas au cycle suivant.

### 2.9 Notifications et installation

- **Activer les notifications** sur un appareil. L'application indique combien d'appareils
  sont abonnés. Si le serveur n'a pas de clés de notification, l'option est masquée plutôt
  que proposée sans effet.
- **Rappel de pesée** hebdomadaire (§ 2.4).
- **Messages de motivation**, deux fois par semaine : le lendemain de la pesée et au milieu
  du cycle — mardi et vendredi pour une pesée le lundi. Treize règles lisent les deux
  dernières semaines et chacune porte trois ou quatre formulations. Ni la même règle ni la
  même phrase deux fois de suite. Activables et désactivables, avec l'heure d'envoi.
- **Invitation à installer l'application** sur l'écran d'accueil : invite native là où le
  navigateur l'expose, marche à suivre détaillée ailleurs. Un refus est mémorisé quinze jours.
- **Retour tactile** optionnel sur les gestes de confirmation, désactivable. Jamais porteur
  d'information : l'interface reste compréhensible sans lui.

### 2.10 Profil et réglages

- Voir ses objectifs calculés (calories, protéines, glucides, lipides, métabolisme de base,
  dépense totale estimée, IMC et sa catégorie).
- Modifier nom, sexe, âge, taille, poids visé, niveau d'activité, but.
- Fixer un objectif calorique manuel qui remplace le calcul.
- Régler la **part des calories d'activité recréditées** au budget, de 0 à 100 %, par un
  curseur — 25 % par défaut.
- Régler le créneau de pesée et les messages de motivation.
- Gérer ses proches, ses connexions externes, ses notifications.
- Basculer thème clair / sombre / système, et choisir une palette de couleurs.
- Voir la version de l'application installée.

### 2.11 Console d'administration

Réservée à une liste d'adresses fixée à l'installation. Vide, la console est fermée à tous.
Elle affiche : comptes inscrits, nouveaux sur 7 et 30 jours, comptes actifs sur sept jours,
saisies totales et hebdomadaires, saisies par compte actif, produits distincts, aliments en
cache et personnalisés, séances (nombre, minutes, calories, part importée), pesées et
comptes qui se pèsent, appareils abonnés aux notifications, palmarès des aliments du
dernier mois, histogramme des quatorze derniers jours. Elle permet de chercher un compte et
de lui envoyer une notification d'essai, en annonçant le nombre d'appareils réellement
touchés.

---

## 3. MODÈLE DE DONNÉES

Convention générale : **une journée est une chaîne `AAAA-MM-JJ`**, jamais un instant daté.
Un repas du 12 mars reste le 12 mars quel que soit le fuseau du client ou du serveur.
Toutes les entités portent une date de création et de dernière modification.

### 3.1 Utilisateur

| Champ | Type | Contraintes |
|---|---|---|
| identifiant | identifiant technique | clé primaire |
| e-mail | texte | **unique**, **indexé**, forcé en minuscules, sans espaces |
| empreinte du mot de passe | texte | obligatoire, **jamais renvoyée** par défaut |
| nom affiché | texte | obligatoire, 2 à 60 caractères |
| sexe biologique | énuméré `male` / `female` | défaut `female` |
| âge | entier | 10 à 120, défaut 30 |
| taille (cm) | nombre | 100 à 250, défaut 170 |
| poids (kg) | nombre | 25 à 400, défaut 70 |
| poids visé (kg) | nombre | facultatif, 25 à 400 |
| niveau d'activité | énuméré `sedentary`/`light`/`moderate`/`active`/`very_active` | défaut `light` |
| but | énuméré `lose`/`maintain`/`gain` | défaut `maintain` |
| objectif calorique manuel | entier | facultatif, 800 à 8000 |
| part d'activité recréditée (%) | entier | 0 à 100, **défaut 25** |
| créneau de pesée | sous-objet | voir ci-dessous |
| réglage de motivation | sous-objet | voir ci-dessous |
| date de fin d'accueil | date | absente tant que le questionnaire n'est pas rempli |
| empreinte du jeton de rafraîchissement | texte | facultatif, **jamais renvoyée** par défaut |

**Créneau de pesée** : actif (booléen, défaut faux) ; jour de la semaine (0 = dimanche à
6 = samedi, défaut 1) ; heure (0 à 23, défaut 8) ; fuseau horaire IANA (défaut
`Europe/Paris`, renseigné automatiquement par le navigateur) ; dernier jour notifié
(`AAAA-MM-JJ` local, interne, jamais exposé).

**Réglage de motivation** : actif (booléen, **défaut vrai**) ; heure (0 à 23, défaut 9) ;
dernier jour envoyé (interne) ; clés des huit derniers messages envoyés, le plus récent en
tête (interne). Les deux jours d'envoi ne sont **pas** stockés : ils se déduisent du jour
de pesée — figer deux numéros de jour les laisserait dériver dès que le membre déplace sa
pesée. Le fuseau n'est pas dupliqué : c'est celui du créneau de pesée, sans quoi les deux
copies finiraient par diverger.

### 3.2 Aliment (fiche produit)

Sert à la fois de cache local des fiches issues de la base ouverte et de catalogue des
aliments personnalisés.

| Champ | Type | Contraintes |
|---|---|---|
| code-barres | texte | facultatif, **unique quand présent**, indexé |
| nom | texte | obligatoire, indexé en texte intégral |
| marque | texte | facultatif |
| URL de l'image | texte | facultatif |
| conditionnement | texte | facultatif (« 400 g ») |
| valeurs pour 100 g | sous-objet | obligatoire |
| Nutri-Score | texte `a`..`e` | facultatif |
| degré de transformation | entier 1 à 4 | facultatif |
| origine | énuméré `openfoodfacts` / `custom` | défaut `openfoodfacts` |
| créé par | identifiant d'utilisateur | renseigné uniquement pour un aliment personnalisé, indexé |
| date de dernière synchronisation | date | facultative |

**Valeurs pour 100 g** (toutes des nombres, défaut 0) : énergie en kcal, protéines,
glucides, dont sucres, lipides, dont acides gras saturés, fibres, sel. Bornes de saisie :
énergie 0 à 1000 ; tous les autres 0 à 100 (grammes pour 100 g).

### 3.3 Ligne de journal alimentaire

| Champ | Type | Contraintes |
|---|---|---|
| utilisateur | identifiant | obligatoire, indexé |
| journée | `AAAA-MM-JJ` | obligatoire, indexé (index composé utilisateur + journée) |
| repas | énuméré `breakfast`/`lunch`/`dinner`/`snack` | obligatoire |
| fiche produit | référence | facultative |
| code-barres | texte | facultatif |
| nom | texte | obligatoire |
| marque, image | texte | facultatifs |
| quantité (g ou ml) | nombre | obligatoire, 0,1 à 5000 |
| **instantané des valeurs pour 100 g** | sous-objet | obligatoire |

L'instantané est le point capital : la ligne **fige** les valeurs au moment de la saisie.
Une correction ultérieure de la fiche produit ne réécrit jamais l'historique déjà consigné.

### 3.4 Séance d'activité

| Champ | Type | Contraintes |
|---|---|---|
| utilisateur | identifiant | obligatoire, indexé (index composé avec la journée) |
| journée | `AAAA-MM-JJ` | obligatoire, indexé |
| clé d'activité | texte | obligatoire (clé du catalogue, `custom`, ou `external`) |
| libellé | texte | obligatoire |
| durée (min) | entier | 1 à 1440 |
| intensité métabolique | nombre | ≥ 0,5 (bornée à 25 à la saisie) |
| dépense (kcal) | nombre | ≥ 0, **figée à la saisie** |
| dépense relevée à la main | booléen | défaut faux |
| note | texte | facultative, 200 caractères |
| provenance | énuméré `manual` / service n° 1 / service n° 2 | défaut `manual` |
| identifiant externe | texte | absent pour une saisie manuelle |
| début réel | date-heure | renseigné pour les séances importées |
| date d'import | date-heure | facultative |

**Index unique partiel** sur (utilisateur, provenance, identifiant externe), **restreint aux
lignes qui portent un identifiant externe** : il rend la synchronisation idempotente sans
faire entrer en collision toutes les saisies manuelles, qui n'en ont pas et se heurteraient
sur la même valeur nulle.

Le début réel est indispensable : la journée seule ne suffit pas à repérer un doublon, deux
services pouvant remonter la même sortie le même jour sous deux identifiants différents.

### 3.5 Pesée

Utilisateur (indexé), journée, poids (25 à 400), note facultative (200 caractères).
**Index unique (utilisateur, journée)** : une seconde saisie le même jour corrige la
première plutôt que d'empiler deux valeurs contradictoires.

### 3.6 Objectifs figés par jour

C'est le mécanisme qui empêche l'historique de se réécrire.

Utilisateur (indexé), **premier jour d'application** (`AAAA-MM-JJ`), calories, protéines (g),
glucides (g), lipides (g), métabolisme de base, dépense totale estimée, part d'activité
recréditée (0 à 100), poids ayant servi au calcul (trace documentaire, pas une pesée).
**Index unique (utilisateur, jour)**.

Un document **par jour de changement**, et non par jour calendaire : il vaut pour sa journée
et toutes les suivantes, jusqu'au prochain changement.

### 3.7 Lien entre deux comptes (« proche »)

Demandeur (indexé), destinataire (indexé), **clé de paire unique** (les deux identifiants
triés et concaténés — un seul lien par paire, quel que soit celui qui invite), statut
(`pending` / `accepted`, défaut `pending`), « le demandeur ouvre ses repas » (booléen,
défaut faux), « le destinataire ouvre ses repas » (booléen, défaut faux).

C'est un lien à deux, pas un groupe : deux personnes suffisent au besoin, et un groupe
aurait exigé un administrateur, des invitations en cascade et des règles de départ.
Plusieurs proches restent possibles : autant de liens. Être proches ne donne accès à rien
d'autre qu'aux envois explicites ; la lecture des repas de l'autre est une permission à
part, accordée par chacun pour lui-même.

### 3.8 Envoi d'aliments

Expéditeur (indexé), destinataire (indexé), journée et repas d'origine, **liste d'aliments
figés** (code-barres, nom, marque, image, quantité de l'expéditeur, instantané des valeurs
pour 100 g — **jamais de référence à une fiche produit**, qui appartiendrait à l'expéditeur),
statut (`pending`/`accepted`/`declined`, défaut `pending`), date d'expiration.

**Purge automatique par la base à l'expiration**, sans tâche planifiée.

### 3.9 Abonnement aux notifications

Utilisateur (indexé), point de terminaison délivré par le navigateur (**unique** — il
identifie l'appareil), deux clés de chiffrement, description du navigateur (300 caractères)
pour reconnaître l'appareil dans les réglages.

### 3.10 Connexion à un service externe

Utilisateur (indexé), fournisseur, jeton d'accès **chiffré**, jeton de rafraîchissement
**chiffré**, date d'expiration de l'accès, périmètres accordés, identifiant et nom du compte
chez le fournisseur, date de dernière synchronisation, dernière erreur, nombre de séances
importées (défaut 0).
**Index unique (utilisateur, fournisseur)** : reconnecter remplace.

### 3.11 Import rejeté

Utilisateur (indexé), fournisseur, identifiant externe.
**Index unique (utilisateur, fournisseur, identifiant externe)**.
Trace une séance importée puis supprimée à la main, pour qu'elle ne revienne pas.

### 3.12 Lecture hebdomadaire

Utilisateur (indexé), jour de génération (local), titre, liste d'observations (titre,
détail, gravité `good`/`watch`/`act`, chiffres cités), action à tenir, profondeur observée
en jours, **document chiffré remis au modèle** (conservé pour relecture ultérieure), modèle
ayant réellement répondu, jetons consommés.
**Index unique (utilisateur, jour)** : un rafraîchissement dans la journée remplace la
précédente plutôt que d'empiler des variantes du même bilan.

Conserver les chiffres avec le texte est ce qui permet de relire une lecture ancienne sans
se demander sur quelles données elle portait — et de constater après coup qu'une remarque
était fausse.

### 3.13 Registre de dépense

Collection **en ajout seul**, séparée des lectures : utilisateur (indexé), jour local, mois
(`AAAA-MM`, dénormalisé, indexé), jetons d'entrée, de sortie, lus et écrits en cache, coût
en euros **figé à l'écriture**, modèle. Index (mois, utilisateur) et (utilisateur, jour).

La séparation est le point : une lecture rédigée deux fois le même jour remplace la
précédente, et sa dépense disparaîtrait avec elle — le plafond mensuel se tromperait alors
systématiquement dans le sens de la générosité, en oubliant précisément les appels qu'il est
chargé de compter.

---

## 4. RÈGLES MÉTIER

### 4.1 Authentification et autorisations

- Le mot de passe est stocké haché par un algorithme à coût de calcul configurable, réglé
  suffisamment haut pour qu'un essai prenne une fraction de seconde perceptible. Il fait au
  moins 8 caractères et au plus 128.
- L'adresse est normalisée en minuscules à l'inscription comme à la connexion.
- À la connexion, un compte inexistant déclenche malgré tout une comparaison de hachage
  contre une valeur factice, pour que le temps de réponse ne révèle pas l'existence du
  compte. Le message d'erreur est identique dans les deux cas.
- Deux jetons : un jeton d'accès court (quinze minutes par défaut) et un jeton de
  rafraîchissement long (trente jours par défaut), **signés avec deux secrets distincts**.
- **Rotation stricte** : un seul jeton de rafraîchissement est valide à la fois par compte.
  Son empreinte est stockée et vérifiée à chaque usage ; chaque rafraîchissement en émet un
  nouveau et invalide l'ancien. La déconnexion l'efface.
- Le client tente un rafraîchissement automatique sur le premier refus d'autorisation, une
  seule fois par requête, jamais sur les routes d'authentification elles-mêmes. Les requêtes
  concurrentes attendent le même rafraîchissement.
- **Un échec de rafraîchissement par absence de réseau ne clôt pas la session.** Seul un
  refus explicite du serveur la ferme. Cette distinction est essentielle : la confondre
  déconnecte l'utilisateur à la moindre coupure de métro, sans retour possible sans
  ressaisir ses identifiants.
- **Cloisonnement absolu** : toute lecture, modification ou suppression d'une donnée est
  filtrée sur l'identifiant du porteur du jeton. Une ressource appartenant à un autre compte
  répond « introuvable », jamais « interdit » — l'existence n'est pas révélée.
- La console d'administration est réservée aux adresses d'une liste blanche fixée à
  l'installation. **L'adresse est lue dans le jeton signé, jamais dans la requête.** La
  liste vide ferme la console à tous, elle ne l'ouvre pas à tous. Le profil renvoie un
  indicateur d'administration, mais il ne commande que l'affichage d'un lien : chaque route
  revérifie l'autorisation. Le refus est muet — il n'y a pas à révéler qu'une console existe
  à qui n'y a pas droit.

### 4.2 Limites de débit

Plafond global de 120 requêtes par minute et par client. Plafonds resserrés : 5 inscriptions
par minute, 10 connexions par minute, 10 invitations de proches par minute (la réponse
révèle si une adresse a un compte — ce plafond borne l'énumération : dix essais suffisent à
inviter un proche, pas à passer un annuaire au crible), 3 demandes de lecture hebdomadaire
par minute.

Derrière un proxy, le client doit être identifié par son adresse réelle, et **la confiance
accordée aux en-têtes du proxy doit être restreinte aux plages d'adresses privées et à la
boucle locale** : sinon un client public s'attribue une identité arbitraire et contourne la
limite, ou bien tous les clients partagent un seul quota.

### 4.3 Calcul des objectifs

1. **Métabolisme de base** (Mifflin-St Jeor) :
   `10 × poids(kg) + 6,25 × taille(cm) − 5 × âge`, puis `+ 5` pour un homme, `− 161` pour
   une femme. Arrondi à l'entier.
2. **Dépense totale estimée** = métabolisme de base × facteur du niveau d'activité :
   sédentaire 1,2 · légèrement actif 1,375 · modérément actif 1,55 · actif 1,725 ·
   très actif 1,9. Arrondi à l'entier.
3. **Objectif calorique** = dépense totale × facteur du but : perte 0,8 · maintien 1 ·
   prise 1,12. **Plancher de sécurité : jamais sous le métabolisme de base.** Un objectif
   manuel, s'il est renseigné, remplace ce résultat.
4. **Protéines** = poids × g/kg selon le but : perte 2 · maintien 1,6 · prise 1,8
   (plus haut en déficit pour préserver la masse maigre).
5. **Lipides** = objectif calorique × part selon le but (perte 0,28 · maintien 0,30 ·
   prise 0,27) ÷ 9.
6. **Glucides** = `max(0, (objectif − protéines×4 − lipides×9) / 4)`.
7. **IMC** = poids / taille(m)², arrondi à une décimale. Catégories : < 18,5 insuffisance
   pondérale ; < 25 corpulence normale ; < 30 surpoids ; < 35 obésité modérée ; au-delà
   obésité sévère.

Ce calcul n'est implémenté qu'une fois, côté serveur, et l'aperçu du questionnaire d'accueil
passe par lui : deux implémentations d'une même formule métabolique finiraient par diverger,
et le chiffre annoncé ne serait plus celui qui s'enregistre.

### 4.4 Objectifs figés jour par jour — invariant central

Une pesée ou une retouche du profil recalcule l'objectif **à compter du jour même, dans le
fuseau de l'utilisateur**. Les journées passées gardent le leur.

- Toute modification qui change au moins un des sept champs comparés (calories, protéines,
  glucides, lipides, métabolisme de base, dépense totale, part recréditée) écrit un
  enregistrement daté du jour courant de l'utilisateur. Une retouche sans effet — changer
  son nom, son créneau de pesée — n'écrit rien.
- **Première fois qu'un objectif change** alors que rien n'est encore figé : on fige
  d'abord ce qui s'appliquait jusque-là, daté de **la veille**, pour que les journées plus
  anciennes en héritent et que le nouvel objectif ne remonte pas le temps.
  **Exception** : si le compte a été créé le jour même, il n'y a rien à figer — l'objectif
  « d'avant » n'était que celui d'un profil par défaut que personne n'avait renseigné, et le
  figer jugerait les journées rattrapées à l'aune d'un profil fictif.
- **Lecture d'une période** : chaque journée prend le dernier objectif figé à sa date ou
  avant. Une journée antérieure à tout objectif figé (un repas rattrapé d'avant
  l'inscription) prend le plus ancien. Sans aucun historique, c'est le profil courant qui
  parle — rien n'a changé depuis, il donne le même résultat.
- Plusieurs retouches dans la même journée se remplacent : seule la dernière vaut pour ce
  jour-là.

**Conséquence vérifiable** : une pesée en baisse aujourd'hui ne doit abaisser ni l'objectif
d'hier, ni la couleur d'une pastille du calendrier, ni la ligne d'objectif d'un graphique
sur une journée passée. Sans cette règle, des journées respectées à l'époque viraient
rétroactivement au dépassement.

### 4.5 Budget du jour et dépense recréditée

- **Dépense d'une séance** = `intensité × 3,5 × poids(kg) ÷ 200 × durée(min)`, arrondie à
  l'entier, **figée à la saisie** : le poids évolue, l'historique ne se réécrit pas.
- Une **dépense relevée** prime toujours sur le calcul, même pour une activité du catalogue :
  la mesure d'un appareil vaut mieux qu'une moyenne. Elle est marquée comme telle, et
  changer la durée ne la recalcule pas — écraser une mesure réelle par une estimation serait
  une régression.
- Pour une activité hors catalogue, il faut soit une intensité, soit une dépense. Ni l'une
  ni l'autre : refus explicite. Quand seule la dépense est connue, l'intensité est déduite
  par la formule inverse et bornée entre 0,5 et 25 — à titre documentaire uniquement.
- **Part recréditée** : seule la fraction choisie dans le profil (0 à 100 %, **défaut 25 %**)
  rejoint le budget du jour. Justification : les formules d'intensité surestiment largement,
  elles englobent le métabolisme de base déjà compté dans la dépense totale, et le facteur
  d'activité du profil pré-compte lui aussi une part de l'effort. Tout recréditer annulerait
  une bonne part du déficit visé et relancerait la faim le jour même.
- **Budget du jour** = objectif du jour + part recréditée de la dépense du jour.
  **Reste** = budget − apports. Négatif en dépassement.
- **Bilan net d'une journée** = apports − part recréditée. C'est **cette** valeur qui est
  comparée à l'objectif partout : tableau de bord, calendrier, graphiques, séries,
  statistiques, alerte fringale. Retrancher la dépense brute ferait passer pour respectée
  une journée que le tableau de bord donne en dépassement — les deux écrans se
  contrediraient.
- La part recréditée est relue **jour par jour** dans les objectifs figés. La changer
  aujourd'hui ne repeint pas le passé.

### 4.6 Proratisation et totaux

- Totaux d'une ligne = valeurs pour 100 g × quantité ÷ 100, chaque nutriment arrondi à une
  décimale.
- Totaux d'un repas = somme des lignes ; totaux du jour = somme des repas ; arrondis à une
  décimale à chaque niveau.
- Cette règle de proratisation est **unique** dans toute l'application. Elle ne doit être
  réimplémentée ni côté client, ni dans une requête d'agrégation de la base : deux vérités
  divergeraient.

### 4.7 Résolution d'un aliment à l'ajout

Ordre de priorité strict : identifiant de fiche interne, puis code-barres (cache local, puis
base ouverte), puis saisie libre (nom **et** valeurs pour 100 g). Aucun des trois : refus
avec un message qui dit les trois possibilités. Les nutriments facultatifs manquants valent 0.

### 4.8 Cache des fiches produits

- Un scan cherche d'abord en cache local. Une fiche de moins de sept jours est servie telle
  quelle ; un aliment personnalisé l'est toujours.
- Au-delà, on interroge la base ouverte et on met à jour la fiche.
- **Si la base ouverte est injoignable mais qu'une fiche périmée existe, on sert la fiche
  périmée** : elle reste plus utile qu'une erreur.
- Produit inconnu et absent du cache : « produit inconnu, vous pouvez le saisir manuellement ».

### 4.9 Recherche par nom et classement

La base ouverte trie par popularité, ce qui place « sauce tomate » et « pizza » devant
« tomate ». Le classement est donc refait localement. Règles, en points relatifs — les points
ne valent que les uns contre les autres :

- **Normalisation** : minuscules, accents retirés, découpage sur espaces, virgules,
  apostrophes et tirets. Pluriel français retiré (marque finale `s`/`x`) **seulement
  au-delà de trois lettres**, sinon « jus » deviendrait « ju » et « riz » « ri ».
- **Correspondance exacte du libellé entier** avec la requête : +120.
- **Position du premier terme de la requête** dans le libellé : +60 au premier mot, +24 au
  deuxième, +10 au-delà. L'écart entre le premier et le deuxième rang est volontairement
  large : en français la tête du groupe nominal vient en premier — « sauce tomate » est une
  sauce, « tomate cerise » est une tomate. Cette seule position sépare déjà l'aliment brut
  de la préparation qui le contient. À qualité égale, le mot le plus à gauche l'emporte.
- Un mot qui ne fait que **commencer** par le terme cherché ne compte que pour 0,6
  (« pruneaux » commence par « prune » sans en être).
- **Part des termes de la requête retrouvés** × 30.
- **Brièveté** : 40 points, moins 7 par mot excédentaire, plancher 0 (« sauce tomate
  basilic cuisinée » s'éloigne à chaque mot).
- **Absence de marque** : +12 (une tomate n'appartient à personne).
- **Degré de transformation** : 1 → +18, 2 → +8, 3 → 0, 4 → −6.
- Tri **stable** : à note égale, l'ordre de la base ouverte (donc la popularité) départage.

**Complétion du dernier mot.** Le moteur de la base ouverte indexe des mots entiers :
« chocol » ne rencontre littéralement jamais « chocolat », mais bien la douzaine de produits
qui portent ce fragment — le nombre de résultats ne trahit donc pas le problème. Deux appels
partent simultanément : la recherche telle quelle, et une demande de complétion portant sur
**la requête entière** (« pain de mi » → « pain de mie » ; le fragment final seul n'aurait
aucun contexte). Le mot retenu est celui qui occupe la position du fragment dans la
suggestion, et seulement s'il le prolonge réellement — sur une requête déjà complète, la
complétion ne rend rien et le second aller-retour n'a pas lieu. S'il existe, une seconde
recherche part avec le terme complété, et ses résultats passent devant. Une panne de la
complétion n'invalide pas la recherche.

**Filtrage marché.** La recherche est cadrée sur la langue française et sur les produits
distribués en France : la base est mondiale, et une recherche de « tomate » y remonte sinon
surtout des conserves étrangères introuvables en rayon. Si les deux tentatives ne ramènent
rien, une dernière recherche mondiale est lancée — le produit a peut-être été rapporté de
voyage.

**Neutralisation de la syntaxe.** Les caractères réservés du langage de requête sont
**remplacés par des espaces**, pas échappés : une apostrophe ou un deux-points saisis tels
quels font échouer l'analyse côté serveur, et la recherche plein texte n'en tire aucun parti.

**Conversion d'énergie** : quand seule l'énergie en kilojoules est publiée, elle est
convertie (1 kcal = 4,184 kJ). Les résultats sans énergie sont écartés. Quand plusieurs
marques sont listées, seule la première est affichée.

### 4.10 Copie d'un repas

- Les instantanés stockés sont **dupliqués tels quels** : aucune résolution auprès de la
  base ouverte, donc aucun risque qu'une fiche corrigée entre-temps change les valeurs d'un
  repas déjà mangé — et un seul aller-retour au lieu d'un par ligne.
- Les lignes **s'ajoutent** à l'existant, elles ne le remplacent pas : écraser un repas déjà
  renseigné serait destructif pour un geste qu'on peut vouloir répéter.
- Une sélection partielle est bornée au repas source : un identifiant d'un autre jour,
  d'un autre repas ou d'un autre compte est ignoré silencieusement.
- Une liste de sélection **vide est refusée** plutôt que lue comme « tout » : une sélection
  vidée par erreur copierait l'inverse de ce qui est demandé (au moins un élément, au plus
  cent).
- Si la sélection ne correspond à rien, le message distingue « ces aliments ne figurent plus
  dans ce repas » de « ce repas ne contient rien à copier ».
- Le système renvoie les lignes créées, ce qui permet au client de proposer d'annuler la
  copie sans recharger la journée.
- L'historique proposé remonte à trente jours — au-delà, on ne se souvient plus de ce qu'on a
  mangé un jour donné et la liste devient un mur de dates indistinctes. Les suggestions
  d'aliments remontent à quatre-vingt-dix jours : assez pour dégager des habitudes, assez
  court pour qu'un aliment abandonné cesse d'être proposé. La journée affichée est exclue de
  l'historique : se proposer de copier un repas sur lui-même n'a pas de sens.

### 4.11 Suggestions d'aliments

Regroupement sur l'identifiant le plus fiable disponible — fiche interne, puis code-barres,
puis nom normalisé — pour qu'un même produit saisi de deux façons ne compte pas double.
Mode « fréquent » : tri par nombre d'occurrences, puis récence. Mode « récent » : l'inverse.
La **dernière quantité saisie** accompagne chaque suggestion et pré-remplit le formulaire.
Aucune jointure avec le catalogue de fiches : les lignes portent déjà un instantané complet,
et un produit retiré de la base reste proposable.

### 4.12 Poids et profil

- Le poids du profil suit **toujours** la pesée la plus récente : c'est lui qui alimente le
  métabolisme de base et la dépense des séances. Deux points d'entrée pour la même grandeur
  les laisseraient diverger, et la courbe théorique repartirait d'un poids que la balance n'a
  jamais affiché.
- Dès qu'une pesée existe, **le poids n'est plus modifiable depuis le profil**. La tentative
  est refusée avec un message qui indique où se fait la pesée. Les autres champs restent
  modifiables.
- Enregistrer une pesée antérieure à la plus récente ne touche pas au profil.
- **Supprimer** la pesée la plus récente ramène le poids du profil à la précédente — sans
  cela, une faute de frappe supprimée (72 au lieu de 82) laisserait l'objectif calculé sur un
  poids qui n'existe plus.
- La fin du questionnaire d'accueil crée la première pesée, datée du jour : le point de départ
  est déjà connu, inutile de le ressaisir. L'amorçage n'a lieu que si l'historique est vide ;
  passé ce cap, la pesée est un geste délibéré et corriger son profil ne doit pas réécrire la
  mesure du jour. Un échec d'amorçage (course entre deux enregistrements simultanés) ne fait
  jamais échouer l'enregistrement du profil : c'est un confort.
- Un second envoi du questionnaire est **idempotent** : il ne recrée pas de pesée et ne
  réécrit pas le poids. Sans cette garantie, le second passage échouerait précisément sur le
  verrou ci-dessus.

### 4.13 Poids théorique

- Point de départ : la **première pesée de l'année écoulée**. Sans pesée, la courbe est vide.
- Chaque journée renseignée pèse sur le lendemain matin :
  `poids −= (dépense totale estimée du jour + part recréditée des séances − apports) / 7700`.
  La valeur de 7 700 kcal par kilo est un repère d'usage courant ; le tissu adipeux n'est pas
  de la graisse pure, et ce repère suffit à une tendance sur quelques semaines.
- **Même lecture de la dépense que le budget** : compter les séances en entier prédirait une
  perte que l'application elle-même juge surestimée.
- **Une journée sans repas saisi est tenue pour équilibrée**, pas pour un jeûne. Inventer
  zéro calorie produirait une courbe spectaculairement fausse ; mieux vaut une courbe qui ne
  dit rien de ce qu'elle ignore.
- **La journée en cours n'entre pas dans le calcul** : elle n'est pas finie, et un
  petit-déjeuner seul passerait pour un déficit spectaculaire. Le client transmet son jour
  courant pour que la courbe s'y arrête.
- Un point par jour, poids arrondi au centième, le point du jour reflétant le bilan des
  journées **précédentes** (c'est une pesée du matin).

### 4.14 Analyse des écarts pesée / théorique

Fenêtre par défaut : quatre-vingt-dix jours (réglable de 30 à 365). Au-delà d'un an, les
objectifs figés remontent à un profil qui n'a plus rien à voir avec celui d'aujourd'hui et la
conclusion ne porterait sur rien de réglable ; en deçà d'un mois, le bruit des pesées gagne.

**Découpage en intervalles.** Entre deux pesées consécutives, on reconstitue le bilan
énergétique. La pesée du matin reflète les journées **qui la précèdent, la sienne exclue**.
Un intervalle est écarté s'il fait moins de cinq jours (le bruit y domine), si sa pesée de
fin est postérieure au jour courant, ou si **moins de 70 % de ses journées sont
renseignées** — la courbe théorique y a supposé l'équilibre, et une supposition ne peut pas
servir de preuve.

**Agrégation.** Somme des bilans prédits, somme des variations réelles converties en
calories, sur les seules journées renseignées. En dessous de **quatorze journées
renseignées au total, l'analyse se tait** plutôt que de conseiller sur du sable.

**Marge de bruit.** Une pesée porte une incertitude d'environ 0,4 kg (eau, glycogène, heure
de la mesure). Les intervalles contigus se chaînent : la pesée qui ferme l'un ouvre le
suivant, et son bruit s'annule. Seules comptent les extrémités de chaque série contiguë — un
intervalle écarté rompt la chaîne. La marge quotidienne vaut donc
`0,4 × √(2 × nombre de séries) × 7700 / journées renseignées`. C'est ce qui fait fondre la
marge à mesure que l'historique s'allonge.

**Verdict.** Seuil = `max(marge, 75 kcal/jour)`. Écart quotidien en valeur absolue sous le
seuil → **aligné**, aucune suggestion. Au-dessus et positif → **plus lent que prévu** (le
modèle surestime la dépense, ou sous-estime les apports). Au-dessus et négatif → **plus
rapide que prévu**.

**Confiance** : élevée si ≥ 28 journées **et** rapport signal/seuil ≥ 2 ; moyenne si
≥ 21 journées **ou** rapport ≥ 1,5 ; faible sinon.

**Suggestions — lectures concurrentes du même écart, jamais des corrections à cumuler.**
En appliquer une suffit ; l'analyse suivante dira si elle a suffi.

1. **Part recréditée** — le réglage le plus incertain du modèle. Proposée seulement si des
   séances existent et si la dépense moyenne atteint 80 kcal/jour, si la valeur qui
   refermerait l'écart reste entre 0 et 100 %, et si elle diffère de la valeur courante d'au
   moins un pas de 5 % (une suggestion doit tomber sur une valeur réglable au curseur).
   Valeur exacte : `part actuelle − 100 × écart cumulé / dépense brute cumulée`, arrondie au
   pas de 5.
2. **Niveau d'activité** — le palier dont le facteur approche le mieux
   `(dépense totale moyenne − écart quotidien) / métabolisme de base`, s'il diffère du
   palier actuel et reste dans l'échelle. Hors échelle, aucun palier ne convient : l'écart
   vient d'ailleurs, la suggestion n'est pas émise.
3. **Saisie des apports** — toujours proposée en dernier : huile de cuisson, portions à
   l'œil, boissons et grignotages oubliés. C'est la cause la plus courante, et la seule
   qu'aucun réglage ne corrige.

### 4.15 Statistiques de suivi (séries, régularité)

Lecture commune aux messages de motivation et à l'alerte fringale — les deux doivent compter
les séries **de la même façon**, sinon « série de 4 jours » vaudrait 4 dans un écran et 5
dans l'autre.

- **La journée en cours est toujours exclue.**
- Seules les journées effectivement renseignées entrent : une journée absente n'est pas une
  journée à zéro calorie, et la compter comme telle inventerait un jeûne dans chaque série.
- *Journées renseignées* sur la fenêtre ; *journées tenues* = celles dont le bilan net reste
  ≤ objectif ; *série sous objectif* et *série de suivi* comptées en remontant depuis la
  veille et **interrompues par le premier trou** ; *déficit cumulé* = somme des
  (objectif − bilan net), positif quand c'est de l'avance prise.
- *Jours depuis la dernière saisie* : absent si rien n'a été saisi sur la profondeur explorée
  — ce qui distingue le compte neuf de la longue absence.
- *Comparaison de semaines* : moyenne des bilans nets des sept derniers jours moins celle
  des sept précédents. **Absente si l'une des deux semaines compte moins de deux journées** :
  comparer une semaine pleine à un unique repas produirait un écart spectaculaire et faux.

### 4.16 Alerte fringale — sélection des arguments

Dix règles, ordonnées par priorité décroissante ; les trois meilleures applicables sont
retenues (au-delà, la page se lit comme un sermon et se referme). Historique lu : vingt et un
jours ; fenêtre de la « séance récente » : trois jours.

| Priorité | Condition |
|---|---|
| 100 | Budget restant ≥ 150 kcal |
| 95 | Meilleure séance des trois derniers jours ≥ 200 kcal |
| 90 | Déficit de la semaine ≥ 1000 kcal **et** ≥ 3 journées renseignées |
| 85 | Série sous objectif ≥ 3 jours |
| 80 | Perte ≥ 0,3 kg sur la quinzaine |
| 70 | Série de suivi ≥ 5 jours |
| 65 | ≥ 4 journées renseignées **et** au plus une non tenue |
| 60 | Apports du jour ≤ 60 % de l'objectif |
| 55 | ≥ 2 séances en trois jours |
| 10 | **Repli — s'applique toujours** |

Un argument vrai et daté — « votre sortie de mardi a brûlé 480 kcal » — vaut mieux que dix
encouragements génériques. Les repères temporels sont relatifs (« hier », « il y a 3 jours »).
La règle de repli garantit qu'un compte neuf ne tombe pas sur une page vide au moment précis
où il a besoin d'aide. Le budget restant affiché ici est calculé **exactement comme** celui du
tableau de bord : en afficher un autre ferait mentir l'un des deux écrans.

Le craquage éventuel est enregistré comme **une collation ordinaire**, à la quantité
effectivement mangée (entière, ou la moitié si le sort l'a décidé). Il ne vit pas dans une
collection à part : un craquage assumé reste un repas.

### 4.17 Messages de motivation

- **Deux créneaux par semaine**, exprimés comme des décalages par rapport au jour de pesée :
  +1 et +4 jours, modulo 7. Une pesée le lundi donne mardi et vendredi — le premier message
  suit le week-end, quand la semaine se relance ; le second tombe au milieu du cycle, avant
  le week-end qui fait dérailler le plus de suivis. Exprimés relativement, ils suivent le
  membre qui déplace sa pesée au lieu de partir à contretemps. Le décalage de 1 évite en
  outre de doubler le rappel de pesée le jour même.
- Balayage horaire de tous les comptes dont la motivation n'est **pas explicitement
  désactivée** (un compte antérieur au réglage, sans le champ en base, se comporte comme un
  compte neuf : il reçoit les messages — un filtre d'égalité stricte ne le ramènerait jamais).
- Envoi si l'heure locale du membre correspond, si le jour est l'un des deux créneaux, et si
  le jour n'a pas déjà été marqué.
- **Le jour est marqué avant l'envoi, pas après** : un échec laisserait sinon le créneau
  ouvert, et la passe suivante — dans l'heure — rejouerait le même membre. Mieux vaut manquer
  un message que d'en envoyer trois.
- Treize règles, par priorité décroissante : objectif de poids atteint (100) · retour après
  ≥ 3 jours d'absence (90) · pesée en retard de ≥ 12 jours (80) · perte ≥ 0,3 kg sur la
  quinzaine, en objectif de perte (70) · reprise ≥ 0,5 kg, en objectif de perte (65) ·
  plateau (60) · semaine parfaite, ≥ 7 journées dont ≥ 6 tenues (55) · série sous objectif
  ≥ 3 jours (50) · volume d'activité ≥ 150 min (seuil hebdomadaire recommandé par l'OMS) ou
  ≥ 4 séances (45) · aucune séance avec ≥ 3 journées renseignées (40) · amélioration de
  ≥ 150 kcal entre les deux semaines (35) · suivi partiel de 1 à 4 journées (30) ·
  **repli (10), toujours applicable**.
- Chaque règle porte **au moins trois formulations** : en deçà, la rotation se voit.
- **Anti-répétition à deux niveaux** : les huit derniers messages envoyés sont mémorisés ;
  une règle servie dans les trois derniers messages cède la place à la suivante applicable ;
  au sein de la règle retenue, une formulation déjà servie est écartée. Si toutes ont servi,
  on reprend quand même — mieux vaut un message déjà lu que pas de message.
- Le point de départ du choix de formulation dépend du membre et du jour : deux personnes
  dans la même situation le même jour ne lisent pas la même phrase, mais un second passage
  dans l'heure choisirait le même texte.
- La mémoire n'est mise à jour que si au moins un appareil a été touché.
- Un journal vierge n'est pas une absence : la règle du retour ne s'applique pas à qui n'a
  jamais rien saisi.
- Un membre dont le bilan échoue ne prive pas les suivants du leur.
- La journée en cours est exclue de bout en bout : à 9 h du matin, un petit-déjeuner seul
  passerait pour un jeûne et fausserait chaque règle.

### 4.18 Rappel de pesée

Balayage horaire des comptes dont le rappel est actif. Envoi si le jour et l'heure locaux
correspondent et si le jour n'a pas déjà été marqué. **Le rappel n'est pas envoyé si une
pesée existe dans les sept derniers jours**, mais le jour est marqué quand même. Modifier le
créneau efface la trace du dernier envoi, sinon un nouveau créneau tombant le même jour
resterait sans rappel. Un fuseau inconnu du système fait **ignorer** le membre plutôt que de
le notifier à une heure arbitraire.

Le balayage horaire est préféré à un planificateur par membre : les créneaux sont à la
granularité de l'heure, et une requête indexée par heure coûte moins qu'un planificateur à
réarmer à chaque modification de profil.

### 4.19 Notifications

- L'abonnement est identifié par le point de terminaison du navigateur : une réinstallation
  en produit un nouveau, une reconnexion réutilise le même.
- Un abonnement refusé par le service de notification avec un code « introuvable » ou
  « définitivement parti » est **supprimé** : le conserver ferait échouer chaque envoi
  ultérieur.
- Sans clés de notification côté serveur, le module se désactive, l'annonce au démarrage, et
  le client masque l'option.
- Une notification qui échoue ne fait jamais échouer l'action qui l'a déclenchée (invitation,
  envoi d'aliments).
- Les notifications portent une étiquette distincte par famille, pour qu'un message de
  motivation n'écrase pas un rappel de pesée sur l'appareil.
- Chaque notification porte la route à ouvrir au clic.

### 4.20 Proches et partage

- On ne peut inviter qu'un **compte existant**, par son adresse. Adresse inconnue : refus
  explicite. S'inviter soi-même : refus.
- **Une seule relation par paire**, garantie par une clé triée : deux invitations croisées
  ne créent pas deux liens concurrents. Si l'autre m'avait déjà invité, inviter en retour
  **vaut acceptation**.
- Inviter deux fois : conflit. Inviter quelqu'un qui est déjà un proche : conflit.
- Seul le destinataire peut accepter, et seulement tant que l'invitation est en attente.
- Retirer un proche, refuser une invitation, annuler la sienne : **un seul geste**, dans
  tous les cas le lien disparaît. Les envois **en attente** entre les deux comptes partent
  avec lui : accepter après coup les aliments d'une personne retirée de ses proches n'aurait
  pas de sens.
- **Envoi** : l'expéditeur doit avoir un lien accepté. Les aliments sont figés au moment de
  l'envoi : l'expéditeur peut ensuite corriger ou supprimer ses lignes, l'envoi n'en dépend
  plus. Aucune référence à une fiche produit n'est transmise — elle appartient à l'expéditeur.
- **Acceptation** : le destinataire seul, sur un envoi en attente et non expiré. Il désigne
  les aliments par leur **position dans l'envoi** ; des positions dupliquées ou hors bornes
  sont refusées. Le passage à « accepté » est **conditionnel à l'état « en attente »** :
  deux validations simultanées (deux appareils, un double appui) n'ajoutent les aliments
  qu'une fois. Si l'ajout au journal échoue ensuite, l'envoi **redevient disponible** plutôt
  que d'être perdu.
- Un envoi ne s'accepte qu'une fois. L'expéditeur ne peut pas accepter à la place du
  destinataire.
- **Ouverture des repas** : permission distincte, réglée par chacun pour son propre côté,
  **fermée par défaut**. Elle ne vaut que dans un sens. Sans elle, l'accès aux repas du
  proche est refusé explicitement.
- La consultation des repas d'un proche **inclut le jour même** — c'est le cas le plus
  courant, le dîner que l'autre vient de saisir.
- Une reprise chez un proche se fait **aux portions du repreneur**, pas à celles du proche.
- Les envois sans réponse expirent au bout de quatorze jours et sont purgés automatiquement :
  un dîner vieux de deux semaines n'a plus rien à faire dans une boîte de réception.
- Les envois et invitations déclenchent une notification, rédigée côté serveur (le possessif
  suit le genre du repas : « son petit-déjeuner », « sa collation »).
- Un compte supprimé fait disparaître les liens qui le désignaient de la liste des proches,
  sans erreur.

### 4.21 Services externes

- **Les jetons sont chiffrés au repos** par un chiffrement authentifié à clé symétrique de
  256 bits, avec vecteur d'initialisation aléatoire et étiquette d'authentification : une
  copie de la base ne suffit pas à les rejouer, et toute altération du chiffré est détectée
  au déchiffrement — un déchiffrement silencieusement corrompu produirait des appels
  incompréhensibles à déboguer. Le format porte un préfixe de version, pour éviter une
  migration aveugle le jour où l'algorithme change. **Sans clé de chiffrement configurée, les
  deux intégrations sont désactivées** — un jeton en clair donnerait accès à des données de
  santé. La clé est validée au démarrage, pas au premier chiffrement en pleine requête.
- **L'état OAuth est un jeton signé, daté, à durée de vie courte (dix minutes)**, portant
  l'identifiant du membre, le fournisseur visé et un marqueur de finalité. Le secret qui le
  signe est **dérivé** du secret d'authentification, dans un espace de clés distinct : un
  jeton d'accès volé ne peut pas être présenté comme un état valide, ni l'inverse. Un état
  émis pour un fournisseur est refusé sur l'autre.
- La page de retour du fournisseur **n'est pas authentifiée par en-tête** — le navigateur y
  arrive par simple redirection. L'état signé est le seul lien avec la session. Elle répond
  par une **redirection vers le front** avec un statut lisible (`ok`, `refus`, `erreur`),
  jamais par du JSON : c'est un écran, pas un appel d'API. Un refus de consentement n'est pas
  une panne : il est journalisé sans bruit.
- **Deux traitements différents du jeton de rafraîchissement** : l'un des fournisseurs en
  émet un neuf à chaque échange et le nouveau doit remplacer l'ancien, sous peine de perdre
  la connexion au renouvellement suivant ; l'autre ne l'émet qu'au premier consentement et
  l'omet ensuite — **l'écraser avec une valeur vide couperait la connexion à l'expiration
  d'après**. L'écriture est donc conditionnelle à la présence d'une valeur. L'URL
  d'autorisation du second demande explicitement un accès hors ligne et force le
  consentement, sans quoi aucun jeton de rafraîchissement n'est jamais émis.
- Le jeton d'accès est renouvelé **cinq minutes avant** son expiration — renouveler pile à
  l'heure perdrait les appels en vol — et **persisté avant tout usage**.
- **Fenêtre de synchronisation** : quatorze jours à la première connexion, puis deux jours de
  recouvrement à partir de la dernière synchronisation — une séance corrigée après coup chez
  le fournisseur serait sinon figée sur sa première valeur.
- **Idempotence** : l'écriture se fait en insertion-ou-mise-à-jour sur (utilisateur,
  fournisseur, identifiant externe). Un cycle rejoué ne duplique rien ; une séance corrigée
  chez le fournisseur est mise à jour.
- **Dédoublonnage inter-services** : deux séances de provenances **différentes** dont les
  heures de début s'écartent de moins de quinze minutes sont tenues pour la même. La journée
  seule ne suffirait pas à les repérer, et sans ce filtre une même sortie compterait deux
  fois dans le budget du jour.
- **Rejets persistants** : supprimer une séance importée enregistre son identifiant externe.
  Le cycle suivant l'ignore, et n'en paie même pas l'appel de détail. Sans cette trace,
  l'utilisateur verrait resurgir une ligne qu'il vient d'effacer, sans comprendre pourquoi.
- La dépense importée est **marquée comme relevée** : aucun recalcul ne viendra jamais
  l'écraser. L'intensité associée est déduite de la dépense, à titre documentaire.
- **Journée locale** : les deux interfaces renvoient une date avec décalage horaire
  (`2026-03-12T07:30:00+01:00`). C'est ce décalage qui porte la journée vécue. La ramener en
  temps universel rangerait une séance de 00 h 30 la veille.
- **Quotas** : ils se comptent par application, pas par utilisateur. Les synchronisations
  s'enchaînent **en série**, jamais en parallèle — vingt synchronisations simultanées
  épuiseraient d'un coup la fenêtre courte d'un fournisseur — et un cycle encore en cours
  fait sauter le suivant. Chez le fournisseur qui n'expose les calories que sur le détail de
  chaque séance, le nombre d'appels de détail est plafonné par cycle (vingt-cinq) et le
  reliquat part au cycle suivant ; la synchronisation s'écourte d'elle-même quand le quota
  restant annoncé par le fournisseur devient faible.
- **Échecs** : une erreur d'autorisation demande une reconnexion ; une limite de débit
  annonce une nouvelle tentative plus tard ; le reste est un message générique. L'erreur est
  **stockée sur la connexion et affichée**. Une reconnexion réussie l'efface.
- **Révocation** : la déconnexion locale aboutit même si la révocation chez le fournisseur
  échoue — l'inverse retiendrait l'utilisateur prisonnier d'une intégration qu'il veut retirer.
- Aucun rappel entrant (« webhook ») n'est déclaré : pour un suivi calorique, une séance vue
  avec deux heures de retard ne change rien à la journée, et des webhooks exigeraient un
  point d'entrée public déclaré chez chaque fournisseur.
- **Réserve documentée** : la dépense remontée par l'un des services est la dépense **totale**
  de la séance, métabolisme de base compris. Elle n'est pas retouchée ; le réglage de part
  recréditée absorbe cette marge. Il est recommandé, quand on importe des dépenses mesurées,
  de basculer le niveau d'activité du profil sur « sédentaire » : le facteur d'activité
  pré-compte sinon un effort que les capteurs mesurent déjà.

### 4.22 Lecture hebdomadaire rédigée

**Partage du travail — la règle fondatrice.** Tout ce qui se calcule est calculé par
l'application et remis au modèle sous forme de **nombres arrêtés** (moyennes, parts de
macronutriments, découpage semaine / week-end, écart au poids théorique, palmarès
d'aliments, répartition par repas, mélange des séances). Le modèle ne compte jamais : il
croise les dimensions et hiérarchise ce qui mérite d'être dit. C'est ce qui rend la lecture
vérifiable — chaque phrase doit pouvoir se retrouver dans le document, affiché à côté. Le
document est volontairement compact, de l'ordre de deux kilo-octets : y noyer quarante séries
journalières produirait un commentaire vague sur tout plutôt qu'une remarque juste sur une
chose.

**Fenêtre** : vingt-huit jours au maximum — sept journées ne suffisent pas à distinguer une
habitude d'un week-end prolongé — **arrêtée à la date d'inscription**. Sur un compte de douze
jours, la fenêtre pleine donnait une couverture de 43 % et la lecture concluait à un journal
lacunaire, en reprochant seize journées que le membre n'avait aucun moyen de renseigner. La
journée en cours est exclue.

**Seuil** : en deçà de dix journées renseignées, aucun appel — il n'y aurait rien à croiser,
et l'appel produirait une paraphrase des trois chiffres disponibles.

**Consigne imposée au modèle** :
- N'inventer aucun nombre ; ne calculer ni moyenne, ni pourcentage, ni projection, ni
  conversion. Si un chiffre manque, écrire la remarque sans lui ou ne pas l'écrire.
- Une valeur absente signifie « la période ne permet pas de la calculer », **jamais zéro** :
  n'en tirer aucune remarque et ne pas la citer.
- Si l'analyse des écarts est insuffisante en données, ne rien conclure sur l'écart
  pesée / théorique.
- Si la couverture est inférieure à 0,6, le dire et signaler que la lecture porte sur un
  journal lacunaire. En dessous de quatorze jours, annoncer une lecture provisoire portant
  sur un début d'historique.
- Ne jamais annoncer une période plus longue que celle réellement observée, ni reprocher
  l'absence de journées antérieures au début de la fenêtre.
- Chercher **le croisement, pas la répétition** : l'écran affiche déjà les moyennes.
- Ton : deuxième personne, direct, sobre, sans exclamation ni familiarité. Constater, ni
  encourager ni sermonner. Une observation défavorable s'énonce comme un fait, jamais comme
  un reproche, et jamais en jugeant la personne.
- **Interdits** : aucun diagnostic ni interprétation médicale, aucune mention de pathologie
  ou de carence ; ne jamais recommander de descendre sous le métabolisme de base, de sauter
  des repas ou de jeûner ; ne fixer aucun objectif de poids ni rythme de perte ; ne commenter
  ni le corps, ni l'apparence, ni la volonté.
- **Garde-fou de sécurité** : si les données montrent une perte supérieure à 1 % du poids par
  semaine, ou des apports moyens sous le métabolisme de base, **l'unique observation de
  gravité maximale doit être l'invitation à consulter un médecin ou un diététicien — et rien
  d'autre**.
- **Forme** : trois à cinq observations, au plus une de gravité maximale, chacune avec les
  chiffres qui la fondent ; une seule action à tenir la semaine suivante, formulée comme une
  action concrète.

Le domaine est sensible : l'application comporte déjà un parcours anti-craquage, et le ton
d'une lecture hebdomadaire ne doit pas le contredire en culpabilisant.

**Sortie structurée** imposée par un schéma, plutôt qu'une consigne « réponds en JSON » :
une remarque contenant une apostrophe suffirait à casser un JSON libre, et l'échec
arriverait en production, pas au test. Attention : les schémas de sortie structurée
n'acceptent pas toutes les contraintes — cardinalité maximale, longueurs, bornes numériques
sont refusées, et une valeur non permise fait échouer la requête entière, sans réponse
dégradée. La borne haute du nombre d'observations est donc appliquée **à la relecture**
(troncature) : cinq observations justes suivies d'une sixième de trop restent cinq
observations justes.

**Robustesse de l'appel** :
- Un refus du modèle arrive avec un code de succès, pas une erreur : il faut vérifier le
  motif d'arrêt avant de lire le contenu.
- La réponse est relue : titre, action et au moins une observation sont exigés — une réponse
  arrêtée au plafond de sortie peut arriver amputée malgré le contrat.
- La consigne fixe est placée en tête de requête et mise en cache entre appels ; elle est
  assez longue pour que la mise en cache soit acceptée. Sur un rafraîchissement isolé,
  l'écriture du cache est payée sans être relue — perte négligeable devant le gain du
  balayage hebdomadaire, où les appels se suivent.
- Un plafond de jetons de sortie borne le coût d'un appel quoi qu'il arrive, en englobant
  l'éventuel raisonnement facturé en sortie.
- Les erreurs sont distinguées du plus précis au plus général : une limite de débit se
  réessaie, une clé invalide jamais.

**Trois garde-fous de dépense, superposés du plus fin au plus grossier** :
1. **Verrou de fréquence** : une heure entre deux rédactions, mesurée sur la **dernière
   écriture** et non la création — une lecture du jour étant remplacée en place, se fonder
   sur la création laisserait passer une rédaction par minute une fois la première heure
   écoulée.
2. **Plafond journalier** : trois rédactions par membre et par jour. La lecture porte sur
   vingt-huit jours ; les chiffres ne bougent pas assez en une journée pour qu'une quatrième
   dise autre chose que la troisième.
3. **Plafonds mensuels** en euros, par membre et pour l'ensemble des comptes. C'est le filet,
   et le seul qui tienne si les autres se révèlent troués.

Le contrôle précède l'appel : la dépense peut donc franchir le plafond **d'un appel**, jamais
davantage — le borner exactement supposerait de connaître le coût avant de l'engager, ce que
la longueur de la réponse interdit. Le coût est calculé à partir des jetons rapportés,
**cache compris** (un plafond qui ignorerait le cache se tromperait dans le sens de la
générosité), avec une précision de l'ordre du millionième d'euro — arrondir au centime ferait
compter zéro à des milliers d'appels. Il est figé à l'écriture, les tarifs pouvant changer, et
enregistré **dès que l'interface a répondu — avant même de relire sa réponse** : une réponse
illisible a été facturée comme une autre, et ne compter que les appels réussis reviendrait à
ignorer exactement les moins inquiétants. Un échec d'écriture au registre ne prive pas le
membre de sa lecture, mais est journalisé comme une erreur. Le mois est celui du **membre**,
pas du serveur : un compte à l'autre bout du monde verrait sinon son plafond repartir avec un
jour de décalage. Les messages de refus sont distincts pour chacun des trois plafonds, et
celui du plafond global ne nomme pas les autres comptes.

Le bouton de rafraîchissement n'est actif que si les trois freins l'autorisent : un bouton
actif qui rendrait un refus serait pire que pas de bouton. Sans clé d'interface configurée,
le module annonce son indisponibilité — et **avant tout autre diagnostic**, sinon un serveur
sans clé répondrait « pas assez de données » à un membre au journal vide, l'envoyant saisir
des repas pour débloquer une fonction qui n'existe pas chez lui.

Génération automatique : balayage horaire, créneau au lendemain de la pesée à l'heure des
messages de motivation — la lecture arrive quand la pesée de la semaine vient d'entrer dans
les chiffres. Une lecture existant déjà pour ce jour fait sauter le membre. Un membre en
échec ne prive pas les suivants.

### 4.23 Validation générale des entrées

- Toute requête est validée par liste blanche : **un champ non déclaré fait échouer la
  requête**, il n'est pas silencieusement ignoré.
- Toute date de journée doit respecter le format `AAAA-MM-JJ`, sinon refus.
- Tout code-barres fait 6 à 14 chiffres.
- Les listes d'identifiants sont bornées à cent éléments et ne peuvent pas être vides quand
  elles sont fournies.
- Les valeurs hors bornes (âge aberrant, poids impossible, quantité nulle) sont refusées avec
  un message en français désignant le champ fautif.
- Les erreurs de validation, d'autorisation, d'introuvable et de conflit sont distinctes et
  portent chacune un message lisible par un humain.
- Aucune conversion de type implicite : une chaîne là où un nombre est attendu est refusée,
  sauf conversion explicitement déclarée pour les paramètres d'URL.

### 4.24 Comportements en cas d'erreur

| Situation | Comportement attendu |
|---|---|
| Base de produits injoignable au scan | Fiche périmée servie si elle existe ; sinon message d'indisponibilité |
| Base de produits injoignable à la recherche | Résultats personnels seuls, sans erreur bloquante |
| Réseau coupé côté client | Bandeau « hors ligne », session préservée, tentatives à intervalle croissant (3 s, 6 s, 12 s, 24 s, puis 30 s), rechargement des écrans au retour |
| Session réellement expirée | Retour à l'écran de connexion, état local vidé |
| Notification impossible | L'action qui l'a déclenchée aboutit quand même |
| Synchronisation externe en échec | Erreur lisible stockée et affichée sur la connexion ; les autres connexions sont traitées |
| Un membre en échec dans un balayage | Les suivants sont traités |
| Modèle de rédaction indisponible | Message d'indisponibilité explicite, jamais un faux diagnostic de données insuffisantes |
| Caméra refusée ou indisponible | Saisie manuelle du code-barres proposée |
| Fuseau horaire inconnu | Le membre est ignoré par les envois planifiés, plutôt que notifié à une heure arbitraire |
| Origine refusée par la politique d'origines croisées | Refus **journalisé côté serveur** : sans cela, le navigateur bloque et rien ne dit pourquoi |

---

## 5. INTERFACES

### 5.1 Écrans et parcours

**Navigation principale — cinq destinations maximum**, au-delà une barre inférieure devient
illisible : Journal · Activité · **Alerte** · Tendances · Profil. Barre inférieure sur
mobile, colonne latérale à partir du grand écran. « Alerte » occupe le **rang central**, le
plus facile à atteindre au pouce, et conserve ses couleurs d'alerte au repos : c'est le seul
écran qu'on ouvre dans l'urgence, déjà debout devant le placard. Une seule destination porte
ce traitement — deux boutons rouges dans la même barre et plus aucun ne signale quoi que ce
soit. Le scan **n'y figure pas** : il s'atteint depuis l'en-tête du journal et depuis chaque
repas, toujours avec le repas et le jour en contexte, ce qu'un onglet global ne pourrait pas
transmettre.

| Écran | Contenu | État vide | État d'erreur |
|---|---|---|---|
| **Connexion / Inscription** | Formulaire, bascule vers l'autre mode | — | Message unique pour identifiants faux |
| **Accueil** (4 étapes) | Fil d'étapes, formulaires, aperçu des objectifs à la dernière étape ; **pas de barre de navigation** — il n'y a qu'un chemin, et pouvoir en sortir reviendrait à contourner le questionnaire | — | Champ invalide signalé sur place |
| **Journal** | Sélecteur de jour, anneau calorique, barres de macros, 4 cartes de repas, boutons d'ajout et de reprise, boîte de réception des envois | « Aucun aliment » par repas + invitation à scanner | Bandeau hors ligne ; squelettes de chargement |
| **Scan** | Flux caméra, cadre de visée, saisie manuelle en repli | — | Caméra refusée → saisie manuelle ; code inconnu → proposition de création |
| **Activité** | Formulaire de séance avec estimation en direct, liste des séances du jour, total | « Aucune séance » | Champ invalide signalé |
| **Alerte fringale** | Fil à trois étapes (Souffler / Chiffrer / Décider), arguments, sélecteur d'aliment, roue, résultat, écran de fin | Argument de repli si aucune donnée | Produit introuvable signalé sans quitter le parcours |
| **Tendances** | Sélecteur 7/14/30 j, graphique empilé avec légende cliquable, tableau dépliable, moyennes, calendrier mensuel, comparaison de périodes, courbe de poids + poids théorique, analyse des écarts, lecture hebdomadaire | « Pas encore de données » avec invitation à saisir | Bloc d'analyse muet **et expliqué** quand les données ne permettent pas de conclure |
| **Profil** | Objectifs calculés, informations personnelles, curseur de part recréditée, créneau de pesée, motivation, notifications, proches, connexions externes, apparence, version | Listes vides explicitées | Erreur de connexion externe affichée sur la carte concernée |
| **Administration** | Cartes de statistiques, histogramme 14 jours, palmarès, recherche de compte, envoi d'essai | — | Refus muet pour un non-administrateur |

**Parcours clés**
- Inscription → accueil obligatoire → journal.
- Journal → bouton de repas → (scanner | chercher | reprendre) → portion → retour au journal
  avec totaux à jour.
- Journal → glisser une ligne sur un autre repas → totaux à jour.
- Alerte → Souffler → Chiffrer → Décider → (collation enregistrée | rien) → retour au journal.
- Envoi reçu → journal → fenêtre d'acceptation (cases, facteur global, jour, repas) → lignes
  ajoutées.
- Profil → connecter un service → page du fournisseur → retour avec statut → première
  synchronisation.

**Accessibilité et confort** : toute information portée par une couleur est doublée d'un
texte ; le graphique a une alternative en tableau ; les cibles tactiles font au moins 44 px ;
les états de chargement sont des squelettes, pas des écrans blancs ; le retour tactile est
un bonus jamais porteur d'information ; l'écran d'ouverture tient jusqu'à la restauration de
session, pour que l'écran de connexion ne clignote jamais devant un utilisateur déjà connecté.

**Écrans lourds chargés à la demande** : le lecteur de codes-barres, les graphiques, le
parcours d'alerte et la console d'administration ne doivent pas peser sur le premier
chargement de tous les autres.

### 5.2 Points d'entrée d'API

Toutes les routes sont préfixées par une version (`/v1`). Sauf mention contraire, elles
exigent un jeton d'accès en en-tête `Authorization: Bearer …` et répondent 401 sans lui.
Codes transverses : 400 validation, 401 non authentifié, 403 interdit, 404 introuvable,
409 conflit, 429 débit dépassé, 503 dépendance externe indisponible.

#### Authentification (publique)

| Verbe | Chemin | Requête | Réponse | Codes |
|---|---|---|---|---|
| POST | `/auth/register` | `{email, password, displayName}` | `{accessToken, refreshToken, user}` | 201 · 400 · 409 |
| POST | `/auth/login` | `{email, password}` | idem | 200 · 400 · 401 |
| POST | `/auth/refresh` | `{refreshToken}` | idem | 200 · 401 |
| POST | `/auth/logout` | — | vide | 204 · 401 |

#### Profil

| Verbe | Chemin | Requête | Réponse | Codes |
|---|---|---|---|---|
| GET | `/me` | — | profil public | 200 |
| PATCH | `/me` | champs partiels du profil | profil public | 200 · 400 |
| POST | `/me/onboarding` | profil complet + nom + créneau | profil public | 201 · 400 |
| POST | `/me/targets-preview` | profil complet | `{targets, bmi, bmiCategory}` | **200** (calcul, pas écriture) · 400 |

Le **profil public** contient : identifiant, e-mail, nom, sexe, âge, taille, poids, poids
visé, niveau d'activité, but, objectif manuel, part recréditée, IMC et catégorie, objectifs
calculés (calories, protéines, glucides, lipides, métabolisme de base, dépense totale),
créneau de pesée (sans la trace interne du dernier envoi), réglage de motivation (actif,
heure), indicateur d'administration, indicateur de fin d'accueil. **Jamais** l'empreinte du
mot de passe ni celle du jeton de rafraîchissement.

#### Aliments

| Verbe | Chemin | Requête | Réponse | Codes |
|---|---|---|---|---|
| GET | `/foods/barcode/{code}` | — | fiche produit | 200 · 400 · 404 · 503 |
| GET | `/foods/search?q=&limit=` | `q` ≥ 2 car., `limit` 1–50 (défaut 20) | liste de fiches | 200 · 400 |
| GET | `/foods/custom` | — | aliments personnalisés | 200 |
| POST | `/foods/custom` | `{name, brand?, quantity?, nutriments}` | fiche produit | 201 · 400 |

#### Journal

| Verbe | Chemin | Requête | Réponse | Codes |
|---|---|---|---|---|
| GET | `/journal?day=` | jour facultatif (défaut : aujourd'hui) | `{day, meals[4], totals}` | 200 · 400 |
| GET | `/journal/suggestions?mealType=&mode=&limit=` | `mode` ∈ {frequent, recent}, `limit` 1–50 (défaut 12) | suggestions | 200 · 400 |
| GET | `/journal/meals/history?mealType=&excludeDay=&limit=` | `limit` 1–30 (défaut 10) | journées + lignes | 200 · 400 |
| POST | `/journal/meals/copy` | `{fromDay, fromMealType, toDay, toMealType, entryIds?}` | lignes créées | 201 · 400 · 404 |
| POST | `/journal/entries` | `{day, mealType, quantityG, foodId? \| barcode? \| (name + nutrimentsPer100g)}` | ligne | 201 · 400 |
| PATCH | `/journal/entries/{id}` | `{quantityG?, mealType?}` | ligne | 200 · 400 · 404 |
| DELETE | `/journal/entries/{id}` | — | vide | 204 · 404 |

Chaque ligne renvoyée porte ses **totaux proratisés**, en plus de l'instantané pour 100 g.

#### Activités

| Verbe | Chemin | Requête | Réponse | Codes |
|---|---|---|---|---|
| GET | `/activities/catalog` | — | 36 entrées `{key, label, category, met}` | 200 |
| GET | `/activities?day=` | — | `{day, entries, totalCaloriesBurned, totalDurationMin}` | 200 · 400 |
| POST | `/activities/entries` | `{day, activityKey, durationMin, met?, caloriesBurned?, label?, note?}` | séance | 201 · 400 |
| PATCH | `/activities/entries/{id}` | `{durationMin?, caloriesBurned?, note?}` | séance | 200 · 400 · 404 |
| DELETE | `/activities/entries/{id}` | — | vide | 204 · 404 |

#### Poids

| Verbe | Chemin | Requête | Réponse | Codes |
|---|---|---|---|---|
| GET | `/weights?from=&to=&limit=` | `limit` 1–500 (défaut 200) | `{entries[], latest, totalDeltaKg}` | 200 · 400 |
| POST | `/weights` | `{day, weightKg, note?}` | point de poids | 201 · 400 |
| DELETE | `/weights/{id}` | — | vide | 204 · 404 |

#### Tableau de bord

| Verbe | Chemin | Réponse |
|---|---|---|
| GET | `/dashboard?day=` | `{day, targets, consumed, burned, activityCaloriePercent, burnedCredit, remainingKcal, macros{proteins, carbohydrates, fat}{current, target, percent}, meals, activities, activityTotals}` |
| GET | `/dashboard/trend?from=&to=` | `{from, to, targetKcal, days[], averages, trackedDays}` — chaque jour : `{day, consumedKcal, burnedKcal, burnedCreditKcal, activityCaloriePercent, targetKcal, netKcal, proteins, carbohydrates, fat, hasData}` |
| GET | `/dashboard/weight-projection?to=` | `{from, to, kcalPerKg, trackedDays, points[{day, weightKg}]}` |
| GET | `/dashboard/weight-analysis?to=&days=` | `{status, from, to, days, trackedDays, coverage, segments, realDeltaKg, theoreticalDeltaKg, deviationKg, dailyGapKcal, dailyGapMarginKcal, averageBurnKcal, averageCreditedKcal, activityCaloriePercent, suggestions[]}` |

`targetKcal` de la série est celui du **dernier jour** de la période. Les moyennes ne portent
que sur les journées renseignées. Une suggestion porte : nature (`activity-percent`,
`activity-level`, `intake`), phrase prête à afficher, valeur courante et valeur proposée
quand le réglage est chiffré, unité, niveau de confiance.

#### Alerte fringale

| Verbe | Chemin | Réponse |
|---|---|---|
| GET | `/craving/summary?day=` | `{day, targetKcal, consumedKcal, burnedKcal, burnedCreditKcal, remainingKcal, weightKg, encouragements[{key, title, body}]}` |

Le poids est transmis pour que le client puisse convertir des calories en minutes d'effort
avec **la même formule** que le journal d'activité.

#### Lecture hebdomadaire

| Verbe | Chemin | Réponse | Codes |
|---|---|---|---|
| GET | `/insights/weekly` | `{status, day, headline, observations[], focus, windowDays, trackedDays, canRefresh, generatedAt}` ; `status` ∈ {ready, none, insufficient-data, disabled} | 200 |
| POST | `/insights/weekly/refresh` | idem | 200 · 403 (plafond) · 503 (non configuré, verrou horaire, échec) |

La lecture n'est **jamais** déclenchée par un GET : l'écran ne paie pas.

#### Proches et partage

| Verbe | Chemin | Requête / Réponse | Codes |
|---|---|---|---|
| GET | `/contacts` | liste de proches | 200 |
| POST | `/contacts` | `{email}` | 201 · 400 · 404 · 409 · 429 |
| POST | `/contacts/{id}/accept` | — | **200** · 404 |
| PATCH | `/contacts/{id}` | `{shareMeals}` | 200 · 404 |
| DELETE | `/contacts/{id}` | — | 204 · 404 |
| GET | `/contacts/{id}/meals/history?mealType=` | journées du proche | 200 · 403 · 404 |
| POST | `/contacts/{id}/meals/copy` | `{fromDay, fromMealType, toDay, toMealType, items[{entryId, quantityG}]}` | 201 · 400 · 403 · 404 |
| POST | `/shares` | `{contactId, fromDay, fromMealType, entryIds[]}` → `{id, itemCount}` | 201 · 400 · 404 |
| GET | `/shares/inbox` | envois en attente | 200 |
| POST | `/shares/{id}/accept` | `{day, mealType, items[{index, quantityG}]}` → lignes créées | **200** · 400 · 404 · 409 |
| POST | `/shares/{id}/decline` | — | 204 · 404 |

Un proche est renvoyé sous la forme : `{id, status, direction: incoming|outgoing,
person{id, displayName, email}, iShareMeals, theyShareMeals}`. Un envoi reçu porte
l'expéditeur, le jour et le repas d'origine, ses dates de création et d'expiration, et ses
aliments indexés par position, avec quantité, énergie proratisée et énergie pour 100 g.

#### Notifications

| Verbe | Chemin | Requête / Réponse | Codes |
|---|---|---|---|
| GET | `/push/public-key` | → `{publicKey \| null, devices}` | 200 |
| POST | `/push/subscribe` | `{endpoint, keys{p256dh, auth}, userAgent?}` | 204 · 400 |
| DELETE | `/push/subscribe` | `{endpoint}` | 204 · 400 |

Une clé publique nulle signale au client que le serveur n'a pas de clés : il masque l'option.

#### Services externes

| Verbe | Chemin | Requête / Réponse | Codes |
|---|---|---|---|
| GET | `/integrations` | liste `{provider, displayName, available, connected, accountName?, lastSyncAt?, lastError?, importedCount}` | 200 |
| POST | `/integrations/{provider}/authorize` | → `{url}` | 200 · 400 · 503 |
| GET | `/integrations/{provider}/callback?code=&state=&error=` | **publique**, répond par une **redirection** vers `<front>/profil?integration=…&statut=ok\|refus\|erreur` | 302 |
| POST | `/integrations/{provider}/sync` | → `{provider, imported, duplicates}` | 201 · 404 · 503 |
| DELETE | `/integrations/{provider}` | → `{disconnected: true}` | 200 · 404 |

Un fournisseur inconnu répond 400.

#### Administration et santé

| Verbe | Chemin | Requête / Réponse | Codes |
|---|---|---|---|
| GET | `/admin/stats` | statistiques complètes | 200 · 401 · 403 |
| GET | `/admin/users?q=&limit=` | `limit` 1–200 | 200 · 401 · 403 |
| POST | `/admin/push-test` | `{userId, title?, body?}` → `{delivered}` | 201 · 401 · 403 · 404 |
| GET | `/health` | **publique** → `{status: ok\|degraded, mongo: up\|down, timestamp}` | 200 |

Une documentation interactive de l'API est exposée par le serveur.

---

## 6. CONTRAINTES TECHNIQUES

### 6.1 Contraintes réelles

- **Base de données orientée documents avec index uniques, index uniques partiels et
  expiration automatique de documents.** Trois mécanismes en dépendent directement :
  l'idempotence de la synchronisation externe (index unique restreint aux lignes qui portent
  un identifiant externe), la purge des envois expirés sans tâche planifiée, et l'unicité
  d'une pesée ou d'un jeu d'objectifs par jour. Une base relationnelle conviendrait, au prix
  d'un index partiel et d'une tâche de purge écrits à la main. Les instantanés nutritionnels
  et le document remis au modèle de rédaction sont des objets imbriqués : une base
  documentaire les accueille naturellement.
- **Processus persistant pour les tâches planifiées.** Le rappel de pesée, les messages de
  motivation, la lecture hebdomadaire automatique et la synchronisation externe reposent sur
  des balayages horaires (et bihoraires). **Sur une exécution sans état, déclenchée uniquement
  par les requêtes, aucun de ces quatre mécanismes ne fonctionne.** Le reste de l'application
  y fonctionne parfaitement. Une régénération qui vise ce mode d'hébergement doit remplacer
  les balayages par un ordonnanceur externe appelant des routes dédiées.
- **Contexte sécurisé obligatoire pour la caméra.** Le scan exige HTTPS, ou l'hôte local en
  développement. Jamais en HTTP simple sur une adresse de réseau local.
- **En-tête de politique de permissions autorisant la caméra sur sa propre origine.** Les
  jeux d'en-têtes de sécurité tout faits interdisent presque toujours la caméra, ce qui casse
  le scan **sans aucun message d'erreur**.
- **Base ouverte de produits alimentaires**, sans clé d'API, sous licence libre. Elle exige
  un contact réel dans l'en-tête d'identification du client. Deux services distincts sont
  utilisés : la fiche produit par code-barres, et un service de recherche plein texte séparé
  — les points d'entrée historiques de recherche répondent en erreur sur le serveur principal.
- **Deux secrets de signature distincts** pour les jetons d'accès et de rafraîchissement.
- **Clé de chiffrement symétrique de 256 bits** pour les jetons OAuth. Sans elle, les
  intégrations doivent se désactiver, pas fonctionner en clair.
- **Confiance restreinte aux en-têtes de proxy**, sur les plages privées et la boucle locale
  uniquement, sinon les limites de débit s'effondrent ou s'appliquent à tous les clients
  confondus.
- **Fuseaux horaires réels** : les journées locales et les créneaux d'envoi doivent être
  calculés dans le fuseau du membre, pas celui du serveur. Un conteneur tourne en temps
  universel ; entre minuit et deux heures à Paris, la date du serveur désigne encore la veille.
  Un fuseau inconnu du système doit être détecté et traité, pas ignoré silencieusement.
- **Application web installable** : manifeste, icônes classiques et masquables, agent de
  service mettant en cache **la seule coquille applicative**. Les données ne sont jamais mises
  en cache — un journal périmé afficherait des totaux faux. La portée du cache est limitée aux
  requêtes de lecture de même origine **hors chemin d'API** ; cette exclusion n'est pas
  théorique dès lors que l'API est servie sous la même origine. Les navigations vont au
  réseau d'abord, avec repli sur la coquille : un document d'entrée périmé continuerait sinon
  de pointer vers des fichiers supprimés. Un numéro de version permet de purger les caches.
  Hors ligne, l'application se lance et affiche l'écran de connexion ou le dernier profil
  connu ; les données, elles, exigent le serveur.

### 6.2 Variables de configuration nécessaires

| Variable | Rôle | Absence |
|---|---|---|
| Chaîne de connexion à la base | obligatoire | démarrage impossible |
| Secret de signature des jetons d'accès | obligatoire | valeur de développement, inutilisable en production |
| Secret de signature des jetons de rafraîchissement | obligatoire | idem |
| Durées de validité des deux jetons | défauts 15 min / 30 jours | défauts appliqués |
| Origines autorisées (liste) | obligatoire en production | défaut développement ; **une chaîne vide doit être traitée comme absente**, sinon la liste devient vide et toutes les requêtes navigateur sont refusées en silence |
| Contact d'identification pour la base de produits | exigé par la politique d'usage | défaut générique |
| Liste blanche d'administration | facultative | **console fermée à tous** |
| Clés de notification (publique, privée, sujet) | facultatives | notifications désactivées, annoncé au démarrage |
| Clé de chiffrement des jetons OAuth | facultative | **intégrations désactivées** |
| Base publique de l'API vue du fournisseur | requise si intégrations | — |
| Base du front pour la redirection de retour | requise si intégrations | — |
| Identifiants OAuth de chaque fournisseur | facultatifs, par fournisseur | ce fournisseur se désactive seul |
| Clé d'accès au modèle de rédaction | facultative | lecture hebdomadaire désactivée, annoncé au démarrage |
| Plafonds mensuels de dépense (global, par membre) | défauts 20 € et 5 € | défauts appliqués |
| Port d'écoute | défaut 3001 | — |
| Base de l'API vue du navigateur (côté front) | vide = même origine | — |

**Principe de dégradation** : chaque dépendance optionnelle absente **désactive proprement
sa fonction, l'annonce au démarrage, et le client masque l'option** — plutôt que d'échouer à
chaque appel ou de proposer un réglage sans effet.

### 6.3 Intégrations externes

- **Base ouverte de produits** : fiche par code-barres, recherche plein texte, complétion par
  taxonomie de catégories. Délai d'attente de quelques secondes, champs demandés restreints
  pour accélérer la réponse. Licence à mentionner, contact réel à déclarer.
- **Deux services de santé / sport** exposant une interface OAuth 2 avec code d'autorisation,
  URI de redirection à déclarer **exactement**, périmètres de lecture d'activité seulement.
  L'un des deux classe son périmètre comme « restreint » et exige une revue de confidentialité
  avant mise en production ; en mode test, il fonctionne immédiatement pour des comptes
  déclarés comme testeurs, ce qui suffit à un usage personnel. L'autre n'accepte qu'un domaine
  nu comme rappel, sans protocole ni chemin.
- **Service de notifications du navigateur** avec paire de clés d'identification du serveur.
- **Modèle de langage** pour la lecture hebdomadaire, avec sortie structurée par schéma, mise
  en cache du préfixe de consigne, et rapport de consommation de jetons incluant le cache.

### 6.4 Choix d'opportunité du premier auteur — remplaçables

Rien de ce qui suit n'est imposé par le comportement décrit :

- Le langage et les cadriciels retenus (serveur à modules et injection de dépendances côté
  API, bibliothèque à composants côté interface).
- La bibliothèque de composants d'interface et le système de classes utilitaires.
- La bibliothèque de graphiques, celle de lecture de codes-barres, celle de glisser-déposer,
  celle de gestion de formulaires et de validation côté client.
- Le découpage en deux projets indépendants (interface et API) avec leurs déploiements
  séparés, et le service de déploiement retenu.
- Le serveur web en frontal qui sert l'interface, réécrit le chemin d'API et gère le repli de
  navigation ; sa configuration doit rester **l'unique définition du routage applicatif** —
  un proxy mutualisé en amont ne doit redéfinir aucune règle applicative, sous peine de voir
  deux définitions diverger.
- Le choix d'exposer l'API sous la même origine que l'interface (ce qui rend la configuration
  d'origines autorisées inutile) plutôt que sur un domaine séparé.
- La palette de couleurs, la typographie, les libellés, les animations de transition entre
  écrans.
- Le modèle de langage précis, son niveau d'effort de raisonnement, et le repli automatique
  vers un autre modèle en cas de refus.
- La granularité horaire des balayages planifiés.
- Le stockage local des jetons et du dernier profil connu côté navigateur.

---

## 7. CRITÈRES D'ACCEPTATION

Assertions vérifiables une par une, sur une instance fraîchement installée.

### 7.1 Santé et authentification

1. La sonde de disponibilité répond 200 et annonce la base comme connectée.
2. Une inscription répond 201 et renvoie un jeton d'accès, un jeton de rafraîchissement et un
   profil.
3. L'adresse renvoyée dans le profil est en minuscules, quelle que soit la casse saisie.
4. Le profil renvoyé à l'inscription porte un objectif calorique supérieur à 1000 kcal.
5. Une seconde inscription avec la même adresse répond 409.
6. Un mot de passe de moins de 8 caractères répond 400.
7. Une connexion avec un mot de passe faux répond 401 ; une connexion avec une adresse
   inexistante répond **401 avec le même message**.
8. Un appel à une route protégée sans jeton répond 401.
9. Un rafraîchissement avec un jeton valide répond 200 et renvoie un **nouveau** jeton de
   rafraîchissement ; l'ancien ne fonctionne plus.
10. Le profil ne contient jamais l'empreinte du mot de passe ni celle du jeton de
    rafraîchissement.

### 7.2 Calculs

11. Pour un homme de 40 ans, 180 cm, 82 kg, niveau « modérément actif », but « perte » :
    métabolisme de base **1775**, dépense totale **2751**, objectif **2201**, protéines
    **164 g**, IMC **≈ 25,3**.
12. Un âge de 5 ans ou de 200 ans répond 400.
13. L'aperçu d'objectifs répond **200**, renvoie un objectif et un IMC, et **ne modifie rien**
    au profil enregistré.
14. Une séance de 45 minutes d'une activité d'intensité 9,8 pour un poids de 82 kg produit
    une dépense de **633 kcal**.
15. Une activité hors catalogue sans intensité ni dépense répond 400.
16. Une dépense saisie à la main survit à une modification de la durée.

### 7.3 Produits et journal

17. Le scan d'un code-barres réel répond 200, avec un nom non vide, une énergie renseignée et
    un Nutri-Score quand la base le fournit.
18. Un second scan du même code est servi depuis le cache local et renvoie **le même
    identifiant de fiche**.
19. Un code-barres inconnu répond 404 ; un code-barres malformé répond 400.
20. Une recherche par nom répond 200 avec au moins un résultat.
21. La requête de recherche demande explicitement la langue française et un filtre de marché ;
    les caractères réservés saisis par l'utilisateur sont neutralisés avant l'envoi.
22. Sur un jeu de résultats contenant « Sauce tomate », « Tomate » et « Tomates cerises », le
    classement place **« Tomate » en premier** et **« Tomates cerises » en second**.
23. Une recherche au pluriel rencontre le libellé au singulier, et réciproquement.
24. Une recherche dont le dernier mot est un fragment (« chocol ») fait remonter en tête les
    résultats du mot complété, tout en conservant les résultats bruts derrière.
25. L'ajout de 30 g d'un produit à 539 kcal/100 g produit un total de **161,7 kcal**
    (proratisation à une décimale).
26. L'ajout d'un aliment personnalisé à 110 kcal/100 g, pour 200 g, produit **220 kcal**.
27. Une entrée sans identifiant, sans code-barres et sans valeurs nutritionnelles répond 400.
28. Un format de date invalide répond 400.
29. Le bilan du jour renvoie **quatre** groupes de repas, même vides.
30. Supprimer une ligne recalcule immédiatement les totaux du jour.

### 7.4 Copie et reprise

31. L'historique d'un repas annonce, pour la journée source, le bon nombre de lignes, les bons
    noms et les bonnes calories.
32. La copie d'un repas répond 201 et renvoie les lignes créées.
33. Les lignes copiées **s'ajoutent** à celles déjà présentes dans le repas de destination.
34. Une copie restreinte à une sélection ne recopie que la sélection.
35. Une sélection **vide** répond 400.
36. Un identifiant de ligne appartenant à un autre repas, un autre jour ou un autre compte
    répond 404 sans rien copier.

### 7.5 Budget et objectifs figés

37. Avec un objectif de 2201 kcal et 633 kcal dépensés, la part recréditée par défaut est de
    **25 %** et la part ajoutée au budget vaut **158 kcal**.
38. Le reste du jour vaut exactement `objectif + part recréditée − apports`.
39. Une série sur 7 jours renvoie **7** journées, et le nombre de journées renseignées est
    exact.
40. La dépense brute et la dépense recréditée apparaissent séparément dans la série
    journalière.
41. Chaque journée de la série porte **son propre** objectif.
42. Après une pesée en baisse : l'objectif du profil **et** celui du jour passent à la nouvelle
    valeur ; **l'objectif de la veille et ceux des jours plus anciens restent inchangés** ;
    l'objectif annoncé pour la période est celui du **dernier** jour.
43. Après avoir porté la part recréditée à 50 % : la journée du jour applique 50 %, **la veille
    conserve 25 %**.
44. Supprimer la pesée la plus récente ramène le poids du profil à la pesée précédente et
    recalcule l'objectif en conséquence.
45. Une tentative de modifier le poids depuis le profil, alors qu'une pesée existe, répond 400
    avec un message qui indique où se fait la pesée ; les autres champs restent modifiables et
    le poids reste inchangé.

### 7.6 Poids théorique et analyse

46. La courbe théorique part de la première pesée et compte **un point par jour** jusqu'au
    jour transmis, inclus.
47. Le nombre de journées comptées exclut la journée en cours.
48. Une journée sans repas saisi ne fait pas varier la courbe.
49. Sur une série cohérente de soixante journées et six intervalles, le verdict est **aligné**
    et **aucune** suggestion n'est émise.
50. Sur une série où la perte réelle est de −3,12 kg pour une perte annoncée de −3,90 kg, le
    verdict est **plus lent que prévu**, l'écart cumulé vaut **0,78 kg** et l'écart quotidien
    **100 kcal**.
51. Une suggestion de part recréditée tombe sur un multiple de 5 et reste entre 0 et 100.
52. Un écart inférieur à la marge de bruit est déclaré **aligné**.
53. Une couverture insuffisante ou une seule pesée donne un statut de **données
    insuffisantes** et zéro intervalle retenu.

### 7.7 Proches et partage

54. Inviter une adresse sans compte répond 404 ; s'inviter soi-même répond 400 ; inviter deux
    fois répond 409.
55. Envoyer un repas à un proche dont l'invitation n'est pas acceptée répond 404.
56. Un envoi de deux aliments répond 201 et annonce **2** éléments.
57. **Le journal du destinataire reste vide tant qu'il n'a pas accepté.**
58. La boîte de réception du destinataire contient l'envoi, avec le nom de l'expéditeur et ses
    deux aliments.
59. L'expéditeur ne peut pas accepter à la place du destinataire (404).
60. Une position d'aliment hors de l'envoi répond 400.
61. L'acceptation partielle n'ajoute que les lignes retenues, **aux portions choisies par le
    destinataire**.
62. Rejouer l'acceptation répond 404.
63. Refuser un envoi répond 204 et vide la boîte de réception.
64. Tenter de reprendre les repas d'un proche qui ne les a pas ouverts répond **403**.
65. Après ouverture, le proche voit les journées et peut reprendre une ligne **à sa propre
    portion** ; la permission ne vaut que dans un sens (403 en sens inverse).
66. Retirer un proche répond 204, **supprime les envois en attente**, et tout nouvel envoi
    répond ensuite 404.

### 7.8 Cloisonnement

67. Le journal d'un second compte est vide alors que le premier contient des lignes.
68. Supprimer une ligne appartenant à un autre compte répond 404.
69. La console d'administration répond 403 à un membre ordinaire et 401 sans jeton.
70. Le profil d'un compte autorisé porte l'indicateur d'administration ; celui d'un membre
    ordinaire ne le porte pas.

### 7.9 Accueil

71. Un compte fraîchement créé porte l'indicateur « accueil non rempli ».
72. Un questionnaire incomplet répond 400.
73. Un questionnaire complet répond 201, marque l'accueil comme rempli, reprend le profil
    transmis, règle le créneau de pesée, et **crée la première pesée au poids saisi**.
74. Un second envoi du même questionnaire répond 201, **ne réécrit pas le poids** et **ne crée
    pas de seconde pesée**.

### 7.10 Alerte fringale et motivation

75. Le bilan d'alerte répond 200 sur un compte neuf et renvoie au moins un argument.
76. Les apports et la dépense du jour y sont exacts, et le reste y est calculé **exactement
    comme** dans le bilan du jour.
77. Une date invalide répond 400.
78. Un message de motivation est toujours produit, même sur un compte au journal vierge.
79. Deux tirages consécutifs ne rendent pas la même formulation ; le troisième ne rend pas la
    même règle que le premier.
80. Une absence de plus de trois jours prime sur les félicitations.
81. Mémoire saturée : un message sort quand même.

### 7.11 Statistiques de suivi

82. Sur une fenêtre de sept jours comportant un trou, les journées renseignées, les journées
    tenues, la série sous objectif et la série de suivi sont comptées exactement, **et la
    journée en cours n'apparaît nulle part**.
83. Le déficit cumulé est la somme des écarts à l'objectif des seules journées renseignées.
84. Un journal vierge est distingué d'une longue absence.
85. Une comparaison de semaines dont l'une compte moins de deux journées ne produit **aucune**
    valeur.

### 7.12 Services externes

86. Le chiffré d'un jeton ne laisse pas apparaître le jeton en clair ; le déchiffrement est
    réversible ; deux chiffrements du même jeton diffèrent ; une altération du chiffré est
    **détectée**.
87. L'URL d'autorisation porte un état signé ; un jeton d'accès présenté comme état est
    **refusé** ; un état émis pour un fournisseur est refusé sur l'autre.
88. Quand le fournisseur omet le jeton de rafraîchissement, celui déjà en base est **conservé**.
89. Une séance importée l'est une fois ; un cycle rejoué n'en importe aucune de plus ; la
    dépense mesurée est conservée et la séance est marquée comme relevée, avec sa provenance.
90. Deux séances de services différents démarrant à moins de quinze minutes d'écart : une
    seule est conservée, l'autre est comptée comme doublon.
91. Deux séances à des horaires éloignés ne sont pas confondues.
92. Une séance importée puis supprimée à la main **ne revient pas** au cycle suivant, et son
    identifiant reste connu.

### 7.13 Lecture hebdomadaire

93. Sans clé d'accès au modèle, la lecture répond 200 avec le statut « désactivé » et **zéro**
    observation — jamais un diagnostic de données insuffisantes.
94. Le document remis au modèle exclut les journées vierges des moyennes, compte exactement les
    journées renseignées, et calcule la couverture (par exemple 10 journées sur 14 → 0,71).
95. Sur un compte de trois jours, la profondeur annoncée est **3**, pas 28.
96. Le coût d'un appel est calculé en incluant les jetons lus et écrits en cache.
97. Le mois est déduit du jour local du membre.
98. Un compte neuf n'est bloqué par aucun plafond ; le plafond journalier, le plafond par membre
    et le plafond global bloquent chacun dans l'ordre attendu, avec un message distinct.
99. Le schéma de sortie n'utilise aucune contrainte refusée par l'interface de génération
    structurée.

### 7.14 Interface

100. Sans profil rempli, toute page redirige vers le questionnaire d'accueil, y compris par
     saisie directe d'une URL ; inversement, un questionnaire déjà rempli ne se repropose pas.
101. Le journal interdit de naviguer vers une date future.
102. Le graphique de tendances propose une alternative en tableau.
103. Cliquer sur une légende de macronutriment réordonne les colonnes empilées.
104. Une coupure réseau affiche un bandeau, conserve la session, et les écrans se rechargent
     seuls au retour du réseau.
105. L'application s'ouvre hors ligne sur son écran de connexion ou son dernier profil connu ;
     **aucune donnée de journal n'est servie depuis le cache**.
106. Le manifeste déclare des icônes classiques et masquables et rend l'application installable.
107. Le parcours d'alerte peut être quitté à chaque étape sans que rien ne soit inscrit au
     journal.
108. Le lecteur de codes-barres, les graphiques, le parcours d'alerte et la console
     d'administration ne sont pas chargés au premier affichage.

---

## Points à confirmer par l'auteur

Les éléments suivants n'ont pas pu être déduits du comportement observé avec certitude.

1. **Décalages des créneaux de motivation (+1 et +4 jours).** L'intention documentée est
   « le lendemain de la pesée, puis le milieu du cycle ». Le choix précis de +4 plutôt que +3
   ou +5 n'est justifié nulle part ailleurs que par l'exemple mardi / vendredi.
2. **Valeur de 0,4 kg pour le bruit d'une pesée** et de 75 kcal/jour pour l'écart négligeable :
   ce sont des repères d'usage, sans source citée. Leur calibrage mériterait d'être confirmé.
3. **Répartition de la roue anti-fringale** : une part accorde tout, deux la moitié, cinq
   imposent un détour. Le rapport est décrit comme « volontairement défavorable sans être
   décourageant », mais la cible exacte n'est pas explicitée.
4. **Trois profondeurs d'historique voisines pour des besoins proches** : vingt et un jours
   pour l'alerte fringale, trente pour les messages de motivation, vingt-huit pour la lecture
   hebdomadaire. L'écart n'est pas justifié.
5. **Objectif calorique manuel** : il remplace le calcul mais les macronutriments continuent
   d'être dérivés de lui. Il n'est pas dit si l'auteur souhaitait aussi permettre de fixer les
   macronutriments à la main.
6. **Poids visé** : il n'entre dans aucune formule et ne sert qu'à afficher un cap et à
   déclencher une règle de motivation. Était-ce l'intention définitive ?
7. **Suppression d'un compte** : aucun parcours ne la propose. Les écrans de proches tolèrent
   pourtant le cas d'un compte disparu, ce qui suggère qu'il a été envisagé sans être traité.
8. **Consultation des repas d'un proche** : la journée en cours y est volontairement incluse,
   alors qu'elle est exclue partout ailleurs. La justification donnée (« le dîner que l'autre
   vient de saisir ») est plausible mais mérite confirmation.
9. **Nombre de règles de motivation** : la documentation interne annonce douze règles, le
   comportement en expose treize (repli compris). L'écart n'est pas expliqué.
10. **Aliments personnalisés** : ils ne sont jamais supprimables ni modifiables une fois créés.
    Choix ou lacune ?
11. **Objectifs figés** : aucun parcours ne permet de consulter ou de corriger l'historique des
    objectifs. Un membre qui découvre qu'un objectif passé était faux ne peut rien y faire.
12. **Taux de conversion euro / dollar fixé à 1** pour le plafond de dépense : assumé comme une
    approximation prudente, mais à revoir si la dépense devient significative.
13. **Sexe biologique limité à deux valeurs**, imposé par la formule métabolique retenue.
    Aucune alternative n'est prévue pour les personnes que ce choix ne décrit pas.
