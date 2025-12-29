# Olympix BugPocer

## Overview

The Olympix BugPocer action enables AI-powered vulnerability detection to be incorporated into continuous integration workflows for code repositories on GitHub. BugPocer leverages proprietary Olympix intermediate representation and symbolic execution to analyze Solidity smart contracts for security vulnerabilities, providing deep security analysis during your CI/CD pipeline.

## Features

- **AI-Powered Security Analysis:** Leverages an agentic architecture and multiple specialized detectors to find vulnerabilities in smart contracts
- **Deep Vulnerability Detection:** Detects complex issues including reentrancy, invariant violations, state machine bugs, economic exploits, and more
- **Automated Security Testing:** Runs automatically on commits to catch security issues early
- **Proof-of-Concept Generation:** Generates PoC exploits to validate findings

## Getting Started

1. Add a GitHub repository secret with your Olympix API token and set an environment variable on GitHub Workflow named `OLYMPIX_API_TOKEN` with the secret you just added
2. Add the `olympix/bugpocer-action` GitHub Action into your workflow

## Usage

Here is an example workflow which triggers on each push and runs BugPocer analysis on your Solidity contracts.

```yaml
name: BugPocer Security Analysis
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  bugpocer-analysis:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run forge install
        run: |
          forge install

      - name: BugPocer Security Analysis
        uses: olympix/bugpocer-action@main
        env:
          OLYMPIX_API_TOKEN: ${{ secrets.OLYMPIX_API_TOKEN }}
        with:
          args: -w . -p src/YourContract.sol
```

## Configuration

The action accepts arguments through the `args` input parameter:

- `-w, --workspace-path`: Path to the workspace directory (default: current directory)
- `-p, --path`: Path to specific Solidity files to analyze (can be specified multiple times)
- `-env, --include-dot-env`: Include .env file for environment variables
- `--confirm-all`: Auto-confirm all prompts (recommended for CI/CD)

## Environment Variables

- `OLYMPIX_API_TOKEN` (required): Your Olympix API token for authentication
- `OLYMPIX_GITHUB_ACCESS_TOKEN` (optional): GitHub token for enhanced integration

## How It Works

BugPocer analyzes your Solidity contracts using a multi-tier detection architecture:

1. **Context Building**: Extracts project invariants, protocols, and patterns
2. **Code Ranking**: Prioritizes high-risk functions and contracts
3. **Multi-Detector Analysis**: Runs specialized detectors in parallel for different vulnerability categories
4. **Verification**: Generates and validates proof-of-concept exploits
5. **Reporting**: Provides detailed findings with severity levels and remediation guidance

## Support Contact

If you have any questions, feedback, or need help, feel free to contact us at contact@olympix.ai
