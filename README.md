# Fourchette

> Restez dans la fourchette.

Application de suivi de l'alimentation et de l'activité physique. Le scan d'un code-barres
récupère les valeurs nutritionnelles du produit depuis **OpenFoodFacts** (base ouverte,
sans clé d'API), les convertit selon la quantité consommée et les confronte aux objectifs
caloriques calculés à partir du profil.

## Installation (PWA)

L'interface est installable depuis le navigateur (Chrome, Edge, Safari iOS) : manifeste,
icônes `any` et `maskable`, et service worker qui met en cache la coquille applicative.
Hors ligne, l'application se lance et affiche l'écran de connexion — les données, elles,
exigent l'API : rien n'est mis en cache côté journal, pour ne jamais afficher de totaux faux.

Le worker n'est enregistré qu'en production. Pour purger les caches après un changement
de stratégie, incrémenter `VERSION` dans `web/public/sw.js`.

## Déploiement derrière un nginx mutualisé

`web/nginx.conf` est l'**unique** définition du routage applicatif : fallback SPA,
réécriture `/api` → `/v1`, en-têtes de cache PWA. Le même fichier s'applique en local
et en production.

Le nginx mutualisé du serveur ne fait que terminer TLS et relayer. Il ne doit
redéfinir **aucune** règle applicative — en particulier pas de bloc `/api` : le
conteneur web s'en charge déjà, et deux définitions concurrentes finissent par diverger.

Les blocs HTTP et HTTPS prêts à déposer sont versionnés dans
[`deploy/nginx/suivi-regime.duckdns.org.conf`](deploy/nginx/suivi-regime.duckdns.org.conf) :
redirection vers HTTPS, exclusion du challenge ACME, en-têtes de sécurité, relais vers le
conteneur. L'en-tête `Permissions-Policy` y autorise explicitement `camera=(self)` — les
jeux d'en-têtes tout faits contiennent presque toujours `camera=()`, qui casserait le scan
de codes-barres sans message d'erreur.

Ordre de mise en place : nginx refuse de charger un `ssl_certificate` absent, donc bloc
HTTP seul, puis `certbot certonly --webroot -w /var/www/certbot -d suivi-regime.duckdns.org`,
puis bloc HTTPS et `nginx -t && systemctl reload nginx`.

Les ports publiés n'écoutent que sur `127.0.0.1` (`BIND_ADDR`) : les conteneurs ne sont
joignables que par le proxy de l'hôte, jamais depuis l'extérieur en clair. Si le proxy
est lui-même conteneurisé, retirer les `ports` du service `fourchette-front-web` et
faire rejoindre au proxy le réseau compose, puis viser `http://fourchette-front-web:80`.

## Architecture

| Dossier | Rôle | Stack |
|---------|------|-------|
| `api/` | Serveur REST | NestJS 11, TypeScript, Mongoose, JWT |
| `web/` | Interface | React 19, TypeScript, Vite, shadcn/ui, Tailwind v4 |

Les deux dossiers sont indépendants : chacun a son `package.json`, son build et son projet
Vercel.

## Démarrage

Prérequis : Node 20+ et une base MongoDB (locale ou Atlas M0 gratuit).

```bash
npm run install:all

cp api/.env.example api/.env      # renseigner MONGODB_URI et les secrets JWT
cp web/.env.example web/.env.local

npm run dev:api    # http://localhost:3001/v1   (Swagger sur /v1/docs)
npm run dev:web    # http://localhost:5173
```

Sans MongoDB sous la main, `npm run start:memory --prefix api` démarre l'API sur une base
en mémoire — les données disparaissent à l'arrêt du processus.

### Variables d'environnement

**`api/.env`**

| Variable | Rôle |
|----------|------|
| `MONGODB_URI` | Chaîne de connexion MongoDB |
| `JWT_SECRET` / `JWT_REFRESH_SECRET` | Secrets de signature — chaînes longues et aléatoires |
| `JWT_EXPIRES_IN` / `JWT_REFRESH_EXPIRES_IN` | Durées de validité (`15m`, `30d`) |
| `CORS_ORIGINS` | Origines autorisées, séparées par des virgules |
| `OFF_USER_AGENT` | Contact exigé par la politique d'usage d'OpenFoodFacts |
| `ADMIN_EMAILS` | Adresses autorisées sur la console d'administration, séparées par des virgules. Vide = console fermée |
| `INTEGRATIONS_ENCRYPTION_KEY` | Clé AES-256 (64 caractères hex) chiffrant les jetons OAuth. Absente = intégrations désactivées |
| `INTEGRATIONS_CALLBACK_BASE_URL` | Base publique de l'API, vue par le fournisseur OAuth |
| `INTEGRATIONS_WEB_APP_URL` | Base du front, où l'utilisateur est renvoyé après autorisation |
| `GOOGLE_HEALTH_CLIENT_ID` / `GOOGLE_HEALTH_CLIENT_SECRET` | Client OAuth créé dans la [console Google Cloud](https://console.cloud.google.com/) |
| `GOOGLE_HEALTH_REDIRECT_URI` / `STRAVA_REDIRECT_URI` | URI de retour déclarées chez chaque fournisseur. Derrière nginx : `https://<domaine>/api/integrations/<fournisseur>/callback` — **pas** `/v1` |
| `GOOGLE_LOGIN_CLIENT_ID` / `GOOGLE_LOGIN_CLIENT_SECRET` | Client OAuth de la **connexion par compte Google**, distinct du précédent. Vide = bouton masqué, seul le mot de passe subsiste |
| `GOOGLE_LOGIN_REDIRECT_URI` | URI de retour déclarée chez Google, à la chaîne près. Derrière nginx, c'est `https://<domaine>/api/auth/google/callback` — **pas** `/v1`, que le proxy réécrit en interne |
| `STRAVA_CLIENT_ID` / `STRAVA_CLIENT_SECRET` | Application déclarée sur [strava.com/settings/api](https://www.strava.com/settings/api) |

**`web/.env.local`**

| Variable | Rôle |
|----------|------|
| `VITE_API_URL` | Base de l'API ; le client y ajoute `/api`. Vide = même origine |

## Fonctionnalités

- **Scan de code-barres** par la caméra (ZXing, formats EAN-13/8, UPC-A/E, Code 128), avec
  saisie manuelle en repli. Le produit est résolu via OpenFoodFacts puis mis en cache dans
  MongoDB pendant 7 jours.
- **Journal alimentaire** par repas (petit-déjeuner, déjeuner, dîner, collation). Chaque
  entrée fige un instantané des valeurs pour 100 g : une correction ultérieure de la fiche
  produit ne réécrit pas l'historique.
- **Aliments personnalisés** pour les préparations maison absentes de la base.
- **Reprendre** : un seul bouton par repas, trois onglets — aliments fréquents, saisies
  récentes, et copie d'un repas depuis une journée passée. Les aliments de la journée
  choisie arrivent tous cochés : on décoche ceux qu'on ne reprend pas. La copie duplique
  les instantanés stockés, sans repasser par OpenFoodFacts, et s'annule d'un geste depuis
  le message de confirmation.
- **Proches** : un lien entre deux comptes (invitation par l'e-mail d'un compte existant,
  acceptée dans l'application — aucun e-mail n'est envoyé). Depuis un repas, « Envoyer à… »
  transmet tout ou partie des aliments ; l'envoi attend dans le journal du destinataire,
  qui l'accepte en ajustant ses portions (50 à 125 %, ou au gramme), son jour et son repas,
  ou le refuse. Rien n'entre dans un journal sans l'accord de son titulaire. Chacun peut en
  outre ouvrir ses repas à un proche, qui les reprend alors lui-même depuis « Reprendre ›
  Journées ». Fermé par défaut, réglé de chaque côté. Les envois sans réponse expirent au
  bout de 14 jours ; retirer un proche emporte ceux en attente.
- **Activité physique** : catalogue de 36 activités avec valeurs MET issues du *Compendium
  of Physical Activities*. Dépense = `MET × 3,5 × poids ÷ 200 × durée`.
- **Objectifs calculés** : métabolisme de base par la formule de Mifflin-St Jeor, dépense
  totale selon le niveau d'activité, ajustement par objectif (−20 % en perte, +12 % en
  prise), répartition des macros en g/kg.
- **Objectifs figés jour par jour** : une pesée ou une retouche du profil recalcule
  l'objectif à compter du jour même, dans le fuseau de l'utilisateur. Les journées passées
  gardent le leur — calendrier, graphiques et bilans d'hier ne bougent pas. Stockage dans
  `daily_targets` : un document par jour de changement, valable jusqu'au suivant. La part
  recréditée des séances y est figée de la même façon.
- **Tendances** sur 7, 14 ou 30 jours, avec alternative textuelle au graphique. Les dépenses
  s'y tracent sous l'axe, en miroir des apports, pour leur seule part recréditée au budget
  — la dépense brute reste lisible au survol.
- **Poids théorique** : seconde courbe du suivi de poids, depuis la première pesée de
  l'année écoulée. Chaque journée renseignée pèse sur le lendemain : TDEE du jour plus part
  recréditée des séances, moins les apports, à 7 700 kcal le kilo. Une journée sans repas
  saisi compte pour l'équilibre ; celle en cours n'entre pas dans le calcul.
- **Calories d'activité recréditées** : curseur de 0 à 100 % dans le profil, réglant la part
  de la dépense qui rejoint le budget du jour. **25 % par défaut** : les formules MET
  surestiment — elles englobent le métabolisme de base, déjà compté dans le TDEE — et le
  facteur d'activité du profil pré-compte lui aussi une part de l'effort.
- **Suivi de poids amorcé** dès le premier enregistrement du profil : le point de départ est
  déjà connu, inutile de le ressaisir. Les corrections ultérieures du profil ne réécrivent
  pas la mesure.
- **Déplacement d'un aliment entre repas** par glisser-déposer, à la poignée en début de
  ligne. L'écran de modification propose la même bascule : le geste est un raccourci, pas le
  seul chemin.
- **Invitation à installer l'application** : invite native sur Chromium, marche à suivre
  détaillée sur Safari iOS, qui n'expose aucune API pour cela. Un refus est mémorisé
  quinze jours.
- **Google Health et Strava** : import automatique des séances, avec la dépense mesurée par
  le service. Voir la section dédiée ci-dessous.
- **Alerte fringale** : bouton central de la barre de navigation, en saillie et aux couleurs
  d'alerte. Il ouvre un parcours en trois temps, conçu pour être suivi debout devant un
  placard. *Souffler* : deux ou trois arguments tirés des derniers jours de suivi — séance
  récente, série de journées tenues, avance calorique de la semaine, poids en baisse —
  choisis parmi dix règles selon ce que disent réellement les données. *Chiffrer* : on
  cherche ou scanne ce qui fait envie, on ajuste la portion, et l'application dit ce que ça
  coûte, ce qu'il resterait au budget, et l'équivalent en minutes de marche, vélo, course ou
  natation. *Décider* : craquer — l'aliment part alors en collation, sans détour — ou mettre
  sa part en jeu à la roue. Huit issues : tout manger, la moitié, vingt minutes d'attente,
  dix minutes de marche quel que soit le temps, trente squats, un brossage de dents, une
  chanson dansée en entier, une minute de gainage. Chaque détour dure assez pour que l'envie
  retombe — c'est le vrai mécanisme, le jeu n'est que l'emballage. La porte de sortie reste
  ouverte à chaque écran : un parcours sans issue se contourne en fermant l'application, et
  le repas n'est alors jamais noté.
- **Messages de motivation**, deux fois par semaine : le lendemain de la pesée et au milieu
  du cycle — mardi et vendredi pour une pesée le lundi. Douze règles lisent les deux dernières
  semaines (assiduité, séries de journées tenues, volume d'activité, comparaison entre
  semaines, variation de poids, absence prolongée) et chacune porte trois ou quatre
  formulations. Les huit derniers messages envoyés sont mémorisés : ni la même règle ni la
  même phrase deux fois de suite. Réglable dans le profil, avec l'heure d'envoi.
- **Console d'administration** (`/admin`), réservée aux adresses de `ADMIN_EMAILS` : comptes
  inscrits, actifs sur sept jours, saisies totales et hebdomadaires, palmarès des aliments,
  histogramme des quatorze derniers jours, séances et pesées. Permet d'envoyer une
  notification d'essai à un compte choisi, avec le nombre d'appareils réellement touchés.

## Google Health et Strava

Les séances sont reprises toutes les deux heures, et à la demande depuis le profil. La
dépense importée est une mesure : elle n'est jamais recalculée par la formule MET.

> **L'API Web Fitbit est morte.** Google l'a arrêtée en septembre 2026 et a fermé
> l'inscription de nouvelles applications. Son remplaçant est la **Google Health API**
> (`health.googleapis.com/v4`), qui couvre les appareils Fitbit et les données santé
> remontées au compte Google. Les jetons de l'ancienne intégration n'y valent rien : chaque
> utilisateur doit reconsentir.

**Déclarer les applications.** L'URI de redirection doit correspondre exactement :

```
<INTEGRATIONS_CALLBACK_BASE_URL>/v1/integrations/google_health/callback
<INTEGRATIONS_CALLBACK_BASE_URL>/v1/integrations/strava/callback
```

Côté Google : console Cloud, API « Google Health » activée, client OAuth de type application
web. **Le périmètre `googlehealth.*` est classé « restreint »** — la mise en production exige
la revue de confidentialité et de sécurité de Google, sans délai annoncé. En mode test,
l'application fonctionne immédiatement pour les comptes inscrits comme testeurs, ce qui
suffit à un usage personnel.

Côté Strava, le champ « Authorization Callback Domain » ne prend que le domaine, sans
protocole ni chemin.

**Ce que le montage garantit.**

- Les jetons OAuth sont chiffrés en AES-256-GCM avant écriture en base : une copie du dump
  ne suffit pas à les rejouer.
- Les deux fournisseurs ne traitent pas le jeton de rafraîchissement de la même façon.
  Strava en renvoie un neuf à chaque échange, et le nouveau doit remplacer l'ancien. Google
  l'émet **une seule fois**, au premier consentement, et l'omet ensuite : l'écraser avec une
  valeur vide couperait la connexion à l'expiration suivante. D'où `access_type=offline` et
  `prompt=consent` sur l'URL d'autorisation, et l'écriture conditionnelle en base.
- Une séance présente dans les deux services n'est comptée qu'une fois : deux séances de
  provenances différentes démarrant à moins de quinze minutes d'écart sont tenues pour la
  même.
- Une séance importée puis supprimée à la main ne revient pas au cycle suivant.
- Strava ne publie pas les calories sur la liste des activités, seulement sur le détail de
  chacune. Le quota étant compté par application et non par utilisateur, les appels de détail
  sont plafonnés par cycle ; le reliquat part au suivant.

**Deux réserves à connaître.** Le `caloriesKcal` d'une séance Google Health est la dépense
totale sur la durée de la séance, métabolisme de base compris — elle n'est pas retouchée, le
réglage « calories d'activité recréditées » absorbe cette marge. Et si vous importez la
dépense réelle, basculez le niveau d'activité du profil sur « sédentaire » : le facteur de
Mifflin-St Jeor pré-compte sinon un effort que les capteurs mesurent déjà.

Aucun webhook n'est déclaré : un cycle régulier suffit pour un suivi calorique, où une séance
vue avec deux heures de retard ne change rien à la journée.

## API

Documentation interactive : `http://localhost:3001/v1/docs`.

| Méthode | Route | Rôle |
|---------|-------|------|
| `POST` | `/v1/auth/register` · `/login` · `/refresh` · `/logout` | Authentification JWT |
| `GET` `PATCH` | `/v1/me` | Profil et objectifs |
| `GET` | `/v1/foods/barcode/:barcode` | Résolution d'un code-barres |
| `GET` | `/v1/foods/search?q=` | Recherche par nom, cadrée sur le marché français |
| `GET` `POST` | `/v1/foods/custom` | Aliments personnalisés |
| `GET` | `/v1/journal/meals/history?mealType=` | Journées passées où ce repas est renseigné |
| `POST` | `/v1/journal/meals/copy` | Recopie un repas d'un jour sur un autre (`entryIds` : une partie seulement) |
| `GET` | `/v1/journal?day=` | Journée par repas |
| `POST` `PATCH` `DELETE` | `/v1/journal/entries[/:id]` | Entrées alimentaires |
| `GET` | `/v1/activities/catalog` | Catalogue MET |
| `GET` `POST` `PATCH` `DELETE` | `/v1/activities[/entries/:id]` | Séances |
| `GET` | `/v1/dashboard?day=` | Bilan du jour |
| `GET` | `/v1/dashboard/trend?from=&to=` | Série journalière (objectif et dépense recréditée par jour) |
| `GET` | `/v1/dashboard/weight-projection?to=` | Poids théorique jour par jour |
| `GET` | `/v1/dashboard/weight-analysis?to=&days=` | Écart pesé/théorique et réglages proposés |
| `POST` | `/v1/me/onboarding` | Questionnaire d'accueil ; idempotent une fois rempli |
| `POST` | `/v1/me/targets-preview` | Objectifs qu'un profil donnerait, sans rien enregistrer |
| `GET` | `/v1/insights/weekly` | Dernière lecture hebdomadaire rédigée |
| `POST` | `/v1/insights/weekly/refresh` | En rédiger une nouvelle (une par heure au plus) |
| `GET` `POST` | `/v1/contacts` | Proches ; invitation par e-mail d'un compte existant |
| `POST` | `/v1/contacts/:id/accept` | Accepter une invitation |
| `PATCH` `DELETE` | `/v1/contacts/:id` | Ouvrir ses repas (`shareMeals`) ; retirer, refuser, annuler |
| `GET` | `/v1/contacts/:id/meals/history?mealType=` | Journées d'un proche qui a ouvert ses repas |
| `POST` | `/v1/contacts/:id/meals/copy` | Reprendre des lignes d'un proche, à ses portions |
| `POST` | `/v1/shares` | Envoyer tout ou partie d'un repas à un proche |
| `GET` | `/v1/shares/inbox` | Envois reçus en attente |
| `POST` | `/v1/shares/:id/accept` · `/decline` | Accepter (portions, jour, repas) ou refuser |
| `GET` | `/v1/integrations` | État des connexions Google Health et Strava |
| `POST` | `/v1/integrations/:provider/authorize` | URL de consentement OAuth |
| `POST` | `/v1/integrations/:provider/sync` | Synchronisation à la demande |
| `DELETE` | `/v1/integrations/:provider` | Déconnexion et révocation |
| `GET` | `/v1/craving/summary?day=` | Budget du jour et arguments de réassurance (alerte fringale) |
| `GET` | `/v1/admin/stats` · `/users?q=` | Console d'administration — réservée à `ADMIN_EMAILS` |
| `POST` | `/v1/admin/push-test` | Notification d'essai vers un compte choisi |
| `GET` | `/v1/health` | Sonde de disponibilité |

## Tests

```bash
npm run test:e2e
```

Démarre un MongoDB en mémoire, monte l'application Nest complète et déroule le parcours
réel — inscription, cloisonnement entre comptes, scan d'un code-barres OpenFoodFacts,
calculs caloriques, tendances. Le test touche la vraie API OpenFoodFacts : une connexion
réseau est nécessaire.

```bash
npm run test:integrations
```

Vérifie le module Google Health/Strava sans réseau : réversibilité du chiffrement et
détection d'une altération, refus d'un jeton d'accès présenté comme état OAuth, idempotence
de la synchronisation, dédoublonnage entre services, et persistance du rejet d'une séance
importée puis supprimée.

```bash
npm run test:features
```

Vérifie sans réseau la recherche d'aliments, la console d'administration, l'alerte fringale
et les messages de motivation. `fetch` est remplacé le temps des contrôles de recherche :
ce qui est vérifié, c'est la requête construite — langue française, filtre du marché,
complétion du dernier mot, neutralisation de la syntaxe du moteur —, pas la disponibilité
d'OpenFoodFacts. Couvre aussi le refus de la console à un membre ordinaire, le calcul du
budget restant, et la rotation anti-répétition des messages.

## Déploiement Vercel (offre gratuite)

Deux projets Vercel pointant sur le même dépôt, avec des **Root Directory** différents.

### 1. Base de données

Créer un cluster **MongoDB Atlas M0** (gratuit). Dans *Network Access*, autoriser
`0.0.0.0/0` : les fonctions Vercel n'ont pas d'adresse IP fixe.

### 2. Projet API

- Root Directory : `api`
- Variables d'environnement : `MONGODB_URI`, `JWT_SECRET`, `JWT_REFRESH_SECRET`,
  `CORS_ORIGINS` (l'URL du front), `OFF_USER_AGENT`
- `api/vercel.json` route toutes les requêtes vers la fonction `api/index.ts`, qui met
  l'instance Nest en cache entre deux invocations pour réutiliser la connexion Mongo.

### 3. Projet Web

- Root Directory : `web`
- Variable d'environnement : `VITE_API_URL` = l'URL du projet API
- `web/vercel.json` gère la réécriture SPA et autorise la caméra via `Permissions-Policy`.

Le scan par caméra exige un contexte sécurisé : il fonctionne en HTTPS sur Vercel et sur
`localhost` en développement, jamais en HTTP simple sur une IP de réseau local.

## Données produits

Les fiches proviennent d'[OpenFoodFacts](https://world.openfoodfacts.org), sous licence
ODbL. La recherche plein texte passe par le service *Search-a-licious*
(`search.openfoodfacts.org`), les endpoints historiques `/cgi/search.pl` et
`/api/v2/search` répondant en 503 sur le serveur principal.

Renseignez un contact réel dans `OFF_USER_AGENT` : la politique d'usage de l'API l'exige.
