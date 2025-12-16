# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Mina Protocol app for Ledger hardware wallets (Nano S, Nano X, Nano S+, Stax, Flex, and Apex P). It's a C application that runs on Ledger's secure element to provide address generation, transaction signing, and message signing for the Mina blockchain.

## Build Commands

All builds require the Ledger SDK and should be run inside the Docker container:

```bash
# Enter the dev container (macOS)
docker run --rm -ti --user "$(id -u):$(id -g)" --privileged -v "$(pwd -P):/app" ghcr.io/ledgerhq/ledger-app-builder/ledger-app-dev-tools:latest

# Build for Nano S (default)
make DEBUG=1

# Build for specific device (inside container)
BOLOS_SDK=$NANOX_SDK make    # Nano X
BOLOS_SDK=$NANOSP_SDK make   # Nano S+
BOLOS_SDK=$STAX_SDK make     # Stax
BOLOS_SDK=$FLEX_SDK make     # Flex
BOLOS_SDK=$APEX_P_SDK make   # Apex P

# Load onto device (inside container)
make load

# Clean build artifacts
make clean
make docker-clean  # Clean via Docker

# Build installers for all devices
make installers
```

## Testing

### Ragger Tests (Python - official Ledger test framework)

```bash
# Install dependencies
pip install -r tests/requirements.txt

# Run tests with Speculos emulator
pytest tests/ --tb=short -v --device nanos

# Run on physical device
pytest tests/ --tb=short -v --device nanos --backend ledgercomm

# Run single test
pytest tests/test_mina.py::test_get_address -v --device nanos

# Run with Speculos directly
speculos --model nanos build/nanos/bin/app.elf
```

### Zemu Tests (TypeScript - Zondax test framework)

```bash
cd tests_zemu
yarn install
yarn test
```

## Architecture

### Source Organization (`src/`)

- **main.c** - APDU command dispatcher. Routes commands based on instruction byte (INS):
  - `0x01` GET_CONF - Returns app version
  - `0x02` GET_ADDR - Address generation
  - `0x03` SIGN_TX - Transaction signing
  - `0x04` TEST_CRYPTO - Crypto tests (debug builds only)
  - `0x05` SIGN_MSG - Message signing

- **Cryptography**:
  - `crypto.c/h` - Key derivation and Schnorr signing using secp256k1
  - `poseidon.c/h` - Poseidon hash function (Mina's native hash)
  - `curve_checks.c/h` - Elliptic curve validation
  - `random_oracle_input.c/h` - Hash input construction

- **UI** (dual implementation for BAGL and NBGL):
  - `*_bagl.c` - UI for Nano S/S+/X devices
  - `*_nbgl.c` - UI for Stax/Flex/Apex devices

- **Transaction handling**:
  - `transaction.c/h` - Transaction structure and serialization
  - `parse_tx.c/h` - Transaction parsing from APDU data

### Build Configuration

- `RELEASE_BUILD=1` - Production build (no debug features)
- `DEBUG=1` - Enable PRINTF statements
- `ON_DEVICE_UNIT_TESTS=1` - Include on-device crypto tests
- `NO_STACK_CANARY=1` - Disable stack canary (required for release)

### BIP44 Path

Mina uses derivation path `44'/12586'/account'/0/0` where 12586 is Mina's SLIP-0044 coin type.

## Command-line Wallet

A Python utility for interacting with the Ledger app:

```bash
./utils/mina_ledger_wallet.py get-address 1
./utils/mina_ledger_wallet.py send-payment 1 <from> <to> <amount>
./utils/mina_ledger_wallet.py delegate 1 <from> <delegate>
./utils/mina_ledger_wallet.py sign-message 1 "message"
```
