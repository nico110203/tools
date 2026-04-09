# Mes Artefacts Claude

Galerie personnelle pour héberger mes artefacts Claude sur GitHub Pages.

## Mise en place

1. Crée un repo GitHub (ex: `mes-artefacts`)
2. Copie tous ces fichiers dans le repo
3. Dans **Settings > Pages**, active GitHub Pages (branche `main`, dossier `/ (root)`)
4. Ton site est en ligne sur `https://ton-username.github.io/mes-artefacts/`

## Ajouter un artefact

1. Copie le fichier `.html` de ton artefact dans le dossier `artefacts/`
2. Ouvre `artefacts.json` et ajoute une entrée :

```json
{
    "titre": "Nom de l'artefact",
    "description": "Courte description.",
    "fichier": "nom-du-fichier.html",
    "tag": "react",
    "date": "2026-04-09"
}
```

3. Commit et push :

```bash
git add .
git commit -m "Ajout: nom de l'artefact"
git push
```

C'est tout ! La page d'accueil se met à jour automatiquement.

## Tags suggérés

- `react` : composants React
- `outil` : calculatrices, convertisseurs, etc.
- `jeu` : jeux interactifs
- `data` : visualisations de données
- `ui` : maquettes et démos d'interface

## Astuce

Pour les artefacts React (.jsx), tu devras les envelopper dans un fichier HTML complet
avec les imports nécessaires (React, ReactDOM, Babel standalone, etc.).
Tu peux demander à Claude de te les convertir en HTML autonome avant de les ajouter.
