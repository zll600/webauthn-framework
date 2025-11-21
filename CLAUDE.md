# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **FIDO2/WebAuthn framework for PHP and Symfony**, providing passwordless authentication using security keys and biometric authenticators. It's organized as a monorepo that splits into multiple packages via git-split.

## Development Commands

All commands use **Castor** (PHP task runner) defined in `castor.php`.

### Testing
```bash
# Run all tests
castor phpunit

# Run specific test(s)
castor phpunit -- --filter=TestName

# Run specific test file
./bin/phpunit tests/library/Unit/SomeTest.php

# JavaScript tests (Stimulus)
castor js
```

### Code Quality
```bash
# Check code style (Easy Coding Standard)
castor ecs

# Fix code style issues
castor ecs_fix

# Check refactoring opportunities (Rector)
castor rector

# Apply refactorings
castor rector_fix

# Static analysis (PHPStan at max level)
castor phpstan

# Architecture layer validation (Deptrac)
castor deptrac

# PHP syntax check
castor lint
```

### Before Submitting PR
```bash
# Run all fixes and checks in sequence
castor prepare_pr
```

## Architecture

### Monorepo Structure

Three main packages in `src/`:

1. **`webauthn/`** (`web-auth/webauthn-lib`) - Core PHP library
   - Pure PHP FIDO2/WebAuthn implementation
   - No framework dependencies (besides Symfony components)
   - Handles attestation/assertion validation and cryptographic operations

2. **`symfony/`** (`web-auth/webauthn-symfony-bundle`) - Symfony integration
   - Controllers, repositories, DI configuration
   - Security bundle integration
   - Doctrine ORM credential storage

3. **`stimulus/`** (`web-auth/ux`) - Frontend Stimulus controllers
   - Browser WebAuthn API interaction

Packages are split to separate repos via `.gitsplit.yml`.

### Core Architectural Patterns

**Ceremony Step Pattern** - Validation is decomposed into discrete steps:
- Located in `src/webauthn/src/CeremonyStep/`
- Each step implements specific WebAuthn specification requirements
- Examples: `CheckChallenge`, `CheckSignature`, `CheckOrigin`, `CheckCounter`
- Orchestrated by `CeremonyStepManager`
- Steps are registered and executed in sequence during validation ceremonies

**Attestation Statement Support Pattern** - Pluggable attestation format handlers:
- Located in `src/webauthn/src/AttestationStatement/`
- Supports: Android Key, Apple, FIDO U2F, Packed, TPM, None, Compound
- Each format handler implements `AttestationStatementSupport` interface
- Managed by `AttestationStatementSupportManager`

**Repository Pattern**:
- `PublicKeyCredentialSourceRepositoryInterface` - Credential storage
- `PublicKeyCredentialUserEntityRepositoryInterface` - User entity management
- Doctrine implementation: `DoctrineCredentialSourceRepository`

**Event Dispatcher Pattern**:
- PSR-14 event dispatcher for lifecycle hooks
- Events in `src/webauthn/src/Event/`

### Key Components

**Core Validators** (entry points for validation logic):
- `AuthenticatorAttestationResponseValidator` - Validates registration responses
- `AuthenticatorAssertionResponseValidator` - Validates authentication responses

**Ceremony Management**:
- `CeremonyStepManager` in `src/webauthn/src/CeremonyStep/CeremonyStepManager.php`
- Orchestrates ceremony steps for attestation and assertion validation
- Steps can be registered/customized via dependency injection

**Credential Management**:
- `PublicKeyCredentialSource` - Represents stored credentials
- `PublicKeyCredentialDescriptor` - References credentials
- Counter validation to detect cloned authenticators

**Symfony Bundle Integration**:
- Controllers in `src/symfony/src/Controller/`
- DI configuration in `src/symfony/src/DependencyInjection/`
- Compiler passes for service registration
- Security authenticator factory integration

### Architectural Boundaries (Deptrac)

Strict layer enforcement:
- **Webauthn** (core) → Vendors + MetadataService only
- **SymfonyBundle** → Vendors + Webauthn + MetadataService
- **UX/Stimulus** → Vendors only
- **MetadataService** → Vendors only

Violations will fail CI checks via `castor deptrac`.

## Development Workflow

### Adding a New Ceremony Step

1. Create class in `src/webauthn/src/CeremonyStep/` implementing `CeremonyStep` interface
2. Register in `CeremonyStepManagerFactory` or via Symfony compiler pass
3. Add unit tests in `tests/library/Unit/CeremonyStep/`
4. Consider both attestation and assertion ceremony contexts

### Adding a New Attestation Format

1. Create support class in `src/webauthn/src/AttestationStatement/`
2. Implement `AttestationStatementSupport` interface
3. Register via Symfony bundle compiler pass
4. Add comprehensive tests with real attestation examples in `tests/library/Unit/AttestationStatement/`

### Test Structure

Tests are in `tests/`:
- `framework/` - Framework-level integration tests
- `library/` - Core library unit tests (Unit & Functional subdirs)
- `symfony/` - Symfony bundle functional tests
- `MDS/` - Metadata Service tests

## Important Context

### Requirements
- PHP 8.2+
- Extensions: ext-json, ext-openssl
- Symfony 6.4, 7.0, or 8.0

### Code Standards
- PSR-12 coding standard (enforced via ECS)
- Strict types declared in all files
- PHPStan at max level
- Git-Flow branching strategy
- Current development branch: **5.3.x**
- Main branch for PRs: **5.2.x**

### Key Dependencies
- `spomky-labs/cbor-php` - CBOR encoding/decoding
- `web-auth/cose-lib` - COSE cryptography
- `spomky-labs/pki-framework` - PKI operations

### Monorepo Development
- Use `./link` script to link local packages for development
- Changes in `src/*` affect corresponding split repositories
- CI runs git-split automatically on push

### Security
- Private vulnerability reporting via GitHub Security Advisories
- Contact: security@spomky-labs.com
- **Never** file public issues for security vulnerabilities
