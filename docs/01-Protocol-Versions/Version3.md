# Paseto Version 3

## GetNonce

Throw an exception. We don't do this in version 3.

## Encrypt

Given a message `m`, key `k`, and optional footer `f` (which defaults to empty 
string), and an optional implicit assertion `i` (which defaults to empty string):

1. Before encrypting, first assert that the key being used is intended for use
   with `v3.local` tokens, and has a length of 256 bits (32 bytes).
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set header `h` to `v3.local.`
3. Generate 32 random bytes from the OS's CSPRNG to get the nonce, `n`.
4. Split the key into an Encryption key (`Ek`) and Authentication key (`Ak`),
   using HKDF-HMAC-SHA384, with `n` appended to the info rather than the salt.
    * The output length **MUST** be 48 for both key derivations.
    * The derived key will be the leftmost 32 bytes of the first HKDF derivation.
   
   The remaining 16 bytes of the first key derivation (from which `Ek` is derived)
   will be used as a counter nonce (`n2`):
   ```
   tmp = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-encryption-key" || n,
       salt = NULL
   );
   Ek = tmp[0:32]
   n2 = tmp[32:]
   Ak = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-auth-key-for-aead" || n,
       salt = NULL
   );
   ```
5. Encrypt the message using `AES-256-CTR`, using `Ek` as the key and `n2` as the nonce.
   We'll call the encrypted output of this step `c`:
   ```
   c = aes256ctr_encrypt(
       plaintext = m,
       nonce = n2
       key = Ek
   );
   ```
