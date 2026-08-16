# PoC: Intercepted Independence Activation SMS

## Challenge Summary

| Item | Value |
|---|---|
| Challenge | Intercepted Independence Activation SMS |
| Category | Crypto / Telecom |
| Files provided | `gsm_7bit_intercept.conf`, `sms_gateway_routing.log` |
| Target MSISDN | `+919876543210` |
| Cipher stack | `ROT13_THEN_VIGENERE` |
| Vigenère key | `JIOKEY` |
| Flag format | `TRIVARNA{...}` |

---

## 1. Recon

The archive `Intercepted_Independence_Activation_SMS.zip` contains two files:

- **`gsm_7bit_intercept.conf`** — a large SMSC config dump (mostly decoy `param_XXXX = value_XXX` noise across dozens of `[SMSC_POLICY_NN]` sections), ending in a real target block:

  ```ini
  # --- TARGET MSISDN INTERCEPT SPEC ---
  [MSISDN_+919876543210]
  msisdn = +919876543210
  cipher_stack = ROT13_THEN_VIGENERE
  vigenere_key = JIOKEY
  # -------------------------------------
  ```

- **`sms_gateway_routing.log`** — thousands of decoy `[DEBUG]/[INFO]/[WARN] Trace event #NNNN` lines, with one real intercept buried inside:

  ```
  2026-08-03T14:10:00.123Z [WARN] [SMSC-GW1-INTERCEPT] Intercepted suspicious SMS packet from MSISDN +919876543210
  2026-08-03T14:10:00.125Z [INFO] [SMSC-GW1-INTERCEPT] MSISDN: +919876543210 Encrypted Payload:checksum=1e6e0a04
  bgbd{jxo_ker_izp13_qjdvyamf_qvwaxpj_zypzszvap_2026}session_id=SESS-4301 status=OK payload_checksum=1e6e0a04
  2026-08-03T14:10:00.128Z [DEBUG] [SMSC-GW1-INTERCEPT] Key Alias Header: VIGENERE_KEY_ALIAS_JIOKEY
  ```

  The ciphertext of interest: **`bgbd{jxo_ker_izp13_qjdvyamf_qvwaxpj_zypzszvap_2026}`**

---

## 2. Approach

The config tells us encryption was applied as `ROT13_THEN_VIGENERE`, i.e.:

```
ciphertext = Vigenere_encrypt( ROT13(plaintext), key=JIOKEY )
```

To recover the plaintext we invert the pipeline in **reverse order**:

```
plaintext = ROT13( Vigenere_decrypt(ciphertext, key=JIOKEY) )
```

(ROT13 is self-inverse, so "undoing" it is just applying it again.)

---

## 3. PoC Script

