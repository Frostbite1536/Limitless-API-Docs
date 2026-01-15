# User Questions

Real questions and issues from developers using the Limitless Exchange API.

## Questions Index

| Issue | Description |
|-------|-------------|
| [Tracking LP Rewards](lp-rewards-tracking.md) | How to check if you're earning LP rewards and how much |
| [Automating Market Search by Category](market-search-automation.md) | How to fetch market data from an API for analysis (e.g., which markets have ended) instead of manually copying or using spreadsheets |
| [Finding Markets by Name or Characteristics](finding-markets-by-name.md) | How to find a market slug by name (e.g., "1hr BTC") using search, filtering, or browsing |
| [Portfolio History and Wallet Types](portfolio-history-wallet-types.md) | Does `/portfolio/history` work with smart wallets or only EOA wallets? |
| [Smart Wallet Signer Mismatch](smart-wallet-signer-mismatch.md) | "Signer does not match with correct address" error |
| [Smart Wallet Signature Type](smart-wallet-signature-type.md) | Signature errors when using smart wallet with wrong signatureType |
| [Signature Verification Failed](signature-verification-failed.md) | Order signature errors and debugging |
| [Order Not Filling](order-not-filling.md) | Order accepted but no trade executes |
| [Invalid Token ID](invalid-token-id.md) | Token ID or position not found errors |

## Quick Diagnosis

### Getting a 400 error?

| Error Message | Likely Cause | Solution |
|---------------|--------------|----------|
| "Signer does not match" | Smart wallet mismatch | [See guide](smart-wallet-signer-mismatch.md) |
| "Invalid signature" | Wrong verifyingContract | [See guide](signature-verification-failed.md) |
| "Invalid signature" (with correct verifyingContract) | Wrong signatureType for smart wallet | [See guide](smart-wallet-signature-type.md) |
| "Invalid token ID" | Wrong positionId | [See guide](invalid-token-id.md) |
| "Invalid price" | Price outside 0.01-0.99 | Check your price calculation |

### Order submitted but nothing happened?

Your GTC order is probably waiting for a match. [See guide](order-not-filling.md)

## Contributing

Have a question that's not covered? Open an issue or PR to add it!

## Related Documentation

- [FAQ](../guides/faq.md) - General questions
- [Troubleshooting](../guides/placing-orders.md#troubleshooting) - Order placement issues
- [API Endpoints](../endpoints/) - Full endpoint documentation
