# better-storage — Plan d'implémentation

> Stockage de fichiers unifié. S3, R2, local, Vercel Blob. Upload, download, signed URLs, registry, proxy.

**Conventions** : [ECOSYSTEM.md](../better-agnostic/ECOSYSTEM.md) dans better-agnostic.

---

## Problème résolu

Les fichiers sont stockés partout : S3, R2, Vercel Blob, Supabase Storage, filesystem local. Chaque provider a son API, son format d'URL, sa logique de signed URL.

- **Vendor lock-in** : Migrer de S3 à R2 = réécrire tout le code.
- **Pas d'abstraction** : Upload, download, signed URL — chaque lib fait différemment.
- **Pas d'ID applicatif** : La clé S3 (`uploads/2024/abc.jpg`) est un détail d'implémentation. Le client devrait connaître un ID stable.
- **Pas de contrôle d'accès** : Servir un fichier privé nécessite un proxy ou des signed URLs avec logique custom.
- **Pas de validation** : Type MIME, taille, dimensions — à implémenter soi-même.

better-storage unifie : `upload()`, `download()`, `delete()`, `getUrl()`, `sign()`. Un adapter par provider. Des plugins pour validation, quota, transformation d'images, registry (ID en DB), et proxy HTTP avec auth.

---

## Concepts clés

| Concept | Description |
|---------|-------------|
| **StorageFile** | `{ key, size, mimeType, etag, lastModified, metadata }`. Représente un fichier. |
| **Key** | Chemin unique dans le storage : `avatars/user-123.jpg`. Généré par plugin-keys si absent. |
| **FileRecord** | Extension du core (plugin-registry) : `{ id, key, ... }` + extensions via declaration merging (`file.proxy`, `file.quota`). |
| **Adapter** | Interface : `upload`, `download`, `delete`, `exists`, `getMetadata`, `list`. |
| **FileRecordExtensions** | Interface vide extensible. Chaque plugin déclare `declare module 'better-storage' { interface FileRecordExtensions { proxy?: {...} } }`. Champs top-level typés, pas de `metadata` wrapper. |

---

## Exemple d'usage

```ts
import { createStorage } from "@better-storage/core";
import { createR2Adapter } from "@better-storage/adapter-r2";
import { createKeysPlugin } from "@better-storage/plugin-keys";
import { createUrlPlugin } from "@better-storage/plugin-url";
import { createValidatePlugin } from "@better-storage/plugin-validate";
import { createRegistryPlugin } from "@better-storage/plugin-registry";
import { createProxyPlugin } from "@better-storage/plugin-proxy";

const storage = createStorage({
  adapter: createR2Adapter({
    accountId: process.env.R2_ACCOUNT_ID,
    accessKeyId: process.env.R2_ACCESS_KEY,
    secretAccessKey: process.env.R2_SECRET_KEY,
    bucket: "myapp",
  }),
  plugins: [
    createKeysPlugin({
      strategy: "ulid",
      prefix: (file, ctx) => `${ctx.tenantId}/${ctx.userId}`,
    }),
    createUrlPlugin({ baseUrl: "https://cdn.example.com", secret: process.env.STORAGE_SECRET }),
    createValidatePlugin({
      rules: [
        { match: "avatars/*", mimeTypes: ["image/jpeg", "image/png"], maxSize: 5 * 1024 * 1024, image: { minWidth: 100, aspectRatio: "1:1" } },
        { match: "*", maxSize: 100 * 1024 * 1024 },
      ],
    }),
    createRegistryPlugin({ adapter: dbAdapter }),
    createProxyPlugin({
      basePath: "/files",
      authorize: async (req, file) => canAccessFile(req.user.id, file.id),
      cache: { "public/*": { maxAge: "1y" }, "avatars/*": { maxAge: "7d" } },
    }),
  ],
});

// Upload — clé auto-générée, validation, enregistrement en DB
const file = await storage.upload(buffer, { mimeType: "image/jpeg" });
// { id: 'file_01J8X...', key: 'tenant-abc/user-123/01J8X....jpg', ... }

await storage.registry.attach(file.id, { resourceType: "user", resourceId: user.id, role: "avatar" });

// URL publique ou signée
storage.url.get(file.key);
storage.url.sign(file.key, { expiresIn: "1h" });

// Proxy — servir via le serveur avec auth
// GET /files/file_01J8X...?w=64&h=64
app.use("/files", storage.proxy.handler());
```

---

## Fonctionnement détaillé des plugins

### plugin-keys (P0)

**Problème** : Éviter les collisions, normaliser les paths, appliquer des conventions.

