# Assistant RH & Opérations — Hôtel de la Promenade

Un assistant conversationnel basé sur la technique **RAG (Retrieval-Augmented Generation)** pour permettre aux employés de l'Hôtel de la Promenade de consulter les documents internes de manière naturelle, en français.

---

## Description

Ce projet implémente un chatbot interne capable de répondre aux questions des employés en se basant **uniquement** sur les documents officiels de l'hôtel (politiques, convention collective, procédures, etc.). Le modèle LLM tourne entièrement en local grâce à **Ollama**, garantissant la confidentialité des données.

---

## Documents sources

Les fichiers PDF suivants sont chargés depuis le dossier `data/` :

| Fichier | Catégorie |
|---|---|
| `Convention collective - services alimentaires.pdf` | RH - Légal |
| `Hébergement - Critères du service d'entretien ménager dans un hôtel 4 diamants.pdf` | Qualité - Service |
| `Membres du comité SST.pdf` | Sécurité - Santé |
| `politiques_hotel_de_la_promenade_document.pdf` | Opérations - Manuel |
| `Principes de vie internes.pdf` | Interne - Culture |
| `Processus de Gestion des reservations.pdf` | Interne - Culture |

---

## Architecture technique

Le pipeline suit les étapes classiques d'un système RAG :

```
PDF → Chargement → Nettoyage → Découpage (chunks) → Embeddings → VectorStore (Chroma) → Retriever → LLM (Ollama) → Réponse
```

### 1. Chargement & Nettoyage des documents
- Chargement via `PyPDFLoader` (LangChain)
- Nettoyage du texte : suppression des espaces doubles, des en-têtes répétitifs ("PROMENADE"), et recomposition des mots coupés par des sauts de ligne

### 2. Découpage intelligent (chunking)
- Stratégie **conditionnelle** selon la taille du document :
  - **Documents courts** (< 2 000 caractères) : conservés en un seul bloc pour ne pas perdre de contexte (ex : liste des membres SST)
  - **Documents longs** : découpés en chunks de 1 000 caractères avec un chevauchement de 150 caractères, en respectant les séparateurs structurels (`ARTICLE`, `CHAPITRE`, etc.)
- Chaque chunk est enrichi de métadonnées : `source`, `catégorie`, `section`, `page`, `type`
- Résultat : **89 chunks** au total

### 3. Vectorisation & Stockage
- Modèle d'embeddings : **`nomic-embed-text`** (via `OllamaEmbeddings`)
- Base vectorielle : **ChromaDB** (`langchain-chroma`)

### 4. Retrieval (récupération)
- Algorithme : **MMR (Maximal Marginal Relevance)** pour équilibrer pertinence et diversité des sources
- Configuration : `k=6` chunks récupérés, puisés dans un réservoir de `fetch_k=15`, avec `lambda_mult=0.6`

### 5. Génération (LLM)
- Modèle : **`lfm2.5-thinking:1.2b-q8_0`** via Ollama (avec filtrage des balises `<think>`)
- Réponse en **streaming** avec historique conversationnel court (4 derniers messages)
- Prompt système strict : réponses basées uniquement sur le contexte fourni, avec citation obligatoire des sources

---

## Installation & Lancement

### Prérequis

- Python 3.11+
- [Ollama](https://ollama.com) installé et en cours d'exécution en local

### 1. Installer les dépendances Python

```bash
pip install langchain langchain-community langchain-chroma langchain-ollama pypdf
```

### 2. Télécharger les modèles Ollama nécessaires

```bash
ollama pull nomic-embed-text
ollama pull lfm2.5-thinking:1.2b-q8_0
```

### 3. Préparer les données

Placer tous les fichiers PDF dans un dossier `data/` à la racine du projet :

```
project/
├── data/
│   ├── Convention collective - services alimentaires.pdf
│   ├── Hébergement - Critères du service ...pdf
│   ├── Membres du comité SST.pdf
│   ├── politiques_hotel_de_la_promenade_document.pdf
│   ├── Principes de vie internes.pdf
│   └── Processus de Gestion des reservations.pdf
└── genai.ipynb
```

### 4. Lancer le notebook

```bash
jupyter notebook genai.ipynb
```

Exécuter les cellules dans l'ordre. La dernière cellule lance la boucle de chat interactive.

---

## Utilisation

Une fois le notebook lancé, une invite de commande apparaît dans la dernière cellule :

```
Question Employé > c'est quoi un grief ?
Assistant: Un grief est une divergence d'opinion ou d'interprétation concernant une
application ou une violation de la convention collective...
Source : Convention collective - services alimentaires.pdf (Page 9).
```

Taper `exit` ou `quitter` pour arrêter la session.

---

## Comportement de l'assistant

L'assistant est configuré pour :

- Répondre **uniquement** à partir du contexte des documents officiels
- **Citer systématiquement** la source (nom du fichier et page)
- Présenter les procédures sous forme de **liste à puces**
- Rediriger vers un superviseur ou les RH si l'information est introuvable
- Maintenir un ton **professionnel et bienveillant**

---

## Dépendances principales

| Bibliothèque | Usage |
|---|---|
| `langchain` | Framework RAG (orchestration) |
| `langchain-community` | Chargeurs de documents (PyPDFLoader) |
| `langchain-chroma` | Intégration ChromaDB |
| `langchain-ollama` | Intégration Ollama (embeddings + LLM) |
| `ollama` | Client Python pour le LLM local |
| `pypdf` | Lecture des fichiers PDF |

---

## Notes

- Le modèle LLM et les embeddings tournent **entièrement en local** — aucune donnée n'est envoyée vers des services externes.
- La base vectorielle Chroma est recréée à chaque lancement (pas de persistance disque dans la configuration actuelle).
- Un code commenté en début de notebook montre une version simplifiée du chatbot sans RAG (avec `deepseek-r1:1.5b`), utile pour tester Ollama seul.
