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
   * If the header is invalid, a the wallet SHOULD highlight the problem and if not an ECW, request confirmation from the user before allowing additional data to be entered.
     * An invalid header is one that starts with a character other than `0` or `2` through `9`, or one which starts with `0` but whose share index is not `S`.
     * ECWs SHOULD replace the offending characters of the header with `?`.
   * Wallets MAY:
     * Allow users to enter invalid characters, at their discretion.
     * Use predictive text for on-screen keyboards to suggest the codex32 checksum characters but if so MUST require user to manually accept the prediction.
     * Indicate when an entry has a valid checksum, e.g. by highlighting the string green or displaying the "Submit" option but they MUST NOT submit a string with a valid checksum without user request.
   * ECWs MAY additionally indicate when an entry of sufficient length to correct has an invalid checksum, e.g. by highlighting the string red or displaying an "Attempt Correction" option.
1. Once the first string is fully entered, the wallet MUST validate the checksum and header before accepting it.
   * If the checksum does not pass, then an ECW:
      * MUST attempt error correction of substitution errors and erasures.
      * MAY attempt correction by deleting and/or inserting characters, as long as the resulting string has a valid length for a codex32 string. ECWs MAY assume the correct length is the closest of 48, 74, or 118.
      * MUST show any valid correction candidate found to the user for confirmation rather than silently applying it.
         * If insertion and/or deletion correction candidates are found, the shortest edit distance valid string SHOULD be displayed.
           * This is the sum of all edits with erasures and deletes weighted 1 and substitutions and insertions weighted 2.
         * ECWs displaying a candidate correction MAY highlight corrected 4-character windows and/or specific correction locations.
1. After the first string has been entered and accepted, the wallet now knows the identifier, threshold value and valid length.
    * If the first string has index `S`, this is the codex32 secret and the import process is complete.
    * Otherwise, the fourth character of the share will be a numeric character between `2` and `9` inclusive. The user must enter this many shares in total.
    * Wallets MAY encrypt and store recovery progress, to allow recovery without having all shares available at once. The details of this are currently outside of the scope of this specification.
1. The user should then enter the remaining shares, in the same manner as the first.
   * The wallet SHOULD pre-fill the header (threshold value and identifier).
     * For shares after the first, the header is invalid if its threshold and identifier do not match those of the first share or whose share index matches any previous share.
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

There are two levels of generation support: Unshared secrets (US wallets) and SSS share sets (SSS wallets).

Even wallets that don't need SSS features should support seed export as codex32 secrets.

This has several advantages over BIP-39:
- Checksum corrects up to 4 errors and 13 erasures during import
- Hand-verifiable checksums isolate seed integrity checks from hardware exposure risks
- Fixed-length compact encodings are faster to write and confirm
- No wordlist or PBKDF2 dependencies
- Forwards-compatible with Codex32 SSS and SLIP-39

### Codex32 secret backup process:

1. User selects OPTIONAL parameters (`identifier`, `bitlength`).
1. Entropy or an existing master seed is extracted and encoded into a codex32 secret.
1. Secret displays for user to note down on paper or metal.
1. Confirm backup by asking user to re-enter from their transcription.
   * Note: This has different rules than during import/recovery.
1. Once the codex32 secret is confirmed, the wallet should recover the master seed:
   * automatically from memory and import this.
   * manually as per "Import Support" requesting the freshly written codex32 secret.
1. After wallet import, the generation process is complete.

**The master seed should be used directly as a master seed, as specified in BIP32.**
Unlike in BIP39 or other specifications, no PBKDF or other pre-processing should be applied.

### Codex32 share set backup process:

1. User selects OPTIONAL parameters (`k`, `identifier`, `n`, `bitlength`, `indices`...).
1. For the first codex32 string:
    1. Entropy or an existing master seed is extracted and encoded into a codex32 string.
        * An existing master seed MUST use secret index 'S'.
        * Other entropy SHOULD use share index 'A'.
    1. Display and confirm as above.
1. For the next `k - 1` codex32 shares:
    1. Entropy is extracted and encoded in a codex32 share with the next share index.
    1. Display and confirm as above.
1. For the next `n - k` codex32 strings:
    1. the initial `k` codex32 strings are interpolated to the next share index.
    1. Display and confirm as above.
1. Once all `n` shares are confirmed, the generation process is complete.
1. The wallet should then either:
  * for a fresh master seed, recover it, either:
    * automatically from memory and import this.
    * manually as per "Import Support" requesting `k` unique freshly written codex32 shares.
  * an existing master seed, return to the wallet that was just backed up.

