# DockerFiles
Docker implementation examples
# Architecture Basée sur Docker pour une Implémentation Maintenable de Decidim

## Résumé Exécutif

Ce document propose une architecture basée sur Docker pour notre implémentation de Decidim qui impose une séparation stricte entre le code de la plateforme principale et les extensions spécifiques à notre organisation. Cette approche répond directement aux défis de maintenance et de mise à niveau de Decidim, qui est actuellement en version bêta (versionnage 0.x.x), tout en garantissant que nos fonctionnalités personnalisées restent intactes et gérables.

## Défi Actuel

Decidim est une plateforme open-source en évolution constante avec un schéma de versionnage 0.x.x, indiquant qu'elle est toujours en développement actif avec des changements importants qui surviennent entre les versions. Comme pour toute plateforme sophistiquée, la personnalisation est essentielle pour répondre aux besoins et exigences spécifiques de notre organisation.

Le défi auquel nous sommes confrontés ne concerne pas la personnalisation en elle-même — qui est précieuse et nécessaire — mais plutôt la façon d'implémenter ces personnalisations d'une manière qui reste durable au fur et à mesure que Decidim évolue. Lorsque les organisations modifient directement les fichiers de la plateforme principale, chaque mise à niveau devient de plus en plus complexe :

- Le processus de mise à niveau nécessite de réconcilier soigneusement nos modifications avec les nouvelles évolutions de la plateforme
- Nos développeurs doivent consacrer un temps considérable à déterminer quelles modifications peuvent être appliquées en toute sécurité
- Nos fonctionnalités uniques peuvent devenir de plus en plus difficiles à maintenir à travers les mises à jour de version
- La connaissance institutionnelle concernant les parties du noyau qui ont été modifiées peut se perdre avec le temps
- Nous pourrions éventuellement faire face au choix difficile entre rester sur une version obsolète ou perdre nos personnalisations

Ce dont nous avons besoin, c'est une approche qui préserve notre capacité à personnaliser tout en créant une séparation plus nette entre le noyau de Decidim et nos extensions spécifiques à l'organisation.

## Solution Proposée : Noyau Immuable avec Couches d'Extension

Nous proposons une architecture basée sur Docker qui impose une discipline de "ne jamais modifier le noyau" par des mesures techniques. Cela crée une séparation nette entre le noyau de Decidim (que nous traitons comme immuable) et nos extensions (qui contiennent toutes nos personnalisations).

### Diagramme d'Architecture

```
┌─────────────────────────────────────────────────┐
│           Notre Application Personnalisée        │
├─────────────────────────────────────────────────┤
│     Couche d'Extension (Notre Dockerfile)        │
│  ┌─────────────────┐  ┌─────────────────────┐   │
│  │  Surcharges UI  │  │  Modules            │   │
│  │  avec Deface    │  │  d'Extension Ruby   │   │
│  └─────────────────┘  └─────────────────────┘   │
│  ┌─────────────────┐  ┌─────────────────────┐   │
│  │  Assets & Thèmes│  │  Configuration      │   │
│  │  Personnalisés  │  │  Initialiseurs      │   │
│  └─────────────────┘  └─────────────────────┘   │
├─────────────────────────────────────────────────┤
│                                                 │
│        Couche Principale (Image Decidim)        │
│                                                 │
│             decidim/decidim:0.26.2              │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Approche d'Implémentation

Notre implémentation consiste en deux composants Docker clés :

1. **Couche de Base** : L'image Docker officielle de Decidim que nous ne modifions jamais en interne
2. **Couche d'Extension** : Notre Dockerfile personnalisé qui étend l'image de base et ajoute uniquement des extensions

#### Exemple de Dockerfile Personnalisé

```dockerfile
# Spécifier explicitement la version exacte de Decidim que nous étendons
FROM decidim/decidim:0.26.2

# Métadonnées organisationnelles
LABEL maintainer="architecture@notreorganisation.com"
LABEL description="Decidim étendu avec des personnalisations spécifiques à l'organisation"
LABEL version="1.0.0"
LABEL base.decidim.version="0.26.2"

# Installer des gems d'extension sans modifier les gems principaux
RUN gem install \
    decidim-awesome:0.7.0 \
    decidim-alternative_landing:0.26.2 \
    # Toutes les autres gems d'extension dont nous avons besoin

# Ajouter des surcharges Deface pour les personnalisations frontend
# Celles-ci modifient l'apparence sans changer les fichiers de modèle principaux
COPY overrides/deface /code/app/overrides/deface

# Ajouter des modules d'extension Ruby qui étendent la fonctionnalité principale
# sans modifier le code original
COPY extensions /code/lib/extensions

# Ajouter des initialiseurs qui chargent nos extensions
COPY config/initializers/extensions.rb /code/config/initializers/

# Ajouter des assets personnalisés
COPY assets /code/app/assets/custom/

# Ajouter de la documentation sur notre approche d'extension
COPY docs/EXTENSIONS.md /code/docs/