**Fonctionnement** :
- Stratégies : `uuid`, `ulid`, `hash`, `date`, ou fonction custom.
- `prefix: (file, ctx) => \`${ctx.tenantId}/${ctx.userId}\`` : structure des clés.
- `sanitize: true` : nettoie le filename (espaces, accents).
- Si `key` absent à l'upload, génération automatique. Ex: `uploads/2024-11/01J8XABCDEF.jpg`.

### plugin-url (P0)

**Problème** : Générer des URLs publiques et signées. Chaque provider (S3, R2) a sa propre logique.

**Fonctionnement** :
- `storage.url.get(key)` : URL publique (si fichier public).
- `storage.url.sign(key, { expiresIn, allowedIps?, allowedMethods? })` : URL signée avec JWT ou signature provider.
- `storage.url.verify(signedUrl)` : vérifie la validité côté serveur.
- `storage.url.presign(key, { expiresIn, maxSize, allowedMimeTypes })` : presigned PUT pour upload direct depuis le client (évite de passer par le serveur).

### plugin-validate (P0)

**Problème** : Rejeter les fichiers invalides (mauvais type, trop gros, dimensions incorrectes) avant qu'ils touchent le storage.

**Fonctionnement** :
- Règles par pattern : `{ match: 'avatars/*', mimeTypes, maxSize, image: { minWidth, maxWidth, aspectRatio } }`.
- `strictMimeCheck: true` : vérification des magic bytes, pas juste l'extension.
- Si invalide → `StorageValidationError` avec `reason` (`FILE_TOO_LARGE`, `INVALID_MIME`, `INVALID_DIMENSIONS`).

### plugin-registry (P0)

**Problème** : La clé physique (`uploads/2024/abc.jpg`) change si on migre de bucket. Le client a besoin d'un ID stable.

**Fonctionnement** :
- Chaque fichier uploadé est enregistré en DB avec un `id` applicatif (`file_01J8X...`).
- `storage.registry.getById(id)`, `storage.registry.getByKey(key)`.
- `storage.registry.attach(id, { resourceType, resourceId, role })` : lie le fichier à une ressource (user, invoice).
- `storage.registry.getAttachments('user', 'user-123', { role: 'avatar' })`.
- `storage.registry.softDelete(id)` : marque deletedAt sans supprimer le fichier physique.
- `storage.registry.cleanup({ olderThan, includeOrphans })` : supprime les soft-deleted et orphelins.
- **FileRecordExtensions** : `file.proxy`, `file.quota`, etc. sont des champs top-level pour la config par fichier.

### plugin-proxy (P0)

**Problème** : Servir les fichiers avec contrôle d'accès, cache HTTP, transformation à la volée.

**Fonctionnement** :
- `storage.proxy.handler()` : handler HTTP à monter sur Express/Hono/Fastify/Next.
- `authorize: (req, file) => boolean` : vérifie si l'user peut accéder. Si false → 403.
- `cache` : règles par pattern. `'public/*': { maxAge: '1y' }`, `'documents/*': { private: true }`.
- `mode: 'stream'` (défaut) : stream le fichier depuis le storage. `mode: 'redirect'` : redirect vers une signed URL (moins de bande passante serveur).
- Si plugin-transform actif : `?w=64&h=64&f=webp` pour transformation à la volée.
- Accès par ID : `GET /files/file_01J8X...` → registry.getById → download → stream.
- **Cross-plugin** : `file.proxy?.cache` dans le FileRecord override la config globale (cache par fichier).

### plugin-quota (P1)

**Problème** : Limiter le stockage par user ou tenant.

**Fonctionnement** :
- Règles : `{ scope: 'user', getId: (ctx) => ctx.userId, limit: 1024*1024*1024, prefix: 'uploads/' }`.
- Avant chaque upload : vérification. Si dépassé → `StorageQuotaError`.
- `storage.quota.getUsage(userId)`, `storage.quota.isExceeded(userId)`.
- **FileRecordExtensions** : `file.quota?.countedBytes` pour l'agrégation via registry.

### plugin-transform (P1)

**Problème** : Générer des variantes (thumb, medium) à l'upload ou à la volée.

**Fonctionnement** :
- Variants : `{ avatar: [{ original }, { thumb: { width: 64, height: 64 } }, { medium: { width: 256 } }] }`.
- À l'upload avec `variant: 'avatar'` : génère toutes les variantes (Sharp, Cloudflare Images, imgproxy).
- **FileRecordExtensions** : `file.transform?.variants` : `{ thumb: { id, key }, medium: { id, key } }`.
- Proxy : `GET /files/file_ABC?variant=thumb` → sert la variante sans re-transformer.

