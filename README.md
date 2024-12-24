# noble-post-quantum

Auditable & minimal JS implementation of public-key post-quantum cryptography.

- 🔒 Auditable
- 🔻 Tree-shakeable: unused code is excluded from your builds
- 🔍 Reliable: tests ensure correctness
- 🦾 ML-KEM & CRYSTALS-Kyber: lattice-based kem from FIPS-203
- 🔋 ML-DSA & CRYSTALS-Dilithium: lattice-based signatures from FIPS-204
- 🐈 SLH-DSA & SPHINCS+: hash-based Winternitz signatures from FIPS-205
- 🪶 37KB (15KB gzipped) for everything with bundled hashes

Take a glance at [GitHub Discussions](https://github.com/paulmillr/noble-post-quantum/discussions) for questions and support.

> [!IMPORTANT]
> NIST published [IR 8547](https://nvlpubs.nist.gov/nistpubs/ir/2024/NIST.IR.8547.ipd.pdf),
> prohibiting classical cryptography (RSA, DSA, ECDSA, ECDH) after 2035.
> Australian ASD does same thing [after 2030](https://www.cyber.gov.au/resources-business-and-government/essential-cyber-security/ism/cyber-security-guidelines/guidelines-cryptography).
> Take it into an account while designing a new cryptographic system.

### This library belongs to _noble_ cryptography

> **noble cryptography** — high-security, easily auditable set of contained cryptographic libraries and tools.

- Zero or minimal dependencies
- Highly readable TypeScript / JS code
- PGP-signed releases and transparent NPM builds
- All libraries:
  [ciphers](https://github.com/paulmillr/noble-ciphers),
  [curves](https://github.com/paulmillr/noble-curves),
  [hashes](https://github.com/paulmillr/noble-hashes),
  [post-quantum](https://github.com/paulmillr/noble-post-quantum),
  4kb [secp256k1](https://github.com/paulmillr/noble-secp256k1) /
  [ed25519](https://github.com/paulmillr/noble-ed25519)
- [Check out homepage](https://paulmillr.com/noble/)
  for reading resources, documentation and apps built with noble

## Usage

> npm install @noble/post-quantum

We support all major platforms and runtimes.
For [Deno](https://deno.land), ensure to use
[npm specifier](https://deno.land/manual@v1.28.0/node/npm_specifiers).
For React Native, you may need a
[polyfill for getRandomValues](https://github.com/LinusU/react-native-get-random-values).
A standalone file
[noble-post-quantum.js](https://github.com/paulmillr/noble-post-quantum/releases) is also available.

```js
// import * from '@noble/post-quantum'; // Error: use sub-imports instead
import { ml_kem512, ml_kem768, ml_kem1024 } from '@noble/post-quantum/ml-kem';
import { ml_dsa44, ml_dsa65, ml_dsa87 } from '@noble/post-quantum/ml-dsa';
import {
  slh_dsa_sha2_128f, slh_dsa_sha2_128s,
  slh_dsa_sha2_192f, slh_dsa_sha2_192s,
  slh_dsa_sha2_256f, slh_dsa_sha2_256s,
  slh_dsa_shake_128f, slh_dsa_shake_128s,
  slh_dsa_shake_192f, slh_dsa_shake_192s,
  slh_dsa_shake_256f, slh_dsa_shake_256s,
} from '@noble/post-quantum/slh-dsa';
// import { ml_kem768 } from 'npm:@noble/post-quantum@0.1.0/ml-kem'; // Deno
```

- [ML-KEM / Kyber](#ml-kem--kyber-shared-secrets)
- [ML-DSA / Dilithium](#ml-dsa--dilithium-signatures)
- [SLH-DSA / SPHINCS+](#slh-dsa--sphincs-signatures)
- [What should I use?](#what-should-i-use)
- [Security](#security)
- [Speed](#speed)
- [Contributing & testing](#contributing--testing)
- [License](#license)

### ML-KEM / Kyber shared secrets

```ts
import { ml_kem512, ml_kem768, ml_kem1024 } from '@noble/post-quantum/ml-kem';
import { randomBytes } from '@noble/post-quantum/utils';

// 1. [Alice] generates secret & public keys, then sends publicKey to Bob
const seed = randomBytes(64); // seed is optional
const aliceKeys = ml_kem768.keygen(seed);

// 2. [Bob] generates shared secret for Alice publicKey
// bobShared never leaves [Bob] system and is unknown to other parties
const { cipherText, sharedSecret: bobShared } = ml_kem768.encapsulate(aliceKeys.publicKey);

// 3. [Alice] gets and decrypts cipherText from Bob
const aliceShared = ml_kem768.decapsulate(cipherText, aliceKeys.secretKey);

// Now, both Alice and Both have same sharedSecret key
// without exchanging in plainText: aliceShared == bobShared

// Warning: Can be MITM-ed
const carolKeys = kyber1024.keygen();
const carolShared = kyber1024.decapsulate(cipherText, carolKeys.secretKey); // No error!
notDeepStrictEqual(aliceShared, carolShared); // Different key!
```

Lattice-based key encapsulation mechanism, defined in [FIPS-203](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf).

See [website](https://www.pq-crystals.org/kyber/resources.shtml) and [repo](https://github.com/pq-crystals/kyber).
There are some concerns with regards to security: see
[djb blog](https://blog.cr.yp.to/20231003-countcorrectly.html) and
[mailing list](https://groups.google.com/a/list.nist.gov/g/pqc-forum/c/W2VOzy0wz_E).
Old, incompatible version (Kyber) is not provided. Open an issue if you need it.

> [!WARNING]
> Unlike ECDH, KEM doesn't verify whether it was "Bob" who've sent the ciphertext.
> Instead of throwing an error when the ciphertext is encrypted by a different pubkey,
> `decapsulate` will simply return a different shared secret.
> ML-KEM is also probabilistic and relies on quality of CSPRNG.

### ML-DSA / Dilithium signatures

```ts
import { ml_dsa44, ml_dsa65, ml_dsa87 } from '@noble/post-quantum/ml-dsa';
import { utf8ToBytes, randomBytes } from '@noble/post-quantum/utils';
const seed = randomBytes(32); // seed is optional
const keys = ml_dsa65.keygen(seed);
const msg = utf8ToBytes('hello noble');
const sig = ml_dsa65.sign(keys.secretKey, msg);
const isValid = ml_dsa65.verify(keys.publicKey, msg, sig);
```

Lattice-based digital signature algorithm, defined in [FIPS-204](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf). See
[website](https://www.pq-crystals.org/dilithium/index.shtml) and
[repo](https://github.com/pq-crystals/dilithium).
The internals are similar to ML-KEM, but keys and params are different.

### SLH-DSA / SPHINCS+ signatures

```ts
import {
  slh_dsa_sha2_128f, slh_dsa_sha2_128s,
  slh_dsa_sha2_192f, slh_dsa_sha2_192s,
  slh_dsa_sha2_256f, slh_dsa_sha2_256s,
  slh_dsa_shake_128f, slh_dsa_shake_128s,
  slh_dsa_shake_192f, slh_dsa_shake_192s,
  slh_dsa_shake_256f, slh_dsa_shake_256s,
} from '@noble/post-quantum/slh-dsa';
import { utf8ToBytes } from '@noble/post-quantum/utils';

const keys2 = sph.keygen();
const msg2 = utf8ToBytes('hello noble');
const sig2 = sph.sign(keys2.secretKey, msg2);
const isValid2 = sph.verify(keys2.publicKey, msg2, sig2);
```

Hash-based digital signature algorithm, defined in [FIPS-205](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.205.pdf).
See [website](https://sphincs.org) and [repo](https://github.com/sphincs/sphincsplus). We implement spec v3.1 with FIPS adjustments.

There are many different kinds,
but basically `sha2` / `shake` indicate internal hash, `128` / `192` / `256` indicate security level, and `s` /`f` indicate trade-off (Small / Fast).
SLH-DSA is slow: see [benchmarks](#speed) for key size & speed.

### What should I use?

|           | Speed  | Key size    | Sig size    | Created in | Popularized in | Post-quantum? |
| --------- | ------ | ----------- | ----------- | ---------- | -------------- | ------------- |
| RSA       | Normal | 256B - 2KB  | 256B - 2KB  | 1970s      | 1990s          | No            |
| ECC       | Normal | 32 - 256B   | 48 - 128B   | 1980s      | 2010s          | No            |
| ML-KEM    | Fast   | 1.6 - 31KB  | 1KB         | 1990s      | 2020s          | Yes           |
| ML-DSA    | Normal | 1.3 - 2.5KB | 2.5 - 4.5KB | 1990s      | 2020s          | Yes           |
| SLH-DSA   | Slow   | 32 - 128B   | 17 - 50KB   | 1970s      | 2020s          | Yes           |
| FN-DSA    | Slow   | 0.9 - 1.8KB | 0.6 - 1.2KB | 1990s      | 2020s          | Yes           |

We suggest to use ECC + ML-KEM for key agreement, ECC + SLH-DSA for signatures.

ML-KEM and ML-DSA are lattice-based. SLH-DSA is hash-based, which means it is built on top of older, more conservative primitives. NIST guidance for security levels:

- Category 3 (~AES-192): ML-KEM-768, ML-DSA-65, SLH-DSA-[SHA2/shake]-192[s/f]
- Category 5 (~AES-256): ML-KEM-1024, ML-DSA-87, SLH-DSA-[SHA2/shake]-256[s/f]

NIST recommends to use cat-3+, while australian [ASD only allows cat-5 after 2030](https://www.cyber.gov.au/resources-business-and-government/essential-cyber-security/ism/cyber-security-guidelines/guidelines-cryptography).

For [hashes](https://github.com/paulmillr/noble-hashes), use SHA512 or SHA3-512 (not SHA256); and for [ciphers](https://github.com/paulmillr/noble-ciphers) ensure AES-256 or ChaCha.

## Security

The library has not been independently audited yet.

There is no protection against side-channel attacks.
Keep in mind that even hardware versions ML-KEM [are vulnerable](https://eprint.iacr.org/2023/1084).

If you see anything unusual: investigate and report.

## Speed

Noble is the fastest JS implementation of post-quantum algorithms.
WASM libraries can be faster.

| OPs/sec           | Keygen | Signing | Verification | Shared secret |
| ----------------- | ------ | ------- | ------------ | ------------- |
| ECC x/ed25519     | 10270  | 5110    | 1050         | 1470          |
| ML-KEM-768        | 2300   |         |              | 2000          |
| ML-DSA65          | 386    | 120     | 367          |               |
| SLH-DSA-SHA2-192f | 166    | 6       | 111          |               |

For SLH-DSA, SHAKE slows everything down 8x, and -s versions do another 20-50x slowdown.

Detailed benchmarks on Apple M2:

```
ML-KEM
keygen
├─ML-KEM-512 x 3,784 ops/sec @ 264μs/op
├─ML-KEM-768 x 2,305 ops/sec @ 433μs/op
└─ML-KEM-1024 x 1,510 ops/sec @ 662μs/op
encrypt
├─ML-KEM-512 x 3,283 ops/sec @ 304μs/op
├─ML-KEM-768 x 1,993 ops/sec @ 501μs/op
└─ML-KEM-1024 x 1,366 ops/sec @ 731μs/op
decrypt
├─ML-KEM-512 x 3,450 ops/sec @ 289μs/op
├─ML-KEM-768 x 2,035 ops/sec @ 491μs/op
└─ML-KEM-1024 x 1,343 ops/sec @ 744μs/op

ML-DSA
keygen
├─ML-DSA44 x 669 ops/sec @ 1ms/op
├─ML-DSA65 x 386 ops/sec @ 2ms/op
└─ML-DSA87 x 236 ops/sec @ 4ms/op
sign
├─ML-DSA44 x 123 ops/sec @ 8ms/op
├─ML-DSA65 x 120 ops/sec @ 8ms/op
└─ML-DSA87 x 78 ops/sec @ 12ms/op
verify
├─ML-DSA44 x 618 ops/sec @ 1ms/op
├─ML-DSA65 x 367 ops/sec @ 2ms/op
└─ML-DSA87 x 220 ops/sec @ 4ms/op
```

SLH-DSA (_shake is 8x slower):

|           | sig size | keygen | sign   | verify |
|-----------|----------|--------|--------|--------|
| sha2_128f | 18088    | 4ms    | 90ms   | 6ms    |
| sha2_128s | 7856     | 260ms  | 2000ms | 2ms    |
| sha2_192f | 35664    | 6ms    | 160ms  | 9ms    |
| sha2_192s | 16224    | 380ms  | 3800ms | 3ms    |
| sha2_256f | 49856    | 15ms   | 340ms  | 9ms    |
| sha2_256s | 29792    | 250ms  | 3400ms | 4ms    |

## Contributing & testing

* `npm install && npm run build && npm test` will build the code and run tests.
* `npm run lint` / `npm run format` will run linter / fix linter issues.
* `npm run bench` will run benchmarks, which may need their deps first (`npm run bench:install`)
* `cd build && npm install && npm run build:release` will build single file

Check out [github.com/paulmillr/guidelines](https://github.com/paulmillr/guidelines)
for general coding practices and rules.

See [paulmillr.com/noble](https://paulmillr.com/noble/)
for useful resources, articles, documentation and demos
related to the library.

## License

The MIT License (MIT)

Copyright (c) 2024 Paul Miller [(https://paulmillr.com)](https://paulmillr.com)

See LICENSE file.
