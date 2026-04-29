# nestjs-multi-storage

A multi-storage filesystem manager for NestJS, based on `fs` function names. Supports local filesystem and Amazon S3-compatible storage backends (e.g. MinIO, Ceph, AWS S3).

## Installation

```bash
npm install nestjs-multi-storage
```

## Quick Start

### 1. Register the module

**Filesystem storage**

```typescript
import { StorageModule } from 'nestjs-multi-storage';

@Module({
  imports: [
    StorageModule.forRoot({
      type: 'fileSystem',
      prefix: '/var/data', // root directory for all operations
    }),
  ],
})
export class AppModule {}
```

**S3-compatible storage (AWS S3, MinIO, …)**

```typescript
import { StorageModule } from 'nestjs-multi-storage';

@Module({
  imports: [
    StorageModule.forRoot({
      type: 's3',
      endpoint: 'http://minio:9000',         // internal service endpoint (ideal for SSR / server-side)
      endpointCDN: 'https://cdn.example.com', // optional: public CDN endpoint for signed URL generation
      region: 'us-east-1',
      bucket: 'my-bucket',
      accessKeyId: 'ACCESS_KEY',
      secretAccessKey: 'SECRET_KEY',
    }),
  ],
})
export class AppModule {}
```

> **Tip – SSR / internal routing:** When running Next.js or another SSR framework alongside your NestJS API, point `endpoint` to the **internal** service URL (e.g. `http://minio:9000` in Docker Compose) instead of routing through the public domain → Ingress → Gateway chain. This eliminates unnecessary network hops and keeps SSR-side storage calls stable regardless of external DNS or TLS configuration.

### 2. Inject and use the service

```typescript
import { Injectable } from '@nestjs/common';
import { StorageService } from 'nestjs-multi-storage';

@Injectable()
export class FileService {
  constructor(private readonly storage: StorageService) {}

  async saveJson(filePath: string, data: object): Promise<void> {
    await this.storage.writeFile(filePath, JSON.stringify(data));
  }

  async loadJson(filePath: string): Promise<object> {
    const buffer = await this.storage.readFile(filePath);
    return JSON.parse(buffer.toString('utf-8'));
  }
}
```

### Async / dynamic configuration

```typescript
StorageModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: config.get<'s3' | 'fileSystem'>('STORAGE_TYPE'),
    endpoint: config.get('S3_ENDPOINT'),          // internal endpoint
    endpointCDN: config.get('S3_ENDPOINT_CDN'),   // public CDN endpoint (optional)
    region: config.get('S3_REGION'),
    bucket: config.get('S3_BUCKET'),
    accessKeyId: config.get('S3_ACCESS_KEY_ID'),
    secretAccessKey: config.get('S3_SECRET_ACCESS_KEY'),
    prefix: config.get('STORAGE_PREFIX'),
  }),
}),
```

### Global module

Pass `isGlobal: true` to make the module available application-wide without re-importing it.

```typescript
StorageModule.forRoot({ ..., isGlobal: true })
StorageModule.forRootAsync({ ..., isGlobal: true })
```

## API Reference

All methods are available on `StorageService`. The optional `bucket` parameter overrides the default bucket configured in the module options (S3 only).

| Method | Signature | Description |
|---|---|---|
| `mkdir` | `(folderPath, bucket?) → Promise<...>` | Create a directory / S3 prefix |
| `readdir` | `(folderPath, bucket?) → Promise<string[]>` | List directory contents |
| `rmdir` | `(folderPath, bucket?) → Promise<void>` | Remove a directory recursively |
| `exists` | `(path, bucket?) → Promise<boolean>` | Check whether a file or directory exists |
| `readFile` | `(filePath, bucket?) → Promise<Buffer>` | Read a file into memory. When `endpointCDN` is configured, a signed URL is generated and the file is fetched directly from the CDN host. |
| `writeFile` | `(filePath, data, bucket?) → Promise<void>` | Write a string or Buffer to a file |
| `rm` | `(filePath, bucket?) → Promise<void>` | Delete a single file |
| `createReadStream` | `(filePath, bucket?) → ReadStream \| Readable` | Open a readable stream |
| `createWriteStream` | `(filePath, options?, bucket?) → WriteStream \| PassThrough` | Open a writable stream (min `highWaterMark`: 5 MB for S3 multipart upload) |

### `s3client` getter

Exposes the underlying `S3Client` instance for advanced use-cases:

```typescript
const raw = this.storage.s3client; // S3Client | undefined
```

## Configuration options

| Option | Type | Required | Description |
|---|---|---|---|
| `type` | `'fileSystem' \| 's3'` | ✅ | Storage backend |
| `prefix` | `string` | | Root directory prefix (filesystem) or key prefix (S3) |
| `endpoint` | `string` | S3 | S3-compatible endpoint URL (use internal service URL for server-side calls) |
| `endpointCDN` | `string` | | Public CDN base URL; if set, `readFile` generates a signed URL against this host |
| `region` | `string` | S3 | AWS / S3 region |
| `bucket` | `string` | S3 | Default bucket name |
| `accessKeyId` | `string` | S3 | Access key ID |
| `secretAccessKey` | `string` | S3 | Secret access key |

## License

MIT