```python
#!/usr/bin/env python3
"""
PoC: Decrypt the intercepted SMS payload from sms_gateway_routing.log

Challenge: Intercepted Independence Activation SMS
-----------------------------------------------------------------
Files provided:
  - gsm_7bit_intercept.conf   -> contains cipher config for the target MSISDN:
        [MSISDN_+919876543210]
        msisdn = +919876543210
        cipher_stack = ROT13_THEN_VIGENERE
        vigenere_key = JIOKEY

  - sms_gateway_routing.log   -> contains the intercepted encrypted payload
    buried among thousands of decoy "Trace event" log lines:
        [SMSC-GW1-INTERCEPT] MSISDN: +919876543210 Encrypted Payload: ...
        bgbd{jxo_ker_izp13_qjdvyamf_qvwaxpj_zypzszvap_2026}

Encryption direction (per config): ROT13_THEN_VIGENERE
    plaintext --ROT13--> --VIGENERE(JIOKEY)--> ciphertext

Decryption is therefore the reverse:
    ciphertext --VIGENERE_DECRYPT(JIOKEY)--> --ROT13--> plaintext
    (ROT13 is self-inverse, so "undo ROT13" == "apply ROT13" again)
"""

import re
import sys
import os

# ------------------------------------------------------------------
# Step 0: locate + extract the encrypted payload directly from the
# routing log (so the PoC is self-contained and doesn't hardcode it)
# ------------------------------------------------------------------

def extract_payload_from_log(log_path):
    """Find the intercepted payload line and pull out the ciphertext."""
    with open(log_path, "r", encoding="utf-8", errors="replace") as f:
        content = f.read()

    # The ciphertext appears on its own line right after the
    # "Encrypted Payload:" marker line, wrapped in { } like flag{...}
    match = re.search(r"[a-z]{4}\{[a-z0-9_]+\}", content)
    if not match:
        raise ValueError("Could not locate ciphertext payload in log")
    return match.group(0)


def extract_key_from_conf(conf_path, msisdn="+919876543210"):
    """Parse the cipher_stack + vigenere_key for the target MSISDN block."""
    with open(conf_path, "r", encoding="utf-8", errors="replace") as f:
        content = f.read()

    section = re.search(
        rf"\[MSISDN_{re.escape(msisdn)}\](.*?)(?:\n\[|\Z)",
        content,
        re.DOTALL,
    )
    if not section:
        raise ValueError(f"Could not find MSISDN section for {msisdn}")

    block = section.group(1)
    cipher_stack = re.search(r"cipher_stack\s*=\s*(\S+)", block).group(1)
    vigenere_key = re.search(r"vigenere_key\s*=\s*(\S+)", block).group(1)
    return cipher_stack, vigenere_key


# ------------------------------------------------------------------
# Step 1: crypto primitives
# ------------------------------------------------------------------

def rot13(text: str) -> str:
    """ROT13 substitution cipher (self-inverse)."""
    result = []
    for ch in text:
        if ch.isalpha():
            base = ord('A') if ch.isupper() else ord('a')
            result.append(chr((ord(ch) - base + 13) % 26 + base))
        else:
            result.append(ch)
    return ''.join(result)


def vigenere_decrypt(text: str, key: str) -> str:
    """Standard Vigenere decryption, non-alpha chars passed through."""
    result = []
    key = key.upper()
    ki = 0
    for ch in text:
        if ch.isalpha():
            base = ord('A') if ch.isupper() else ord('a')
            k = ord(key[ki % len(key)]) - ord('A')
            result.append(chr((ord(ch) - base - k) % 26 + base))
            ki += 1
        else:
            result.append(ch)
    return ''.join(result)


# ------------------------------------------------------------------
# Step 2: reverse the cipher_stack pipeline
# ------------------------------------------------------------------

def decrypt(ciphertext: str, cipher_stack: str, key: str) -> str:
    """
    cipher_stack describes the ENCRYPTION order, e.g.:
        ROT13_THEN_VIGENERE  =>  encrypt = vigenere(rot13(plain))
    So to decrypt we invert in reverse order:
        plain = rot13(vigenere_decrypt(cipher))
    """
    steps = cipher_stack.split("_THEN_")
    data = ciphertext
    for step in reversed(steps):
        if step == "VIGENERE":
            data = vigenere_decrypt(data, key)
        elif step == "ROT13":
            data = rot13(data)  # self-inverse
        else:
            raise ValueError(f"Unknown cipher step: {step}")
    return data


# ------------------------------------------------------------------
# Main
# ------------------------------------------------------------------

def main():
    base_dir = sys.argv[1] if len(sys.argv) > 1 else "."
    log_path = os.path.join(base_dir, "sms_gateway_routing.log")
    conf_path = os.path.join(base_dir, "gsm_7bit_intercept.conf")

    ciphertext = extract_payload_from_log(log_path)
    cipher_stack, vigenere_key = extract_key_from_conf(conf_path)

    print(f"[+] Extracted ciphertext : {ciphertext}")
    print(f"[+] cipher_stack         : {cipher_stack}")
    print(f"[+] vigenere_key         : {vigenere_key}")

    plaintext = decrypt(ciphertext, cipher_stack, vigenere_key)
    print(f"[+] Decrypted plaintext  : {plaintext}")

    # Re-wrap as event's flag format
    inner = plaintext[plaintext.index("{") + 1 : plaintext.index("}")]
    trivarna_flag = f"TRIVARNA{{{inner}}}"
    print(f"[+] Submission flag      : {trivarna_flag}")


if __name__ == "__main__":
    main()
```

---

## 4. Execution

```bash
$ python3 poc_decrypt_sms_intercept.py "Intercepted Independence Activation SMS/"
[+] Extracted ciphertext : bgbd{jxo_ker_izp13_qjdvyamf_qvwaxpj_zypzszvap_2026}
[+] cipher_stack         : ROT13_THEN_VIGENERE
[+] vigenere_key         : JIOKEY
[+] Decrypted plaintext  : flag{sms_pdu_rot13_vigenere_telecom_intercept_2026}
[+] Submission flag      : TRIVARNA{sms_pdu_rot13_vigenere_telecom_intercept_2026}
```

---

## 5. Manual Walkthrough (for verification)

| Step | Operation | Result |
|---|---|---|
| 0 | Ciphertext | `bgbd{jxo_ker_izp13_qjdvyamf_qvwaxpj_zypzszvap_2026}` |
| 1 | Vigenère decrypt (key=`JIOKEY`) | `synt{fzf_cqh_ebg13_ivtrarer_gryrpbz_vagreprcg_2026}` |
| 2 | ROT13 | `flag{sms_pdu_rot13_vigenere_telecom_intercept_2026}` |

---

## 6. Final Flag

```
TRIVARNA{sms_pdu_rot13_vigenere_telecom_intercept_2026}
```
