## Fix: declare `pycryptodome` in manifest requirements

`openpulsar.py` does `from Crypto.Cipher import AES` at load time, but the manifest
declared `"requirements": []`. On any install that doesn't already have pycryptodome,
that's a `ModuleNotFoundError`. Added `pycryptodome>=3.20.0` so HA installs it.

**Risk:** tiny — just a dependency declaration.
