# EduShare

Plateforme web légère pour partager des ressources d'étude (cours, fiches, sujets corrigés) entre étudiants, classées par université/école, année et spécialité.

- Frontend: HTML/CSS/JavaScript vanilla (SPA avec hash‑router)
- Backend: PHP 8 (XAMPP), MySQL (préféré) avec fallback SQLite
- Upload fichiers: PDF, images, Office… stockés hors webroot (`../uploads`) et servis via URL signée
- PWA: service worker (cache statique) et thème clair/sombre

## Sommaire
- [Fonctionnalités](#fonctionnalités)
- [Pré‑requis](#pré-requis)
- [Installation locale (XAMPP)](#installation-locale-xampp)
- [Base de données](#base-de-données)
- [Configuration (sécurité/captcha)](#configuration-sécuritécaptcha)
- [API (endpoints)](#api-endpoints)
- [Notes de développement](#notes-de-développement)
- [Licence](#licence)

## Fonctionnalités
- Recherche par mots‑clés, université, année, matière, tags; tri (récentes, populaires, commentées)
- Partage de ressources (lien ou fichier) avec prévisualisation (PDF/images)
- Commentaires et système de notes (1–5), favoris, détails par ressource
- Espace « Mes commentaires » et profil utilisateur
- Auth simple (inscription/connexion), rôles basiques (user/moderator/admin)
- Sécurité côté serveur: CSRF, CORS/CSP stricts, validation MIME via `finfo`, sanitisation serveur, rate‑limiting, URLs signées

## Pré‑requis
- Windows + XAMPP (Apache + PHP 8 + MySQL) — ou PHP/SQLite
- Navigateur moderne (Chrome/Edge/Firefox)

## Installation locale (XAMPP)
1. Copier le projet dans `D:\xampp\htdocs\EduShare` (ou `C:\xampp\htdocs\EduShare`).
2. Démarrer Apache et MySQL depuis XAMPP Control Panel.
3. Ouvrir l'application: `http://localhost/EduShare/`
   - En cas de cache, forcer le rechargement (Ctrl+F5). Le service worker est actif.

## Base de données
Par défaut, EduShare tente MySQL; en cas d’échec, SQLite est utilisé (`data/edushare.db`).

### Schéma MySQL (si vous souhaitez créer manuellement)
```sql
CREATE TABLE IF NOT EXISTS resources (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  university VARCHAR(255),
  year VARCHAR(50),
  subject VARCHAR(255),
  class_level VARCHAR(255),
  type VARCHAR(50) DEFAULT 'autre',
  description TEXT,
  url TEXT,
  file_path VARCHAR(500),
  is_public TINYINT(1) DEFAULT 1,
  author_name VARCHAR(255),
  author_email VARCHAR(255),
  is_anonymous TINYINT(1) DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS comments (
  id INT AUTO_INCREMENT PRIMARY KEY,
  resource_id INT,
  content TEXT NOT NULL,
  author_name VARCHAR(255),
  user_id INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_comments_resource FOREIGN KEY (resource_id) REFERENCES resources(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS ratings (
  id INT AUTO_INCREMENT PRIMARY KEY,
  resource_id INT,
  rating INT NOT NULL,
  user_id INT DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_ratings_resource FOREIGN KEY (resource_id) REFERENCES resources(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(191) UNIQUE,
  email VARCHAR(191) UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(32) DEFAULT 'user',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Le code crée/altère automatiquement les tables manquantes au démarrage (MySQL & SQLite).

## Configuration (sécurité/captcha)
Options dans `api-php/config.php`. Variables d’environnement supportées:

- `RECAPTCHA_SECRET` — clé secrète reCAPTCHA v3 (optionnel)
- `HCAPTCHA_SECRET` — clé secrète hCaptcha (optionnel)

Si aucune clé n’est fournie, la vérification CAPTCHA est ignorée (honeypot/délai + rate‑limit restent actifs).

### Mesures de sécurité activées
- CSRF via en‑tête `X-CSRF-Token` (+ cookie non httpOnly)
- CORS restreint (origines locales), CSP, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`
- Sanitisation serveur (trim/strip_tags/longueur max)
- Validation MIME fichiers (`finfo`), stockage hors webroot, **URLs signées temporaires**
- Rate‑limiting IP + logs applicatifs `api-php/logs/app.log`

## API (endpoints)
Base: `http://localhost/EduShare/api-php/`

- `resources.php?action=list` — liste des ressources (inclut `avg_rating`, `rating_count`, `comment_count`, `comments`, `signed_url`)
- `resources.php?action=create` — POST JSON; **CSRF requis**
- `resources.php?action=update` — POST JSON; **CSRF requis**
- `resources.php?action=delete` — POST JSON; **CSRF requis** (droits requis)
- `resources.php?action=add_comment` — POST JSON; **CSRF requis**, anti‑spam (honeypot/délai) + CAPTCHA optionnel; associe `user_id` si connecté
- `resources.php?action=delete_comment` — POST JSON; **CSRF requis**; suppression par l’auteur (ou modérateur/admin)
- `resources.php?action=add_rating` — POST JSON; **CSRF requis**; MAJ si déjà noté
- `resources.php?action=file&name=…&exp=…&sig=…` — sert un fichier via URL signée
- `resources.php?action=register|login|logout|me` — inscription/connexion/déconnexion/session
- `upload.php` — POST `multipart/form-data` (champ `file`) → `{ ok, file_path, signed_url, type, size }`

Chaque POST requiert l’en‑tête `X-CSRF-Token` (le cookie `edushare_csrf` est posé automatiquement).

## Notes de développement
- Service worker: les assets peuvent être servis depuis le cache; en cas de modification, forcer le rechargement (Ctrl+F5) ou incrémenter `app.js?v=…`.
- Les fichiers sont stockés dans `uploads/` (racine du projet, hors `api-php`).
- Pour MySQL, utilisez un utilisateur à droits minimaux. Le fallback SQLite est utile en dev.

## Licence
MIT — voir le fichier `LICENSE` si présent.
