# Documentation du Flow "Flow Day 3"

Ce flow Langflow implémente un système automatique de traitement des requêtes helpdesk avec routage intelligent, recherche dans une base FAQ Notion, création de tickets selon l'urgence, et journalisation des échanges.

---

## Architecture générale

---

## 1. Entrée utilisateur

**Composant :** `ChatInput-alm3r`

**Rôle :** Reçoit le message de l'utilisateur depuis le Playground.

**Sortie :** `message` (type `Message`)

Ce message est envoyé à deux endroits :
- Au LLM routeur (pour la classification)
- Au composant `SaveToFile-YBabM` (pour journaliser la requête brute)

---

## 2. Classification initiale (FAQ vs Ticket)

### 2.1 LLM de routage (`GroqModel-DCjzz`)

- **Modèle :** `llama-3.1-8b-instant`
- **Prompt système (extrait) :**

```
Tu es un routeur d'intentions pour un Helpdesk.
Routes disponibles : faq, ticket_creation, escalation.
Retourne uniquement un JSON : {"route": ...}
```

- **Sortie :** `model_output` (type `LanguageModel`) – connecté au Smart Router

### 2.2 Smart Router (`SmartRouter-5zlTW`)

**Entrées :**
- `input_text` = message utilisateur
- `llm` = modèle Groq ci-dessus

**Routes définies :**
- `faq` : questions informatives, comment, procédure
- `ticket_creation` : signalement de problème, bug, demande d'assistance

**Sorties :**
- `category_1_result` → branche FAQ
- `category_2_result` → branche ticket_creation

Le routeur utilise le LLM pour classer et déclenche la sortie correspondante.

---

## 3. Branche FAQ

### 3.1 Extraction de mots-clés (`GroqModel-0nM1Y`)

**Prompt système :**

```
Extrait 1 ou 2 mots-clés techniques de la question.
Retourne un JSON Notion filter :
{
  "filter": {
    "or": [
      { "property": "Question", "title": { "contains": "MOT_CLE" } }
    ]
  }
}
```

**Sortie :** `text_output` (JSON du filtre Notion)

### 3.2 Interrogation de la base FAQ Notion (`NotionListPages-nDbjA`)

**Paramètres :**
- `database_id` : `3659e73c198580b4a387e91f3906d47d`
- `notion_secret` : à renseigner
- `query_json` : reçoit le JSON généré par l'étape précédente

**Sortie :** liste de pages Notion correspondantes (type `Data`)

### 3.3 Conversion de type (`TypeConverterComponent-N5eo2`)

Transforme le résultat `Data` en `Message` pour l'étape suivante.

### 3.4 Extraction de la réponse (`GroqModel-UsUTe`)

**Prompt :**

```
Extract ONLY the FAQ answer from this Notion response.
Return only the answer text. No JSON.
```

**Sortie :** `text_output` (message textuel)

### 3.5 Affichage final (`ChatOutput-cZVBD`)

Affiche la réponse FAQ dans le Playground.

Sortie également connectée à `SaveToFile-fTsKJ` pour sauvegarde.

---

## 4. Branche Ticket

### 4.1 Second routeur (`SmartRouter-RAuXY`)

**Entrées :**
- `input_text` = message original de l'utilisateur (transmis via `category_2_result` du premier routeur)
- `llm` = toujours le même modèle Groq `DCjzz`

**Routes :**
- `Urgent` → problème critique
- `Non urgent` → problème standard

> **Note :** Le deuxième routeur est configuré avec des routes "Urgent" / "Non urgent", mais son `custom_prompt` est vide. Il utilise donc le prompt par défaut.

**Sorties :**
- `category_1_result` → Urgent
- `category_2_result` → Non urgent

### 4.2 Création de la page Notion (`GroqModel-eAA9p` + `NotionPageCreator-3vDjw`)

**`GroqModel-eAA9p`** reçoit le message de l'utilisateur et le transforme en JSON structuré pour Notion.

**Prompt système :**

```json
Convert user message to JSON:
{
  "Nom": { "title": [ { "text": { "content": "..." } } ] },
  "Description": { "rich_text": [ { "text": { "content": "..." } } ] },
  "Etat": { "select": { "name": "Pas commencé" } }
}
```

**`NotionPageCreator-3vDjw` :**
- `database_id` : `3659e73c-1985-8078-a19a-fa7930762727`
- `properties_json` = sortie du LLM
- Crée effectivement la page dans Notion.

### 4.3 Conversion de type (`TypeConverterComponent-Po6Pq`)

Transforme la `Data` retournée par Notion en `Message`.

### 4.4 Génération du message de confirmation (`GroqModel-s82cE`)

**Prompt :**

```
Ignore input. Reply exactly with:
"Your ticket has been created successfully. Our support team will contact you shortly.
And have been informed about the severity of the problem."
```

**Sortie :** message fixe de confirmation (peu importe l'entrée).

### 4.5 Affichage et sauvegarde

- **`ChatOutput-4EJNY`** : affiche la confirmation dans le chat
- **`SaveToFile-XdLPC`** : sauvegarde dans un fichier local (`/tmp/logss`)

---

## 5. Journalisation (SaveToFile)

Trois composants `SaveToFile` écrivent dans des fichiers locaux (format `.txt`) avec append :

| Composant | Contenu sauvegardé |
|---|---|
| `SaveToFile-YBabM` | Message brut de l'utilisateur |
| `SaveToFile-fTsKJ` | Réponse finale de la branche FAQ |
| `SaveToFile-yuhMr` | Réponse de la branche ticket (non urgent ?) |
| `SaveToFile-XdLPC` | Confirmation finale de création de ticket |

> **Remarque :** `SaveToFile-yuhMr` est connecté à `ChatOutput-9vSgB`, qui semble être une sortie intermédiaire (peut-être pour le ticket non urgent). Cependant, dans le flux principal, la sortie de confirmation est `ChatOutput-4EJNY` reliée à `SaveToFile-XdLPC`.

---

## 6. Composants non utilisés ou redondants

- **`GroqModel-RpLh4`** et **`GroqModel-pwHxK`** sont présents dans le JSON mais non connectés au reste du flow. Ils semblent être des versions de test (Few-shot, Chain-of-Thought) non intégrées.
- **`File-YFo93`** (lecture de fichier) est présent mais non utilisé.
- **`TypeConverterComponent-ZoALA`** et **`GroqModel-n8Ct6`** sont reliés à `NotionPageCreator-J7RnW` (qui n'est pas utilisé dans le flux principal – c'est une autre instance de création de page). Ces éléments semblent être des reliquats d'une version antérieure.

---

## 7. Variables sensibles à configurer

- **Notion Secret :** pour `NotionListPages` et `NotionPageCreator` (deux instances distinctes)
- **Groq API Key :** pour tous les modèles Groq (chaque nœud a son propre champ `api_key`)

---

## 8. Résumé fonctionnel

1. L'utilisateur envoie un message.
2. Un LLM détermine s'il s'agit d'une FAQ ou d'une demande de ticket.
3. **Si FAQ :** extraction de mots-clés → recherche dans la base Notion → extraction de la réponse → affichage.
4. **Si ticket :** second niveau d'urgence → création d'une page Notion → message de confirmation fixe → affichage.
5. Toutes les étapes sont journalisées dans des fichiers texte locaux.

> Ce flow assure une automatisation complète du support de niveau 1, avec traçabilité des requêtes et des réponses.