### Display and confirmation

#### Display guidance

1. Wallets SHOULD split the uppercase codex32 string into 4-character windows.
1. Insert a three-per-em space between windows and two between every 6 window group.
1. Instruct users to write down their codex32 string on one line.
1. To the extent possible given screen limitations, display the formatted string.
    - Monospaced font SHOULD be used for legibility, even if it is not elsewhere.
    - Capability to copy or screenshot the string or screenshot SHOULD be restricted if possible.
    - If the wallet must break the string in pieces, it SHOULD number the views.
      - The user SHOULD NOT write these numbers so a template SHOULD be provided.
1. When the user indicates transcription is complete the wallet, confirmation SHOULD start.

#### Confirmation guidance

During confirmation, wallets:

- MUST NOT provide error correction.
  - All final entry MUST exactly match the string that was displayed.
- SHOULD only display characters in the bech32 character set.
  - MAY attempt to silently replace invalid characters with the bech32 character commonly confused with their keystroke.
    - e.g. B with 8 or O with 0 and I with l.
- SHOULD highlight red completed 4-character windows that contain an error.
  - Preferred to highlighting incorrect windows so everything after an insert or delete does not turn red.
- Error detection MUST NOT highlight error locations at finer resolution than 4-character windows.
- MUST NOT flag/highlight until a window is complete, at the earliest.
  - MAY delay, slow fade, or post-pone this until full entry to avoid distracting the user.
  - Unless it is the last window in a string not divisible by 4. Then it is considered complete when its length is correct.
- SHOULD lock correctly entered, unhighlighted boxes from further edits and turn their font color dark green.
  - This feels satisfying and looks like a "progress bar", practically it prevents entry of new errors after confirmation.
- MAY highlight windows adjacent to one containing an error, useful for some deletion patterns or to make manual insert/substitution brute force harder.
- MAY highlight incorrect strings or incorrect 4-character windows.

There are two verification types:
* Full verification which Desktop Wallets SHOULD use.
* Partial verification which other wallets MAY use, particularly those with limited-input.

These have different assurances of string fidelity, vulnerability to keyloggers, shoulder surfing, user fatigue, user skipping due to time costs, ease of entry, and speed.

#### Partial Verification

If supported, SHOULD support 128-bit and/or 256-bit seeds, MAY support it for 512-bit seeds.

Steps:
1. Split the codex32 string into 4-character windows.
1. Combine consecutive windows into groups of (up to) three windows each.
1. For each group, display it and have the user confirm one window at random.
  - e.g. `MS10 TEST ____`, `___ TEST STUV`, or `MS10 ____ STUV`.
  - Pick some other undisplayed windows from the string(s) at random as options.
    - Minimum options REQUIRED: 6 for 128-bit entropy, 8 for 256-bit entropy, 10 for 512-bit entropy ect.
      - "entropy" is the total bits extracted from the master seed or RNG earlier.
    - For every 128-bits additional entropy, minimum options increases by 2, for example k=3 128-bit seed requires minimum 9 options and k=3 256-bit seed requires 16 options.
    - At optional high `k` options exeed the windows available in the current undisplayed share and should be drawn from adjacent shares:
      - At even higher `k` typing the 4-character window with a binary tree may be more ergonomic than scrolling.
  - Add the correct window to the options, REQUIRED: shuffle combined options.
  - If user selects or types the wrong window, a wallet MUST:
    - Advise it's incorrect and to check their codex32 backup.
    - Request they try again.
    - MAY require they start over from the beginning to discourage guessing.
    - MAY require the group or window in question be revisited at the end until it is correct on the first try.
    - SHOULD exclude windows confirmed on the first try from being selected again in either optional approach.
All groups ensures wallets MUST at the very least display what is unconfirmed.

#### Full Verification

TODO:

Steps:

1. The user should type the string. To the extent possible given screen limitations, data should be displayed in uppercase with visually distinct four-character windows.
  - The `MS1` prefix SHOULD NOT be pre-filled.
  - The header SHOULD NOT be pre-filled.
  - Wallets MAY highlight windows, groups of windows or strings red that contain at least one error, but
    - This includes substitutions, insertions, deletions, transpositions, etc
  - They MUST NOT highlight specific error characters or locations.
