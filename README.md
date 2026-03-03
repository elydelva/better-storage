# better-storage

A headless, **pure TypeScript** storage engine. Unified file storage with upload, download, signed URLs, and adapters — without coupling to a specific framework or database. Same philosophy as better-auth — but for storage.

## Install

```bash
bun add @better-storage/core @better-storage/adapter-local
```

## Usage

```ts
import { createStorage } from "@better-storage/core";
import { createLocalAdapter } from "@better-storage/adapter-local";

const storage = createStorage({
  adapter: createLocalAdapter({ path: "./uploads" }),
});

const file = await storage.upload(buffer, { mimeType: "image/jpeg" });
const stream = await storage.download(file.key);
```

## Packages

| Package | Description |
|---------|-------------|
| `@better-storage/core` | Engine, upload, download, delete, adapter interface |
| `@better-storage/types` | StorageFile, UploadOptions |
| `@better-storage/errors` | StorageError, ValidationError |
| `@better-storage/adapter-local` | Local filesystem |
| `@better-storage/adapter-s3` | AWS S3 |
| `@better-storage/adapter-r2` | Cloudflare R2 |
| `@better-storage/plugin-url` | Public and signed URLs |
| `@better-storage/plugin-keys` | Key generation |
| `@better-storage/plugin-validate` | MIME, size, dimensions |
| `@better-storage/plugin-registry` | DB registry with ID |
| `@better-storage/plugin-proxy` | HTTP proxy with auth |
| `@better-storage/client` | HTTP client |
| `@better-storage/cli` | CLI |

## Development

```bash
bun install
bun run build
bun run test
bun run lint
bun run typecheck
```

## License

MIT — see [LICENSE](LICENSE)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## Code of Conduct

See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)
