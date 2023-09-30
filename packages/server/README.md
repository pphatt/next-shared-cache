# @neshca/server

This is an example of a server that uses the `@neshca/cache-handler` package.

## Installation

### @neshca/server

To install the `@neshca/server` package:

```sh
npm install -D @neshca/server
```

### @neshca/cache-handler in your Next.js app

To install the `@neshca/cache-handler` package:

```sh
npm install -D @neshca/cache-handler
```

## Usage

### @neshca/server

Run server:

```sh
npx @neshca/server
```

### @neshca/cache-handler in your Next.js app

Create a file called `cache-handler.js` next to you `next.config.js` with the following contents:

```js
const { IncrementalCache } = require('@neshca/cache-handler');
const { reviveFromBase64Representation, replaceJsonWithBase64 } = require('@neshca/json-replacer-reviver');
const { fetch } = require('undici');

const baseUrl = process.env.BASE_URL ?? 'http://localhost:8080';

IncrementalCache.prefix = 'cache-testing:';

IncrementalCache.cache = {
    async get(key) {
        const result = await fetch(`${baseUrl}/get?${new URLSearchParams({ key })}`);

        if (!result.ok) {
            return null;
        }

        try {
            const string = await result.text();

            return JSON.parse(string, reviveFromBase64Representation);
        } catch (error) {
            return null;
        }
    },
    async set(key, value, ttl) {
        await fetch(`${baseUrl}/set`, {
            method: 'POST',
            body: JSON.stringify([key, JSON.stringify(value, replaceJsonWithBase64), ttl]),
            contentType: 'text/plain',
        });
    },
    async getTagsManifest() {
        const response = await fetch(`${baseUrl}/getTagsManifest`);

        if (!response.ok) {
            return { version: 1, items: {} };
        }

        const json = await response.json();

        return json;
    },
    async revalidateTag(_prefix, tag, revalidatedAt) {
        await fetch(`${baseUrl}/revalidateTag`, {
            method: 'POST',
            body: JSON.stringify([tag, revalidatedAt]),
            contentType: 'text/plain',
        });
    },
};
module.exports = IncrementalCache;
```

Then, use the following configuration in your `next.config.js` file:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
    experimental: {
        incrementalCacheHandlerPath: require.resolve('./cache-handler'), // path to the cache handler file you created
        isrFlushToDisk: false, // disable writing cache to disk
    },
};

module.exports = nextConfig;
```