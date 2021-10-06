# Estrutura de cache

O cache é usado qnd o backend tem q retornar pro usuário o mesmo resultado da request. Se o usuário faz 20 vezes a msm request, o correto é só consultar 1 vez no banco, e o restante ser retorno do cache.

Instale `ioredis`: `yarn add ioredis`

## src/lib/Cache.js

```js
import Redis from "ioredis";

class Cache {
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT,
      keyPrefix: "cache:",
    });
  }

  set(key, value) {
    // Deleta cache depois de 24h (60s * 60min * 24h)
    // EX = excluir
    return this.redis.set(key, JSON.stringify(value), "EX", 60 * 60 * 24);
  }

  async get(key) {
    const cached = await this.redis.get(key);

    return cached ? JSON.parse(cached) : null;
  }

  invalidate(key) {
    return this.redis.del(key);
  }
}

export default new Cache();
```
