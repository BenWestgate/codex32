# For Wallet Developers

Codex32 is a new format for BIP32 master seeds. It is essentially a replacement for
BIP39 or SLIP39 seed words, and the user workflow should be much the same, except
that:

* The character set is different (bech32 characters rather than a wordlist).
* Seeds may be split across multiple shares, rather than encoded as a single string.
* It is possible to do specific error detection/correction during entry.

There are two levels of wallet support:

* The ability to import seeds/shares; here we essentially just have recommendations about dealing with errors.
* The ability to generate seeds/shares on the device; here our guidance is more involved.

We encourage every wallet to support importing seeds and shares, since
* the technical difficulty is low (roughly on par with that of supporting Segwit addresses, plus optional error-correction support);
* the added functionality is isolated from the rest of the wallet (once the seed is imported you don't care where it came from); and
* beyond correctness of the code, there is little risk (no need to source randomness or execute potentially variable-time algorithms).

Supporting seed generation is a little more involved so the tradeoff between
implementation complexity and user value is less clear, especially since the
Codex provides users instructions on doing generation themselves.

## Error Detection and Correction

Wallets MUST support detection of errors using the codex32 checksum algorithm.
Wallets SHOULD additionally support error correction; such wallets are referred to as "error-correcting wallets (ECWs)" and have additional requirements.

An ECW:

* MUST support correction of up to 4 substitution errors and erasures;
* MAY also support correction of up to 8 erasures, up to 13 if contiguous;
* MAY support correction of further errors, including insertion or deletion errors.

If a wallet is unable to meet these specifications, it is not an ECW and it SHOULD NOT expose error-correction functionality to the user.

## Import Support

Wallets SHOULD support import of 128- and 256-bit seeds; 512-bit seeds are optional; other lengths are NOT RECOMMENDED. 128-bit seeds encode as 48-character codex32 strings, including the `MS1` prefix. 256-bit seeds encode as 74-character codex32 strings. For other bit-lengths, see the BIP.

The process for entering codex32 strings is:

1. The user should enter the first string. To the extent possible given screen limitations, data should be displayed in uppercase with visually distinct four-character windows. The first four-character window should include the `MS1` prefix, which SHOULD be pre-filled.
   * The user MUST be able to enter all bech32 characters.
   * ECWs MUST also allow entry of `?` which indicates an erasure (unknown character).
   * The user SHOULD NOT be able to enter mixed-case characters.
   * If the header is invalid, the wallet SHOULD highlight the problem and if not an ECW, request confirmation from the user before allowing additional data to be entered.
     * An invalid header is one that starts with a character other than `0` or `2` through `9`, or one which starts with `0` but whose share index is not `S`. For shares after the first, a header is also invalid if its threshold and identifier do not match those of the first share or whose share index matches any previous share.
     * ECWs SHOULD replace the offending characters of the header with `?`.
   * Wallets MAY:
     * Allow users to enter invalid characters, at their discretion. (This may be useful to guide error correction, by attempting to replace commonly confused characters.)
     * Use predictive text for on-screen keyboards to suggest the codex32 checksum characters but if so MUST require user to manually accept the prediction.
     * Indicate when the entry has a valid checksum, e.g. by highlighting the string green or displaying the "Submit" option but they MUST NOT submit a string with a valid checksum without user request.
   * ECWs MAY additionally indicate when an entry of sufficient length to correct has an invalid checksum, e.g. by highlighting the string red or displaying an "Attempt Correction" option.
1. Once the first string is fully entered, the wallet MUST validate the checksum and header before accepting it.
   * If the checksum does not pass, then an ECW:
      * MUST attempt error correction of substitution errors and erasures.
      * MAY attempt correction by deleting and/or inserting characters, as long as the resulting string has a valid length for a codex32 string. ECWs MAY assume the correct length is the closest of 48 or 74.
      * MUST show any valid correction candidate found to the user for confirmation rather than silently applying it.
         * If insertion and/or deletion correction candidates are found, the shortest edit distance valid string SHOULD be displayed.
           * This is the sum of all edits with erasures and deletes weighted 1 and substitutions and insertions weighted 2.
         * ECWs displaying a candidate correction MAY highlight corrected 4-character windows and/or specific correction locations.
1. After the first string has been entered and accepted, the wallet now knows the identifier, threshold value and valid length.
   * If the string has index `S`, this is the codex32 secret and the import process is complete.
   * Otherwise, the fourth character of the share will be a numeric character between `2` and `9` inclusive. The user must enter this many shares in total.
   * Wallets MAY encrypt and store recovery progress, to allow recovery without having all shares available at once. The details of this are currently outside of the scope of this specification.
1. The user should then enter the remaining shares, in the same manner as the first.
   * The wallet SHOULD pre-fill the header (threshold value and identifier).
   * If the user tries to repeat an already-entered share index, they SHOULD be prevented from entering additional data until the user confirms they are not repeating a share.
     * non-ECWs SHOULD highlight the error and request the user to correct it before allowing additional data to be entered.
     * ECWs SHOULD replace the offending share index with `?`.
   * If the checksum fails, the wallet MAY attempt correction by deleting and/or inserting characters. However, the wallet MUST assume the valid length of all subsequent shares is equal to the valid length of the first share, so the number of characters inserted and deleted must net out to the correct length.
1. For all invalid codex32 strings entered, if an ECW is able to correct the errors (by deletion, insertion, substitution and/or filling erasures), it MUST show the corrected string to the user and request confirmation that the corrected string **exactly matches** the user's copy of the data. It MUST NOT silently apply corrections without approval from the user.
    * If no valid string is found with a correct hrp, header and unique index within correction distance limits or within 10 seconds of search, give up.
    * ECWs MAY warn the user they've repeated a share if the only valid string found exactly matches a previously entered share.
1. Once all shares are entered, the wallet should recover the master seed and import this.

**The master seed should be used directly as a master seed, as specified in BIP32.**
Unlike in BIP39 or other specifications, no PBKDF or other pre-processing should be applied.

## Generate Support

There are two levels of generation support:

* Exporting a seed as an unshared codex32 secret (threshold `0`, index `S`).
* Generating share sets. Everything said about secrets also applies to shares.

Export is worthwhile without sharing: the checksum corrects errors, can be checked by hand, and needs no wordlist or PBKDF2.

### Entropy

* Random shares and random identifiers MUST come from the same source as fresh seeds, including any user entropy the wallet mixes in.
  Predictable shares let someone holding fewer than `k` shares recover the secret. Random shares MUST NOT be derived from the seed.
* If user entropy is the only source, or the wallet counts on it in case its RNG is weak, the wallet MUST collect at least as many bits as the seed length it claims: 50 d6 rolls for 128 bits, 100 for 256.
* A wallet MUST NOT display intermediate values that determine the seed, such as a running hash of dice rolls.
* A seed that is not a valid BIP32 master key MUST be discarded and drawn again.

### Parameters

* **Length:** 128-bit SHOULD be the default; 256-bit MAY be offered. 512-bit strings (127 characters) are NOT RECOMMENDED for hand backup.
  A wallet with a 512-bit BIP39 seed SHOULD offer a fresh 128-bit seed and a sweep; the pending [BIP-85 codex32 application](https://github.com/bitcoin/bips/pull/1958) can derive one.
* **Human-readable part:** only `ms`, and not a user option.
  Wallets MUST NOT emit unregistered prefixes: other BIP-93 wallets cannot recover them.
* **Threshold:** `0` for a secret. Share sets SHOULD offer `k` = 2 and 3, with 2 as the default; higher values only for power users.
* **Identifier:** public metadata, not authentication.
  * Wallets MUST NOT fix 3 or more characters (for example `SEED` for every seed). They MAY fix 1 or 2.
  * Wallets MUST NOT derive it from a hash of the seed, a private key or other private data. Hashes are far faster to test than key derivation.
  * A secret SHOULD default to the first 20 bits of its BIP32 fingerprint, so descriptors and PSBTs point to the right card.
  * A share set MAY use a random identifier, or a unique one the user chooses. A fingerprint on shares gives a holder of `k` - 1 shares an offline check for guesses at the rest, and links every card to its wallet.
  * Re-sharing MUST NOT reuse the threshold and identifier of the set being re-shared, so shares from the two sets cannot be mixed.
  * Wallets SHOULD remember identifiers they have issued and warn before reusing one.
* **Indices:** wallets MAY pick share indices at random so the cards do not reveal how many exist.
* **Padding:** random shares MUST have uniformly random padding. The padding of `S` is arbitrary; wallets MAY set it to a CRC of the seed bits, with generator `(1 << p) | 3` and initial value 0, where `p` is the number of padding bits.
  Importers MUST accept any padding. For a fresh share set, the last random share is redrawn until `S` is a valid master key with the chosen padding.

### Backup process

For a secret:

1. Encode the seed with threshold `0`, the identifier and index `S`.
1. Display it and confirm it as below.
1. Only then import the seed. If the user skips confirmation, the wallet MUST NOT import a fresh seed; the user restores from the backup instead.

For a share set:

1. For a fresh seed, draw `k` random shares. The user never sees `S`.
   For an existing seed, use `S` and draw `k` - 1 random shares.
1. Display and confirm each string before drawing the next.
1. Derive the remaining `n` - `k` shares by interpolation; display and confirm each one.
1. After all `n` strings are confirmed, import the fresh seed or return to the wallet that was backed up.

Every string MUST be confirmed, including derived shares.
A share-set flow SHOULD NOT make the user see, write down or confirm `S`.

### Display

* Strings SHOULD be shown in uppercase, as BIP-93 recommends for handwriting and QR codes, in a monospaced font, split into 4-character windows.
* Windows SHOULD be separated by a character that cannot be typed, such as U+2004 (three-per-em space). Confirmation can then reject entries that contain it.
* Wallets SHOULD provide a template with numbered boxes; the numbers are not copied. Copying and screenshots SHOULD be restricted where possible.
* A wallet that shows a stored seed again SHOULD reproduce the user's original strings. If it cannot, it MUST say the new string differs from their backup.

### Confirmation

The user re-enters each string from their copy.

* Entry MUST exactly match. There is no error correction during confirmation.
* The header MUST NOT be pre-filled. Pasted text SHOULD be refused.
* Wallets SHOULD highlight each window that contains an error, but only once it is complete and never at finer resolution.
  Correct windows SHOULD be locked.
* If the user's copy is wrong, the wallet SHOULD let them show the string again to fix it, then confirm the whole string again.

Desktop wallets SHOULD use full verification: the user types every string.

Wallets with limited input MAY use partial verification:

1. Split each string into groups of three windows.
1. For each group, hide one window at random and ask for it.
   Offer the correct window and decoys taken from other windows of the set, at least 6 options for 128-bit seeds and 8 for 256-bit.
1. A wrong answer is reported, and that group MUST later be answered right on the first try with new options. Viewing the string again restarts the test.

Wallets that compute a checksum for a hand-made payload MUST refuse input that is already a complete string with a bad checksum, and SHOULD warn that the payload's randomness is the user's responsibility.
