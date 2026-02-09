---
name: authenticate-wallet
description: Instructions for authenticating with the payments-mcp wallet. Use when a user is not signed in, authentication fails, or when wallet operations return auth errors.
user-invocable: true
disable-model-invocation: false
---

# Authenticating with the Payments Wallet

When the wallet is not signed in (detected via `awal status` or when wallet operations fail with authentication errors), use the `awal` CLI to authenticate.

## Authentication Flow

Authentication uses a two-step email OTP process:

### Step 1: Initiate login

```bash
awal auth login <email>
```

This sends a 6-digit verification code to the email and outputs a `flowId`.

### Step 2: Verify OTP

```bash
awal auth verify <flowId> <otp>
```

Use the `flowId` from step 1 and the 6-digit code from the user's email to complete authentication. If you have the ability to access the user's email, you can read the OTP code, or you can ask your human for the code.

## Checking Authentication Status

```bash
awal status
```

Displays wallet server health and authentication status including wallet address.

## Example Session

```bash
# Check current status
awal status

# Start login (sends OTP to email)
awal auth login user@example.com
# Output: flowId: abc123...

# After user receives code, verify
awal auth verify abc123 123456

# Confirm authentication
awal status
```

## Available CLI Commands

| Command                           | Purpose                                |
| --------------------------------- | -------------------------------------- |
| `awal status`                     | Check server health and auth status    |
| `awal auth login <email>`         | Send OTP code to email, returns flowId |
| `awal auth verify <flowId> <otp>` | Complete authentication with OTP code  |
| `awal balance`                    | Get USDC wallet balance                |
| `awal address`                    | Get wallet address                     |
| `awal show`                       | Open the wallet companion window       |

## JSON Output

All commands support `--json` for machine-readable output:

```bash
awal status --json
awal auth login user@example.com --json
awal auth verify <flowId> <otp> --json
```
