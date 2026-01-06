# Limitless Exchange API Documentation

A comprehensive documentation resource for the [Limitless Exchange](https://limitless.exchange) prediction market API, optimized for both human developers and LLM-powered coding assistants.

## What is Limitless Exchange?

Limitless is a decentralized prediction market platform built on the Base blockchain (Chain ID: 8453). It enables trading on market outcomes using USDC collateral through:

- **CLOB Markets**: Central Limit Order Book with limit orders
- **AMM Markets**: Automated Market Maker with liquidity pools
- **NegRisk Groups**: Multi-outcome markets sharing collateral

## Repository Structure

```
Limitless-API/
├── README.md                 # You are here
├── Official_API_Docs.md      # Official documentation from Limitless team
├── api-1.json                # Official OpenAPI 3.0 spec (JSON)
├── api-1.yaml                # Official OpenAPI 3.0 spec (YAML)
└── docs/                     # AI-generated modular documentation
    ├── overview.md           # API overview and quick reference
    ├── endpoints/            # Endpoint documentation by category
    │   ├── authentication.md
    │   ├── markets.md
    │   ├── trading.md
    │   ├── orders.md
    │   └── portfolio.md
    ├── schemas/              # Data structure references
    │   ├── market.md
    │   ├── order.md
    │   └── portfolio.md
    ├── quickstart/           # Language-specific implementation guides
    │   ├── python.md
    │   ├── typescript.md
    │   └── java.md
    ├── guides/               # Task-focused tutorials
    │   ├── authentication.md
    │   ├── placing-orders.md
    │   ├── websockets.md
    │   └── faq.md
    └── user-questions/       # Real issues from developers
        ├── README.md
        ├── smart-wallet-signer-mismatch.md
        ├── signature-verification-failed.md
        ├── order-not-filling.md
        └── invalid-token-id.md
```

## Official Documentation

The following files are **official documentation** from the Limitless team and can also be found at:

**https://api.limitless.exchange/api-v1**

| File | Description |
|------|-------------|
| `Official_API_Docs.md` | Comprehensive API guide with examples |
| `api-1.json` | OpenAPI 3.0 specification (JSON format) |
| `api-1.yaml` | OpenAPI 3.0 specification (YAML format) |

**Always prefer the official documentation for the most accurate and up-to-date information.**

## AI-Generated Documentation (`docs/`)

The `docs/` folder contains **AI-generated documentation** designed specifically for LLM-powered coding assistants.

### Why AI-Optimized Docs?

| Challenge | Solution |
|-----------|----------|
| Large context windows are expensive | Modular files (~200-500 lines each) |
| LLMs need focused context | Topic-specific documents |
| Code examples vary by language | Separate quickstarts per language |
| Common questions repeat | Dedicated FAQ and guides |

### Use Cases

1. **LLM Coding Assistants**: Load only the relevant doc file for a specific task
2. **RAG Systems**: Index modular docs for retrieval-augmented generation
3. **Claude Code**: Fork this repo and let Claude Code access the docs directly
4. **Learning Resource**: Step-by-step guides for understanding the API

### Quick Links

| Task | Document |
|------|----------|
| Get started fast | [Python Quickstart](docs/quickstart/python.md) |
| Understand the API | [Overview](docs/overview.md) |
| Place an order | [Placing Orders Guide](docs/guides/placing-orders.md) |
| Real-time data | [WebSocket Guide](docs/guides/websockets.md) |
| Common questions | [FAQ](docs/guides/faq.md) |
| Troubleshooting errors | [User Questions](docs/user-questions/) |

## Using with Claude Code

This repository is designed to work seamlessly with [Claude Code](https://github.com/anthropics/claude-code). Fork this repo and Claude Code can:

1. Read the modular documentation to understand the API
2. Generate working code using the quickstart examples as templates
3. Answer questions using the FAQ and guides
4. Help debug issues using the schema references

```bash
# Clone and use with Claude Code
git clone https://github.com/Frostbite1536/Limitless-API.git
cd Limitless-API
claude  # Start Claude Code in the repo
```

## Related Resources

### Safe Vibe Coding

For a structured approach to building software with LLMs, check out:

**[Safe Vibe Coding](https://github.com/Frostbite1536/safe-vibe-coding)**

A guide describing collaborative, structured methods for building software with LLMs while maintaining:
- **Correctness** - Ensuring code works as intended
- **Stability** - Preventing regressions and breaking changes
- **Maintainability** - Keeping codebases clean long-term

The Safe Vibe Coding methodology pairs well with this repository when building trading bots or integrations with the Limitless API.

## Key API Concepts

### Venue System (Critical)

Each CLOB market has dynamic contract addresses. **Never hardcode addresses** - always fetch from market data:

```python
market = get_market("btc-100k-2024")
venue_exchange = market['venue']['exchange']  # Use for EIP-712 signing
venue_adapter = market['venue']['adapter']    # For NegRisk SELL orders
```

### Authentication Flow

1. `GET /auth/signing-message` - Get nonce
2. Sign message with wallet
3. `POST /auth/login` - Get session cookie
4. Use cookie for authenticated requests

### Token Approvals

| Order Type | Approve To |
|------------|------------|
| BUY (all CLOB) | USDC → `venue.exchange` |
| SELL (simple) | CT → `venue.exchange` |
| SELL (NegRisk) | CT → `venue.exchange` AND `venue.adapter` |

## API Base URLs

| Service | URL |
|---------|-----|
| REST API | `https://api.limitless.exchange` |
| WebSocket | `wss://ws.limitless.exchange/markets` |
| Documentation | `https://api.limitless.exchange/api-v1` |

## Contributing

Contributions welcome! If you find inaccuracies in the AI-generated docs or want to add examples in other languages:

1. Fork the repository
2. Make your changes
3. Submit a pull request

## Disclaimer

This documentation is for educational purposes only. The AI-generated docs in `docs/` are community-maintained and may contain errors. Always verify against the official documentation.

**Limitless Labs does not maintain this repo. We are not responsible for any losses from trading or use of this AI-generated documentation. Always test with small amounts first.**

## Support

- **Official Docs**: https://limitlesslabs.notion.site/
- **Email**: hey@limitless.network
- **API Spec**: https://api.limitless.exchange/api-v1

## License

This repository contains both official Limitless documentation (used with permission) and AI-generated supplementary materials. See individual files for specific licensing information.
