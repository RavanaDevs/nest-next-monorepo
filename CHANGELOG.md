# 1.0.0 (2026-10-15)


### Bug Fixes

* **api:** enforce authentication, authorization and input validation ([ced381f](https://github.com/RavanaDevs/nest-next-monorepo/commit/ced381fbeb13faf238bc032e997d7d8294a53eae))
* **api:** harden error responses, logging and env validation ([2b459fe](https://github.com/RavanaDevs/nest-next-monorepo/commit/2b459feecf434adbd21833250ff65980fd99444a))
* change clerk id to external id ([2f7507a](https://github.com/RavanaDevs/nest-next-monorepo/commit/2f7507ab6fe1e9cddb8927c7334684e2a7866640))
* **contracts:** build to CommonJS instead of shipping raw TypeScript ([e914d4b](https://github.com/RavanaDevs/nest-next-monorepo/commit/e914d4b81c423f286adc21541b1ddd6053c46ab5))
* gracefull shutdown blockers ([5404cad](https://github.com/RavanaDevs/nest-next-monorepo/commit/5404cad5165d44067bb4a6d6fdb8ebdf14b55bf4))
* jest root dir ([8339bbd](https://github.com/RavanaDevs/nest-next-monorepo/commit/8339bbde2a1c4efbcdd81a84a1131ca713a58295))
* **web:** stop turbo racing typecheck against next build ([29e9ba0](https://github.com/RavanaDevs/nest-next-monorepo/commit/29e9ba0fb4925b49c54195150455d9b81982a81e))


### Continuous Integration

* **release:** automate versioning with semantic-release ([1c4b134](https://github.com/RavanaDevs/nest-next-monorepo/commit/1c4b1342d5dc8a52e8e7257501776208cee9cd92))


### Features

* **api:** decouple users domain with port and drizzle adapter (phase 3) ([e0e4313](https://github.com/RavanaDevs/nest-next-monorepo/commit/e0e4313774b34dfab6628ae6e6afe762ebd1a4b1))
* **api:** implement 3-tier user routes with swagger and zod validation ([fc818d0](https://github.com/RavanaDevs/nest-next-monorepo/commit/fc818d0d7a7619dcf7804320604af3e3834476e4))
* **api:** implement clerk authentication, svix webhooks, casl and security hardening ([79f15d5](https://github.com/RavanaDevs/nest-next-monorepo/commit/79f15d553b260ea09ed414e6317e2be6c867b9f3))
* **api:** implement env validation, pino logger, and problem details filter ([8bdff74](https://github.com/RavanaDevs/nest-next-monorepo/commit/8bdff7472b2b8f233f8806c6266a1ca693fd4438))
* **api:** implement graceful shutdown sequence and terminus health probes ([8cc78be](https://github.com/RavanaDevs/nest-next-monorepo/commit/8cc78beffa544aabfec7be82cf3de06730299408))
* **api:** initial commit ([49da4c7](https://github.com/RavanaDevs/nest-next-monorepo/commit/49da4c70bcfb6c9837097c8fe6df72a06af80064))
* **api:** make auth provider-neutral with strategy pattern and dynamic AuthModule (phase 4) ([8df4fd3](https://github.com/RavanaDevs/nest-next-monorepo/commit/8df4fd3883f7c6e1af075ad335965960576f232c))
* **api:** setup path aliases and architectural lint boundaries (phase 0) ([ae5408b](https://github.com/RavanaDevs/nest-next-monorepo/commit/ae5408bf3b09d099d6b0ad9d67c7fb7760348308))
* **api:** wire drizzle orm with postgres database module ([7856c66](https://github.com/RavanaDevs/nest-next-monorepo/commit/7856c66e09cd767cf6ddff5957e533fa930a435e))
* central config for lint, format, and test ([94dc888](https://github.com/RavanaDevs/nest-next-monorepo/commit/94dc8885f9710515b46e2820ed00850dc82f7a04))
* configure commitlint ([c2a5fbf](https://github.com/RavanaDevs/nest-next-monorepo/commit/c2a5fbf8b8364004fe303da58b0a92ea228fe2c4))
* **contracts:** add shared zod schemas for pagination and users domain ([054c06b](https://github.com/RavanaDevs/nest-next-monorepo/commit/054c06ba7302bc91277565a175864c142f56b4f3))
* initial commit ([436369a](https://github.com/RavanaDevs/nest-next-monorepo/commit/436369a7efc72969e7f2fb9253082ed4606bab14))
* integrate Clerk authentication and implement type-safe API client for backend communication ([2288c9d](https://github.com/RavanaDevs/nest-next-monorepo/commit/2288c9d687acb9093d2e8ac3b5bf7cae30ac9082))


### BREAKING CHANGES

* **release:** Node 22.14 or newer is now required.

## 1.0.0 (2026-08-11)