1. If any errors arise, pressing tab should jump to the next error window, and shift tab to the last. Space bar may also be bound to this as the strings do not contain spaces that need to be typed, they are cosmetic only. 
1. When all characters are correctly entered the entire string will then be locked with dark green text.
1. The software SHOULD procede to the next step automatically without requesting submission.
  
### FIXME Selection of parameters

#### Bitlength

MUST support generating 128- or 256-bit seeds.
SHOULD support generating 128-bit seeds.
MAY support generating 256- or 512-bit seeds; other lengths are NOT RECOMMENDED.

Generating Wallets:
MUST support generation of 128- or 256- bit seeds;
SHOULD support generating 128-bit seeds.
MAY support generating 256- or 512-bit seeds; other lengths are NOT RECOMMENDED.

#### Human-readable part

MUST NOT use custom human-readable parts `hrp` without first registering them with BIP-0093
MUST NOT be exposed as option as `hrp` is fixed for by specific network application.

Regarding the human-readable part `hrp` GWs:
MUST support 'ms'
SHOULD use 2 character strings.

Wallets MUST NOT use custom human-readable parts `hrp` without first registering them with BIP-0093/TBD TODO

#### Threshold

A digit in 0, 2 through 9.

US Wallets:
MAY use the threshold digit as an extra custom identifier character provided it is a digit.
  - For an unshared secret, the threshold parameter (the first digit of the data part) is ignored.
- If the identifier unspecified the digit "0" is recommended.
- 
SSS Wallets: 
SHOULD support k=2 and k=3 to allow users to more flexibly manage risk.
MAY choose k=2 as a sensible default as it's the minimum threshold that provides SSS benefits.
MAY support k=0 for fresh master seeds (be a SSS+US wallet) and SHOULD support k=0 for exporting existing master seeds.
SHOULD NOT show `k` values higher than `3` in a simple GUI as they are a bad trade-off between usability and robustness (which
are damaged) and security (which is improved). Power users may enable them.

#### Identifier

A four-character, valid lowercase bech32 string (not 1, i, o or b) to use in the resulting BIP-93 output. If not specified, this SHOULD be generated the master seed's bip32 fingerprint encoded as bech32.

SSS wallet Note: for fresh codex32 share sets this default requires either generating the secret first to calculate the fingerprint or relabling the strings after the first `k` have been produced and the master seed payload recovered.
- TODO Preferred implementation details are TBD but lean towards relabeling, but there are advantages either way.
  - For example not extracting `k` entropy at once allows the system to accumulate more randomness between shares.

##### Default identifier

Regarding the identifer SecretGWs:
MUST NOT fix 3+ characters as this harms disambiguation of different master seeds.
SHOULD use a deterministic default Hash160 of `master_public_key` data (e.g. BIP-32 key fingerprint) so wallets MAY give useful help finding the correct codex32 strings given descriptors and PSBTs.
SHOULD NOT use hash functions (e.g. RIPEMD160, sha256d) of `master_seed`, `master_xprv` or any other private data to set the identifier.
  - "seed_id" which was RIPEMD160(master_seed) and "WIF" which appends SHA256d(x10+seed_hex+x01) have both been deprecated in Bitcoin Core for years and at any rate are less useful and less safe than the fingerprint.
  - Hashes assist an attacker when he has between `k - 1` and `k` shares and is brute forcing what's left. Hashes are much much faster than public key derivations.
MAY fix 1-2 identifier characters to store public information or cosmetically brand strings.


US Wallets ONLY:

May use non-cryptographically secure deterministic default identifiers (e.g. Parity, CRC or BCH codes) to improve overall error correction.
- US Wallets that plan to support SSS MUST remove this code from their implementation prior to beginning work to implement SSS.

SSS Wallets:

MUST NOT ever derive a default identifier deterministically directly from the seed, private key(s), hashes there of or checksums there of without a contravening public derviation.

#### Payload Padding
Wallets SHOULD use payload `payload` CRC padding with polynomnial `1<<pad_len | 3` as this improves bit error detection to any 2 consecutive bits on 128-bit strings and any 4 consecutive bits on 256-bit strings. The bech32 character set is optimized to produce 1 bit errors between the most similar characters so this may help more than it looks at first glance.

We should standardize this so that recovery tools of the future can exploit this to improve recovery speed 4-16x for 16 and 32 byte seeds.
It IS hand computable, probably easier than the full sharing and checksumming but I haven't tested it yet.
And ammendment to the book could be issued, and the presence or absence of that ammendement could inform error correction software whether to validate this CRC.
Wallets MAY use other padding or allow power users to specify it.

