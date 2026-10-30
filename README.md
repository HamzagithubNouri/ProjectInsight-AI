# ProjectInsight AI
 
Plateforme IA d'aide à la supervision des projets d'intégration étudiants : suivi de l'activité GitHub des équipes et revue de code automatisée par LLM, **100 % en local** (aucune donnée envoyée à une API externe).
 
> Projet réalisé lors de mon stage RDI à l'ESPRIT.
 
## Fonctionnalités
 
- **Suivi des équipes** : contributions de chaque étudiant (commits, pull requests) calculées en direct depuis l'API GitHub
- **Revue de code par IA** : analyse de code (collé ou uploadé) avec un LLM local, avec score de qualité, sévérité, description et correctif proposé pour chaque problème
- **Explications pédagogiques (RAG)** : les problèmes critiques et élevés sont enrichis par une explication adaptée à un étudiant, générée à partir d'une base de 27 fiches pédagogiques
- **Correction automatique (`/fix`)** : génération du fichier corrigé, validée par un guardrail syntaxique (`ast.parse()` + 1 retry automatique)
- **Revue de Pull Request** : analyse du diff fichier par fichier, puis synthèse globale (score, problèmes récurrents) consultable en lecture seule par l'enseignant
- **Historique des revues** : chaque finding est sauvegardé (PostgreSQL) et vectorisé (ChromaDB)
- **Gestion par rôles** : Admin, Enseignant, Étudiant (JWT)
- **Connexion GitHub OAuth** : chaque étudiant lie son compte pour attribuer ses contributions
- **Gestion des classes et équipes** : création de classes, d'équipes, choix du Team Leader, connexion du repository
## Architecture
 
```mermaid
flowchart LR
    A[Angular 18+] -->|HTTP / JWT| B[FastAPI]
    B --> C[(PostgreSQL)]
    B --> D[API GitHub]
    B --> E[LangChain LCEL]
    E --> F[Ollama<br/>LLM locaux]
    E --> G[(ChromaDB<br/>RAG)]
```
 
Backend en couches strictes : `routes → services → repositories → models`, avec les schémas Pydantic séparés des modèles SQLAlchemy.
 
## Module IA
 
### Pipelines LangChain (LCEL)
 
Chaque fonctionnalité IA suit le même schéma : `Prompt | ChatOllama | Parser`.
 
- **`PydanticOutputParser`** : force la sortie du LLM à respecter un schéma Pydantic
- **`OutputFixingParser`** : si le JSON est mal formé, un appel de correction (1 tentative max) évite de faire échouer la requête
- **`RecursiveCharacterTextSplitter`** : découpe les gros diffs de Pull Request avant analyse
### RAG pédagogique
 
```
27 fiches Markdown → vectorisation (nomic-embed-text) → index ChromaDB
→ recherche par similarité (à partir d'un finding) → injection dans le prompt
→ DeepSeek-Coder-V2 16B → EducationalEnrichment
```
 
- Base de connaissances : 27 fiches (Sécurité 7, Maintenabilité 7, Qualité de code 7, Performance 5), au format fixe : mauvais exemple / bon exemple / comment corriger / erreurs fréquentes / explication / concepts liés
- Seuls les findings `critical` et `high` sont enrichis (compromis coût / performance)
- Un second index ChromaDB stocke l'historique des findings détectés
### Garde-fous(Guardrail)
 
- **Séparation des responsabilités** : le LLM ne génère que le texte pédagogique (`EducationalEnrichment`). Les métadonnées du finding (sévérité, titre, lignes) sont réinjectées par le code Python, jamais régénérées par le LLM
- **Guardrail syntaxique** : le code corrigé est validé par `ast.parse()` (Python), avec un retry automatique, puis un avertissement explicite si l'erreur persiste
- **Séparation `/analyze` et `/fix`** : deux appels LLM distincts, une tâche à la fois
### Modèles (Ollama)
 
| Modèle | Rôle |
|---|---|
| Qwen2.5-Coder 7B | Revue de code standard (modèle par défaut) |
| DeepSeek-Coder-V2 16B | Enrichissement pédagogique RAG (précision prioritaire) |
| nomic-embed-text | Embeddings pour ChromaDB |
| CodeLlama 7B | Réservé à une future fonctionnalité de scan de sécurité |
 
## Stack technique
 
| Couche | Technologies |
|---|---|
| Frontend | Angular 18+, TypeScript (strict), Tailwind CSS |
| Backend | Python, FastAPI, SQLAlchemy, Pydantic |
| Base de données | PostgreSQL |
| Authentification | JWT, GitHub OAuth |
| IA | Ollama, LangChain (LCEL), Qwen2.5-Coder, DeepSeek-Coder-V2, CodeLlama |
| RAG | ChromaDB, nomic-embed-text |
| Analyse statique (benchmark) | Pylint, Radon, Bandit |
 
## Dépôts
 
| Partie | Dossier | Dépôt |
|---|---|---|
| Backend (API) | [`backend/`](./backend) | [ProjectInsightBackEnd](https://github.com/HamzagithubNouri/ProjectInsightBackEnd) |
| Frontend (Angular) | [`frontend/`](./frontend) | [ProjectInsightFrontend](https://github.com/HamzagithubNouri/ProjectInsightFrontend) |
 
