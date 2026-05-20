# Documentation du flow "Flow day 3"

Ce flow Langflow implémente un **système automatique de traitement des requêtes helpdesk** avec routage intelligent, recherche dans une base FAQ Notion, création de tickets selon l’urgence, et journalisation des échanges.

---

## Architecture générale

```mermaid
flowchart TD
    A[ChatInput] --> B[GroqModel DCjzz<br/>LLM routeur]
    B --> C[SmartRouter 5zlTW<br/>Classification faq / ticket_creation]
    
    C -->|faq| D[GroqModel 0nM1Y<br/>Extraction mots-clés]
    D --> E[NotionListPages nDbjA<br/>Recherche FAQ]
    E --> F[TypeConverter N5eo2]
    F --> G[GroqModel UsUTe<br/>Extraction réponse]
    G --> H[ChatOutput cZVBD]
    
    C -->|ticket_creation| I[SmartRouter RAuXY<br/>Classification Urgent / Non urgent]
    I --> J[GroqModel eAA9p<br/>Création page Notion]
    J --> K[NotionPageCreator 3vDjw]
    K --> L[TypeConverter Po6Pq]
    L --> M[GroqModel s82cE<br/>Message confirmation]
    M --> N[ChatOutput 4EJNY]
    
    A --> O[SaveToFile YBabM<br/>Sauvegarde entrée]
    H --> P[SaveToFile fTsKJ]
    N --> Q[SaveToFile XdLPC]
1. Entrée utilisateur

Composant : ChatInput-alm3r

Rôle : Reçoit le message de l’utilisateur depuis le Playground.

Sortie : message (type Message)

Le message est envoyé à :

Le LLM routeur (classification)
SaveToFile-YBabM (journalisation brute)
2. Classification initiale (FAQ vs Ticket)
2.1 LLM de routage (GroqModel-DCjzz)

Modèle : llama-3.1-8b-instant

Prompt système :

Tu es un routeur d’intentions pour un Helpdesk.
Routes disponibles : faq, ticket_creation, escalation.
Retourne uniquement un JSON : {"route": ...}

Sortie :

model_output (type LanguageModel)
Connecté au Smart Router
2.2 Smart Router (SmartRouter-5zlTW)

Entrées :

input_text = message utilisateur
llm = modèle Groq

Routes :

faq
ticket_creation

Sorties :

category_1_result → branche FAQ
category_2_result → branche Ticket
3. Branche FAQ
3.1 Extraction de mots-clés (GroqModel-0nM1Y)

Prompt système :

Extrait 1 ou 2 mots-clés techniques de la question.
Retourne un JSON Notion filter :
{
  "filter": {
    "or": [
      { "property": "Question", "title": { "contains": "MOT_CLE" } }
    ]
  }
}

Sortie :

text_output (JSON filtre Notion)
3.2 Interrogation base FAQ (NotionListPages-nDbjA)

Paramètres :

database_id : 3659e73c198580b4a387e91f3906d47d
notion_secret
query_json

Sortie :

Liste de pages Notion (type Data)
3.3 Conversion (TypeConverterComponent-N5eo2)

Transforme Data → Message.

3.4 Extraction réponse (GroqModel-UsUTe)

Prompt :

Extract ONLY the FAQ answer from this Notion response.
Return only the answer text. No JSON.
3.5 Affichage (ChatOutput-cZVBD)

Affiche la réponse dans le Playground.

Sauvegarde via SaveToFile-fTsKJ.

4. Branche Ticket
4.1 Second routeur (SmartRouter-RAuXY)

Routes :

Urgent
Non urgent

Utilise le même modèle Groq (DCjzz).

4.2 Création page Notion
GroqModel-eAA9p

Prompt système :

Convert user message to JSON:
{
  "Nom": { "title": [ { "text": { "content": "..." } } ] },
  "Description": { "rich_text": [ { "text": { "content": "..." } } ] },
  "Etat": { "select": { "name": "Pas commencé" } }
}
NotionPageCreator-3vDjw
database_id : 3659e73c-1985-8078-a19a-fa7930762727
properties_json

Crée la page dans Notion.

4.3 Conversion (TypeConverterComponent-Po6Pq)

Transforme Data → Message.

4.4 Confirmation (GroqModel-s82cE)

Prompt :

Ignore input. Reply exactly with:
"Your ticket has been created successfully. Our support team will contact you shortly. And have been informed about the severity of the problem."
4.5 Affichage & Sauvegarde
ChatOutput-4EJNY
SaveToFile-XdLPC
5. Journalisation
Composant	Contenu sauvegardé
SaveToFile-YBabM	Message utilisateur brut
SaveToFile-fTsKJ	Réponse FAQ
SaveToFile-yuhMr	Réponse ticket intermédiaire
SaveToFile-XdLPC	Confirmation finale ticket

Format : .txt en mode append (/tmp/logss)

6. Composants non utilisés
GroqModel-RpLh4
GroqModel-pwHxK
File-YFo93
NotionPageCreator-J7RnW

Ces éléments sont des reliquats de tests antérieurs.

7. Variables sensibles
Notion Secret
Groq API Key

Chaque nœud Groq possède son propre champ api_key.

8. Résumé fonctionnel
L’utilisateur envoie un message.
Un LLM classe la requête.
Si FAQ → Recherche Notion → Extraction réponse → Affichage.
Si Ticket → Classification urgence → Création page Notion → Confirmation.
Toutes les étapes sont journalisées.
Conclusion

Ce flow met en place une automatisation complète du support niveau 1, avec :

Routage intelligent par LLM
Intégration dynamique avec Notion
Création automatique de tickets
Traçabilité complète des échanges
