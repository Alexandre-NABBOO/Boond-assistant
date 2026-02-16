# Contexte : Boond AI Assistant — Historique technique

## Projet
- **Repo GitHub** : https://github.com/alexandre-nabboo/Boond-assistant
- **App live** : https://alexandre-nabboo.github.io/Boond-assistant/
- **Fichier unique** : `index.html` (HTML/CSS/JS, pas de build)
- **Objectif** : Assistant IA (Claude) qui interagit avec l'API Boond Manager via JWT

## Architecture
- Interface chat web déployée sur GitHub Pages
- Authentification Boond : JWT signé HMAC-SHA256 (`buildBoondJWT`)
- Appels API Boond via `fetch` avec header `Authorization: Bearer <jwt>`
- Claude (Anthropic API) interprète les requêtes en langage naturel et génère des `intent` JSON
- Le front exécute ces intents contre l'API Boond

## Base de données Boond
- Schéma complet 87 tables / 746 champs intégré dans `BOOND_SCHEMA` (variable JS)
- Tables principales : TAB_CRMCONTACT, TAB_PROFIL, TAB_PROJECT, TAB_COMPANY, TAB_OPPORTUNITY...

---

## Règles API Boond critiques (découvertes par tests + doc officielle)

### Lecture
```
GET /api/contacts?keywords=NOM&maxResults=5
GET /api/resources?keywords=NOM&maxResults=5
GET /api/contacts/29286         → fiche complète d'un contact
```

### Écriture — UNIQUEMENT via `/information`
```
PUT /api/contacts/{id}/information      Content-Type: application/json
PUT /api/resources/{id}/information
PUT /api/companies/{id}/information
PUT /api/projects/{id}/information
PUT /api/opportunities/{id}/information
PUT /api/candidates/{id}/information
```

**NE JAMAIS faire :**
- `PUT /api/contacts/{id}` → 405
- `PATCH /api/contacts/{id}` → 405
- `PUT /api/contacts/{id}/relationships/mainManager` → 404

### Format body (JSON:API)
```json
{
  "data": {
    "id": "29286",
    "type": "contact",
    "attributes": {
      "firstName": "Alexandre"
    },
    "relationships": {
      "mainManager": { "data": { "id": "1873", "type": "resource" } },
      "pole":        { "data": { "id": "3",    "type": "pole" } },
      "company":     { "data": { "id": "456",  "type": "company" } }
    }
  }
}
```

### RELATIONSHIP_MAP (clés plates → JSON:API relationships)
| Clé plate dans le body Claude | relationships JSON:API |
|-------------------------------|------------------------|
| `mainManagerId: 1873`         | `mainManager: { data: { id: "1873", type: "resource" } }` |
| `hrManagerId: 1873`           | `hrManager: { data: { id: "1873", type: "resource" } }` |
| `poleId: 3`                   | `pole: { data: { id: "3", type: "pole" } }` |
| `companyId: 456`              | `company: { data: { id: "456", type: "company" } }` |
| `contactId: 123`              | `contact: { data: { id: "123", type: "contact" } }` |
| `agencyId: 1`                 | `agency: { data: { id: "1", type: "agency" } }` |

### IDs importants
- `ID_PROFIL` = ID de la ressource (pour mainManager) ≠ `ID_USER`
- Exemple : Assan SABBANE → ID_PROFIL = 1873
- Exemple : Alexandre MONTEL (contact) → ID = 29286

---

## Fonctions JS clés dans index.html

```
callBoond(method, endpoint, body)
  → si PUT/PATCH/POST avec body : délègue à callBoondWrite()
  → sinon : callBoondRaw() direct

callBoondWrite(endpoint, body)
  → normalizeWriteEndpoint() : retire /information si déjà présent
  → extractIdFromEndpoint() : regex /(\d+)(?:\/|$)/
  → buildInfoBody() : convertit body plat en JSON:API avec RELATIONSHIP_MAP
  → PUT {endpoint}/information

callBoondRaw(method, endpoint, body, contentType)
  → NE PAS envoyer body sur GET/HEAD
  → header Authorization: Bearer <jwt>

buildBoondJWT(credentials)
  → HMAC-SHA256, expire 60s

normalizeWriteEndpoint(endpoint)
  → retire /information$ pour éviter /information/information

extractIdFromEndpoint(endpoint)
  → /\/(\d+)(?:\/|$)/ — matche même si /information suit l'ID
```

---

## Bugs résolus (chronologie)

1. **405 sur PUT /api/contacts/{id}** → L'API n'accepte pas l'écriture sur /{id}, il faut /{id}/information
2. **400 "Pas d'ID"** → `extractIdFromEndpoint` ne matchait pas avec `/information` en suffixe → regex corrigée + `normalizeWriteEndpoint`
3. **mainManager ignoré (200 mais pas de changement)** → `mainManager` était dans `attributes`, doit être dans `relationships`
4. **poleId ignoré (200 mais pas de changement)** → même cause, `pole` est une relationship pas un attribut
5. **"GET/HEAD cannot have body"** → `callBoondRaw` envoyait body même sur GET → guard ajouté
6. **405 en multi-étapes** → `callBoond` appelait `callBoondRaw` directement au lieu de `callBoondWrite` pour les writes → `callBoond` redirige maintenant PUT/PATCH/POST vers `callBoondWrite`
7. **Cache GitHub Pages** → Après chaque push, attendre ~5-10 min ou utiliser `?v=N` en suffixe d'URL pour forcer le rechargement CDN

---

## Schéma officiel Boond
- Doc RAML : https://doc.boondmanager.com/api-externe
- Schemas JSON PUT : https://doc.boondmanager.com/api-externe/raml-build/schemas/contacts/informationBodyPut.json
- Idem pour : resources/, companies/, projects/, opportunities/, candidates/

---

## Comment utiliser ce contexte
Colle ce fichier en début de conversation avec Claude en disant :
"Voici le contexte de notre projet Boond Assistant. [colle le contenu]"
