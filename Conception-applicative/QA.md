# QA : Pack Dev builder

## Test : Conception applicative

### Conditions
- Option choisie : A (architecture + modèle de données)
- Outils : GitHub, Excalidraw, Mermaid, Obsidian (export PDF), IA pour relecture et reformulation
- Temps passé : ~11h, + 2h pour commencer l'option B

### Consignes
- Niveau attendu non indiqué : full.md part sur du très senior (Kubernetes, SIEM, budget 1M€), difficile à situer pour un junior.
- Rôle des ressources pas clair : faut-il tenir compte de full.md et Synthèse.md, qui ajoutent des contraintes absentes du cahier des charges (hébergement souverain, ANSSI, mobile natif) ?
- Limite de 5-6 pages : les schémas comptent-ils ? Ils sont obligatoires.
- "Pages numérotées" peu adapté à un rendu Markdown.
- Le formulaire accepte un repo GitHub mais la consigne n'en parle pas (rendu fait sur repo).
- Certains skills affichés sur la carte (Security, DevOps...) ne se retrouvent pas forcément dans le test.

### Incohérences dans les ressources
- full.md impose SecNumCloud, données en France et exclusion des clouds US, mais prévoit une intégration ChatGPT API en semaine 7-8.
- Agrément FranceConnect estimé à 3-6 mois, mais FranceConnect en prod dès le sprint 1 (4 semaines).
- Phases de croissance : M4-M12 dans le doc de cadrage, mois 7 à 12 dans le cahier des charges.
- "Patriot Act" cité : la référence actuelle serait plutôt le Cloud Act (2018).

## Plateforme

### Bugs
- **Notes de soumission perdues** : notes et lien GitHub disparus après changement d'onglet.
- **Inscription** : CGU et politique de confidentialité non cliquables.
- **Mail d'inscription** : "Une question ?" non cliquable, aucune adresse mail indiquée.
- **Boutique** : description du pack complet (Test technique full stack, 50 €) encore en placeholder.
- **Temps restant** : 3 valeurs différentes affichées ("8 jours", "64h", "8j 23h restants").
- **Paramètres > Apparence** : chargement déclenché pendant le réglage des curseurs (non bloquant).
- **Langues** : mélange FR/EN sur plusieurs écrans et dans certains tips.
- ~~**Tag de test** : "tagRule test" visible sur les cartes des tests~~ (corrigé?)

### Délais du pack
- 4 ateliers de 48h pour un pack de 8 jours, soit pile 4 × 48h : si le timer démarre à l'achat, il faut enchaîner sans pause. Juste pour quelqu'un qui teste à côté du travail.
- Pistes : démarrer le timer à l'ouverture du 1er atelier, ou allonger la validité du pack avec 48h par atelier à partir de son ouverture.

### Suggestions
- Afficher la durée du pack avant l'achat.
- Rendre le bouton "remettre par défaut" (apparence) plus visible, aujourd'hui dans les options avancées.

### Points positifs
- Interface très réussie.
- Bouton boutique animé en desktop.
- Tips sur le côté.