# Nous ne remplaçons pas l'entrypoint ou la commande de l'image de base
# sauf si c'est absolument nécessaire
```

#### Configuration Docker Compose

```yaml
version: '3'
services:
  decidim:
    build: 
      context: .
      dockerfile: Dockerfile.extensions
      args:
        # Cela facilite le changement de version de base lors des mises à niveau
        base_decidim_version: 0.26.2
    ports:
      - 3000:3000
    volumes:
      # Mappages de volumes uniquement pour le développement local
      - ./overrides:/code/app/overrides:ro
      - ./extensions:/code/lib/extensions:ro
      - ./config/initializers:/code/config/initializers:ro
    environment:
      - RAILS_ENV=production
      - DATABASE_HOST=pg
      - DATABASE_USERNAME=postgres
    depends_on:
      - pg
      - redis
      
  # La base de données et les autres services suivent...
```

## Avantages de Cette Approche

### 1. Processus de Mise à Niveau Simplifié

Pour mettre à niveau Decidim, nous :
1. Changeons simplement la version de l'image de base dans notre Dockerfile
2. Construisons une nouvelle image
3. Testons si nos extensions fonctionnent toujours avec la nouvelle version
4. Modifions uniquement nos extensions en cas de problèmes de compatibilité
5. Déployons l'image mise à jour

### 2. Gestion Claire des Versions

Nous pouvons marquer nos images étendues pour correspondre à leur compatibilité :
- `notre-org/decidim-extended:0.26.2-1.0` (Basé sur Decidim 0.26.2, nos extensions v1.0)
- `notre-org/decidim-extended:0.27.0-1.1` (Basé sur Decidim 0.27.0, nos extensions v1.1)

Cela fournit une traçabilité claire entre nos personnalisations et les versions principales.

### 3. Réduction de la Dette Technique

En imposant des modifications uniquement par extension :
- Nous évitons l'accumulation de modifications du code principal
- Nous maintenons une meilleure compatibilité avec la version communautaire
- Nous pouvons bénéficier des corrections de bugs et des mises à jour de sécurité de la communauté
- Les nouveaux développeurs peuvent plus facilement comprendre où se trouvent nos personnalisations

### 4. Amélioration des Tests et de l'Assurance Qualité

Notre architecture permet :
- D'exécuter Decidim vanilla parallèlement à notre version étendue pour des tests comparatifs
- D'isoler les problèmes au noyau ou aux extensions pour un dépannage plus rapide
- Des tests automatisés concentrés sur la frontière entre le noyau et les extensions

### 5. Intégration des Développeurs et Partage des Connaissances

Les nouveaux membres de l'équipe bénéficient de :
- Une séparation claire entre ce qui est "nôtre" et ce qui est "Decidim principal"
- Une courbe d'apprentissage réduite car ils n'ont besoin de comprendre que nos extensions
- Une architecture auto-documentée qui montre notre philosophie de modification

## Stratégies d'Extension

Nos extensions suivront ces modèles pour éviter de modifier le code principal :

1. **Modifications Frontend** : Utiliser Deface pour surcharger les vues sans changer les fichiers originaux
2. **Extensions Backend** : Utiliser les classes ouvertes de Ruby et l'aliasing de méthodes pour étendre la fonctionnalité principale
3. **Assets et Thèmes** : Ajouter des feuilles de style personnalisées et du JavaScript via le pipeline d'assets
4. **Configuration** : Utiliser les options de configuration intégrées de Decidim dans la mesure du possible

## Feuille de Route d'Implémentation

1. **Phase 1 : Configuration de l'Environnement de Développement** (2 semaines)
   - Créer un Dockerfile étendu
   - Configurer docker-compose pour le développement
   - Documenter les modèles d'extension pour les développeurs

2. **Phase 2 : Migration des Personnalisations Existantes** (4 semaines)
   - Identifier toutes les personnalisations actuelles
   - Refactoriser en extensions en utilisant les modèles ci-dessus
   - Valider que la fonctionnalité correspond au système actuel

3. **Phase 3 : Déploiement et Tests** (2 semaines)
   - Configurer un pipeline CI/CD pour la construction d'images étendues
   - Mettre en œuvre une stratégie de test pour les extensions
   - Déployer dans l'environnement de staging

4. **Phase 4 : Premier Test de Mise à Niveau de Version Majeure** (2 semaines)
   - Tenter une mise à niveau vers la prochaine version de Decidim
   - Mesurer l'effort requis pour corriger la compatibilité des extensions
   - Affiner le processus en fonction des résultats

## Conclusion

Cette architecture basée sur Docker transforme notre approche de la personnalisation de Decidim, passant d'un fardeau potentiel de maintenance à un système durable et évolutif. En traitant le noyau comme immuable et en nous concentrant exclusivement sur des extensions propres, nous créons un système plus maintenable qui peut évoluer parallèlement à la version communautaire de Decidim tout en préservant nos exigences spécifiques.

L'approche s'aligne sur les principes modernes de microservices et de conteneurisation, mettant l'accent sur des limites claires et la séparation des préoccupations. Cette architecture ne résout pas seulement nos défis immédiats de mise à niveau de Decidim, mais établit un modèle que nous pouvons appliquer à d'autres plateformes open source que notre organisation pourrait adopter à l'avenir.