### plugin-scan (P2)

**Problème** : Scanner les fichiers pour malware avant de les rendre accessibles.

**Fonctionnement** :
- Adapters : ClamAV, VirusTotal, MetaDefender.
- `quarantine: { prefix, ttl }` : fichiers mis en quarantine avant validation.
- `strategy: 'async'` : upload retourne immédiatement, scan en background. `sync` : bloque jusqu'au résultat.
- `onClean` : déplacer vers le bucket final. `onInfected` : supprimer, notifier.
- **Cross-plugin** : `file.scan?.status` dans le FileRecord. Proxy avec `scanGate: true` refuse de servir si status !== 'clean'.

### plugin-lifecycle (P2)

**Problème** : Suppression automatique (temp après 24h), archivage (invoices après 1 an vers cold storage).

**Fonctionnement** :
- Règles : `{ match: 'temp/*', after: '24h', action: 'delete' }`, `{ match: 'invoices/*', after: '1y', action: 'move', destination: s3ColdAdapter }`.
- `storage.lifecycle.run()` : exécution (cron). `storage.lifecycle.preview()` : ce qui serait affecté.
- **Cross-plugin** : `file.lifecycle?.expiresAt` dans le FileRecord pour override par fichier.

### plugin-multi (P2)

**Problème** : Réplication (écrire sur plusieurs buckets), routing (images → Cloudflare, docs → S3), fallback.

**Fonctionnement** :
- `strategy: 'replicate'` : écrit sur tous les adapters en parallèle.
- `strategy: 'route'` : `route: (key, file) => file.mimeType.startsWith('image/') ? 'images' : 'default'`. Adapters par nom.
- `strategy: 'fallback'` : primary, si échec → fallback, si échec → dernier recours.

### plugin-audit (P1)

**Problème** : Tracer qui a uploadé/téléchargé/supprimé quoi.

**Fonctionnement** :
- Chaque upload, download, delete → `audit.log('file.uploaded', ...)`.
- Bridge better-audit. Contexte HTTP injecté si middleware.
- `storage.audit.getHistory(key)`, `storage.audit.getActivityByUser(userId)`.

---

## Cross-plugin : FileRecordExtensions

Chaque plugin étend `FileRecordExtensions` via declaration merging. Champs **top-level**, pas dans `metadata` :

```ts
// plugin-proxy
declare module "better-storage" {
  interface FileRecordExtensions {
    proxy?: { cache?: { maxAge: string }; disposition?: "inline" | "attachment" };
  }
}

// Usage
file.proxy?.cache?.maxAge;  // override par fichier
```

Interactions :
- `registry` + `proxy` : cache défini par fichier dans l'attachment.
- `scan` + `proxy` : scanGate lit `file.scan?.status`.
- `transform` + `registry` : variants enregistrés, proxy route par variant.
- `quota` + `registry` : usage agrégé depuis `file.quota?.countedBytes`.

---

## Adapters (providers)

| Adapter | Provider | Usage |
|---------|----------|-------|
| `adapter-local` | Filesystem | Dev |
| `adapter-s3` | AWS S3, MinIO, Scaleway | Prod |
| `adapter-r2` | Cloudflare R2 | Prod, zero egress |
| `adapter-vercel-blob` | Vercel Blob | Vercel deploy |
| `adapter-supabase` | Supabase Storage | Stack Supabase |

---

## Structure des packages (kebab-case)

```
packages/
  core/
  types/
  errors/
  adapter-s3/
  adapter-r2/
  adapter-local/
  adapter-vercel-blob/
  adapter-supabase/
  plugin-url/
  plugin-keys/
  plugin-validate/
  plugin-quota/
  plugin-transform/
  plugin-multipart/
  plugin-scan/
  plugin-lifecycle/
  plugin-multi/
  plugin-registry/
  plugin-proxy/
  plugin-audit/
  client/
  cli/
```

---

## TODO

- [ ] Scaffold monorepo
- [ ] Core : types, adapter, FileRecordExtensions
- [ ] adapter-local, adapter-s3, adapter-r2
- [ ] plugin-keys, plugin-url, plugin-validate
- [ ] plugin-registry, plugin-proxy
- [ ] plugin-quota, plugin-transform
- [ ] plugin-audit
- [ ] plugin-multipart, plugin-scan, plugin-lifecycle
- [ ] plugin-multi
- [ ] adapter-vercel-blob, adapter-supabase
- [ ] client, cli
- [ ] Conformance, E2E, docs