6. Pack `h`, `n`, `c`, `f`, and `i` together using
   [PAE](Common.md#authentication-padding)
   (pre-authentication encoding). We'll call this `preAuth`.
7. Calculate HMAC-SHA384 of the output of `preAuth`, using `Ak` as the
   authentication key. We'll call this `t`.
8. If `f` is:
    * Empty: return "`h` || base64url(`n` || `c` || `t`)"
    * Non-empty: return "`h` || base64url(`n` || `c` || `t`) || `.` || base64url(`f`)"
    * ...where || means "concatenate"
    * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

## Decrypt

Given a message `m`, key `k`, and optional footer `f`
(which defaults to empty string), and an optional
implicit assertion `i` (which defaults to empty string):

1. Before decrypting, first assert that the key being used is intended for use
   with `v3.local` tokens, and has a length of 256 bits (32 bytes).
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
   * If `f` is allowed to be a JSON-encoded blob, implementations **SHOULD** allow
     users to provide guardrails against invalid JSON tokens.
     See [this document](../02-Implementation-Guide/01-Payload-Processing.md#optional-footer)
     for specific guidance and example code.
3. Verify that the message begins with `v3.local.`, otherwise throw an
   exception. This constant will be referred to as `h`.
   * **Future-proofing**: If a future PASETO variant allows for encodings other
     than JSON (e.g., CBOR), future implementations **MAY** also permit those
     values at this step (e.g. `v3c.local.`).
4. Decode the payload (`m` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
    * `n` to the leftmost 32 bytes
    * `t` to the rightmost 48 bytes
    * `c` to the middle remainder of the payload, excluding `n` and `t`
5. Split the key (`k`) into an Encryption key (`Ek`) and an Authentication key
   (`Ak`), `n` appended to the HKDF info.
    * For encryption keys, the **info** parameter for HKDF **MUST** be set to
      **paseto-encryption-key**.
    * For authentication keys, the **info** parameter for HKDF **MUST** be set to
      **paseto-auth-key-for-aead**.
    * The output length **MUST** be 48 for both key derivations.
      The leftmost 32 bytes of the first key derivation will produce `Ek`, while
      the remaining 16 bytes will be the AES nonce `n2`.

   ```
   tmp = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-encryption-key" || n,
       salt = NULL
   );
   Ek = tmp[0:32]
   n2 = tmp[32:]
   Ak = hkdf_sha384(
       len = 48,
       ikm = k,
       info = "paseto-auth-key-for-aead" || n,
       salt = NULL
   );
   ```
6. Pack `h`, `n`, `c`, `f`, and `i` together (in that order) using
   [PAE](Common.md#authentication-padding).
   We'll call this `preAuth`.
7. Recalculate HMAC-SHA-384 of `preAuth` using `Ak` as the key. We'll call this `t2`.
8. Compare `t` with `t2` using a constant-time string compare function. If they
   are not identical, throw an exception.
   * You **MUST** use a constant-time string compare function to be compliant.
     If you do not have one available to you in your programming language/framework,
     you MUST use [Double HMAC](https://paragonie.com/blog/2015/11/preventing-timing-attacks-on-string-comparison-with-double-hmac-strategy).
   * Common utilities that were not intended for cryptographic comparisons, such as 
     Java's `Array.equals()` or PHP's `==` operator, are explicitly forbidden.
9. Decrypt `c` using `AES-256-CTR`, using `Ek` as the key and `n2` as the nonce,
   then return the plaintext.
   ```
   return aes256ctr_decrypt(
       cipherext = c,
       nonce = n2
       key = Ek
   );
   ```

## Sign

Given a message `m`, 384-bit ECDSA secret key `sk`, an optional footer `f` 
(which defaults to empty string), and an optional implicit assertion `i`
(which defaults to empty string):

1. Before signing, first assert that the key being used is intended for use
   with `v3.public` tokens, and is the secret key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set `h` to `v3.public.`
3. Pack `pk`, `h`, `m`, `f`, and `i` together using
   [PAE](Common.md#authentication-padding)
   (pre-authentication encoding). We'll call this `m2`.
   * Note: `pk` is the public key corresponding to `sk` (which **MUST** use
     [point compression](https://www.secg.org/sec1-v2.pdf)). `pk` **MUST** be 49
     bytes long, and the first byte **MUST** be `0x02` or `0x03` (depending on 
     [the least significant bit of Y](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.202.2977&rep=rep1&type=pdf);
     section 4.3.6, step 2.2).
     The remaining bytes **MUST** be the X coordinate, using big-endian byte order.
4. Sign `m2` using ECDSA over P-384 and SHA-384 with the private key `sk`.
   We'll call this `sig`. The output of `sig` MUST be in the format `r || s`
   (where `||`means concatenate), for a total length of 96 bytes.
   * Signatures **SHOULD** use deterministic nonces ([RFC 6979](https://tools.ietf.org/html/rfc6979))
     if possible, to mitigate the risk of [k-value reuse](https://blog.trailofbits.com/2020/06/11/ecdsa-handle-with-care/).
   * If RFC 6979 is not available in your programming language, ECDSA **MUST** use a CSPRNG
     to generate the k-value.
   * Hedged signatures (RFC 6979 + additional randomness to provide resilience to fault attacks)
     are allowed.
   ```
   sig = crypto_sign_ecdsa_p384(
       message = m2,
       private_key = sk
   );
   ```
5. If `f` is:
    * Empty: return "`h` || base64url(`m` || `sig`)"
    * Non-empty: return "`h` || base64url(`m` || `sig`) || `.` || base64url(`f`)"
    * ...where || means "concatenate"
    * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

### ECDSA Public Key Point Compression

Given a public key consisting of two coordinates (X, Y):

1. Set the header to `0x02`.
2. Take the least significant bit of `Y` and add it to the header.
3. Append the X coordinate (in big-endian byte order) to the header.

In pseudocode:

```
lsb(y):
   return y[y.length - 1] & 1

pubKeyCompress(x, y):
   header = [0x02 + lsb(y)]
   return header.concat(x)
```

## Verify

Given a signed message `sm`, ECDSA public key `pk` (which **MUST** use 
[point compression](https://www.secg.org/sec1-v2.pdf) (Section 2.3.3)),
and optional footer `f` (which defaults to empty string), and an optional
implicit assertion `i` (which defaults to empty string):

1. Before verifying, first assert that the key being used is intended for use
   with `v3.public` tokens, and is the public key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
3. Verify that the message begins with `v3.public.`, otherwise throw an
   exception. This constant will be referred to as `h`.
4. Decode the payload (`sm` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
    * `s` to the rightmost 96 bytes
    * `m` to the leftmost remainder of the payload, excluding `s`
5. Pack `pk`, `h`, `m`, `f`, and `i` together (in that order) using PAE (see
   [PAE](Common.md#authentication-padding).
   We'll call this `m2`.
   * `pk` **MUST** be 49 bytes long, and the first byte **MUST** be `0x02` or `0x03`
     (depending on the sign of the Y coordinate). The remaining bytes **MUST** be
     the X coordinate, using big-endian byte order.
6. Use ECDSA to verify that the signature is valid for the message:
   ```
   valid = crypto_sign_ecdsa_p384_verify(
       signature = s,
       message = m2,
       public_key = pk
   );
   ```
7. If the signature is valid, return `m`. Otherwise, throw an exception.
