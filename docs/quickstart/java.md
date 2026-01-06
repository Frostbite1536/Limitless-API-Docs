# Java Quickstart Guide

Complete guide to trading on Limitless Exchange using Java.

## Prerequisites

### Dependencies (build.gradle)

```groovy
plugins {
    id 'java'
    id 'application'
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.web3j:core:4.9.8'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.16.0'
    implementation 'org.slf4j:slf4j-simple:2.0.9'
}

application {
    mainClass = 'LimitlessTrader'
}
```

### Environment Variables

```bash
export PRIVATE_KEY="0x..."  # Your wallet private key
```

## Complete Trading Example

### Main Trading Class

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import okhttp3.*;
import org.web3j.crypto.*;
import org.web3j.utils.Numeric;

import java.io.IOException;
import java.math.BigInteger;
import java.nio.charset.StandardCharsets;
import java.util.*;

public class LimitlessTrader {

    private static final String API_URL = "https://api.limitless.exchange";
    private static final int CHAIN_ID = 8453;  // Base mainnet

    private static final OkHttpClient client = new OkHttpClient();
    private static final ObjectMapper mapper = new ObjectMapper();

    // Cache for market data (venue is static per market)
    private static final Map<String, JsonNode> marketCache = new HashMap<>();

    // ========== Authentication ==========

    public static String getSigningMessage() throws IOException {
        Request request = new Request.Builder()
            .url(API_URL + "/auth/signing-message")
            .get()
            .build();

        try (Response response = client.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException("Failed: " + response.code());
            }
            return response.body().string();
        }
    }

    public static AuthResult authenticate(String privateKey) throws Exception {
        Credentials credentials = Credentials.create(privateKey);
        String address = credentials.getAddress();

        // Get signing message
        String signingMessage = getSigningMessage();

        // Sign message
        byte[] messageHash = Sign.getEthereumMessageHash(
            signingMessage.getBytes(StandardCharsets.UTF_8)
        );
        Sign.SignatureData signatureData = Sign.signMessage(
            messageHash, credentials.getEcKeyPair(), false
        );

        // Format signature
        byte[] signature = new byte[65];
        System.arraycopy(signatureData.getR(), 0, signature, 0, 32);
        System.arraycopy(signatureData.getS(), 0, signature, 32, 32);
        signature[64] = signatureData.getV()[0];
        String signatureHex = "0x" + Numeric.toHexStringNoPrefix(signature);

        // Hex encode signing message
        String messageHex = "0x" + Numeric.toHexStringNoPrefix(
            signingMessage.getBytes(StandardCharsets.UTF_8)
        );

        // Login request
        ObjectNode body = mapper.createObjectNode();
        body.put("client", "eoa");

        Request request = new Request.Builder()
            .url(API_URL + "/auth/login")
            .post(RequestBody.create(
                mapper.writeValueAsString(body),
                MediaType.parse("application/json")
            ))
            .header("x-account", address)
            .header("x-signing-message", messageHex)
            .header("x-signature", signatureHex)
            .build();

        try (Response response = client.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException("Auth failed: " + response.code());
            }

            // Extract session cookie
            String sessionCookie = null;
            for (Cookie cookie : Cookie.parseAll(
                request.url(), response.headers()
            )) {
                if (cookie.name().equals("limitless_session")) {
                    sessionCookie = cookie.value();
                    break;
                }
            }

            JsonNode userData = mapper.readTree(response.body().string());
            return new AuthResult(sessionCookie, userData);
        }
    }

    // ========== Market Data ==========

    public static JsonNode getMarket(String slug) throws IOException {
        // Check cache first (venue data is static per market)
        if (marketCache.containsKey(slug)) {
            return marketCache.get(slug);
        }

        Request request = new Request.Builder()
            .url(API_URL + "/markets/" + slug)
            .get()
            .build();

        try (Response response = client.newCall(request).execute()) {
            JsonNode market = mapper.readTree(response.body().string());
            marketCache.put(slug, market);  // Cache for future use
            return market;
        }
    }

    public static String getVenueExchange(String slug) throws IOException {
        JsonNode market = getMarket(slug);
        return market.get("venue").get("exchange").asText();
    }

    public static JsonNode getOrderbook(String slug) throws IOException {
        Request request = new Request.Builder()
            .url(API_URL + "/markets/" + slug + "/orderbook")
            .get()
            .build();

        try (Response response = client.newCall(request).execute()) {
            return mapper.readTree(response.body().string());
        }
    }

    // ========== EIP-712 Signing ==========

    /**
     * Sign order using EIP-712 with venue's exchange address.
     * IMPORTANT: venueExchange must be fetched from market data via getVenueExchange(slug)
     */
    public static String signOrder(String privateKey, Map<String, Object> order, String venueExchange)
        throws Exception {

        Credentials credentials = Credentials.create(privateKey);

        // Build EIP-712 typed data hash
        byte[] domainSeparator = buildDomainSeparator(venueExchange);
        byte[] structHash = buildOrderStructHash(order);

        // Final hash: keccak256("\x19\x01" + domainSeparator + structHash)
        byte[] encoded = new byte[66];
        encoded[0] = 0x19;
        encoded[1] = 0x01;
        System.arraycopy(domainSeparator, 0, encoded, 2, 32);
        System.arraycopy(structHash, 0, encoded, 34, 32);

        byte[] hash = Hash.sha3(encoded);

        // Sign
        Sign.SignatureData sig = Sign.signMessage(
            hash, credentials.getEcKeyPair(), false
        );

        byte[] signature = new byte[65];
        System.arraycopy(sig.getR(), 0, signature, 0, 32);
        System.arraycopy(sig.getS(), 0, signature, 32, 32);
        signature[64] = sig.getV()[0];

        return "0x" + Numeric.toHexStringNoPrefix(signature);
    }

    /**
     * Build EIP-712 domain separator using venue's exchange address.
     * @param venueExchange The exchange address from market.venue.exchange
     */
    private static byte[] buildDomainSeparator(String venueExchange) {
        // EIP-712 domain separator
        byte[] typeHash = Hash.sha3(
            "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
                .getBytes(StandardCharsets.UTF_8)
        );

        byte[] nameHash = Hash.sha3("Limitless CTF Exchange".getBytes());
        byte[] versionHash = Hash.sha3("1".getBytes());

        // Encode: typeHash + nameHash + versionHash + chainId + contract
        byte[] encoded = new byte[160];
        System.arraycopy(typeHash, 0, encoded, 0, 32);
        System.arraycopy(nameHash, 0, encoded, 32, 32);
        System.arraycopy(versionHash, 0, encoded, 64, 32);

        byte[] chainId = Numeric.toBytesPadded(BigInteger.valueOf(CHAIN_ID), 32);
        System.arraycopy(chainId, 0, encoded, 96, 32);

        // Use venue exchange address (from market data)
        byte[] contract = Numeric.hexStringToByteArray(
            Numeric.cleanHexPrefix(venueExchange)
        );
        byte[] contractPadded = new byte[32];
        System.arraycopy(contract, 0, contractPadded, 12, 20);
        System.arraycopy(contractPadded, 0, encoded, 128, 32);

        return Hash.sha3(encoded);
    }

    private static byte[] buildOrderStructHash(Map<String, Object> order) {
        // Order type hash
        String orderType = "Order(uint256 salt,address maker,address signer," +
            "address taker,uint256 tokenId,uint256 makerAmount," +
            "uint256 takerAmount,uint256 expiration,uint256 nonce," +
            "uint256 feeRateBps,uint8 side,uint8 signatureType)";
        byte[] typeHash = Hash.sha3(orderType.getBytes(StandardCharsets.UTF_8));

        // Encode order fields (simplified - implement full encoding)
        // This is a simplified version - full implementation requires
        // proper ABI encoding of all fields

        return typeHash;  // Placeholder
    }

    // ========== Order Management ==========

    public static JsonNode createOrder(
        String sessionCookie,
        Map<String, Object> order,
        int ownerId,
        String marketSlug,
        String orderType
    ) throws IOException {

        ObjectNode payload = mapper.createObjectNode();
        payload.set("order", mapper.valueToTree(order));
        payload.put("ownerId", ownerId);
        payload.put("orderType", orderType);
        payload.put("marketSlug", marketSlug);

        Request request = new Request.Builder()
            .url(API_URL + "/orders")
            .post(RequestBody.create(
                mapper.writeValueAsString(payload),
                MediaType.parse("application/json")
            ))
            .header("Cookie", "limitless_session=" + sessionCookie)
            .build();

        try (Response response = client.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException("Order failed: " + response.code() +
                    " " + response.body().string());
            }
            return mapper.readTree(response.body().string());
        }
    }

    public static JsonNode cancelOrder(String sessionCookie, String orderId)
        throws IOException {

        Request request = new Request.Builder()
            .url(API_URL + "/orders/" + orderId)
            .delete()
            .header("Cookie", "limitless_session=" + sessionCookie)
            .build();

        try (Response response = client.newCall(request).execute()) {
            return mapper.readTree(response.body().string());
        }
    }

    // ========== Portfolio ==========

    public static JsonNode getPositions(String sessionCookie) throws IOException {
        Request request = new Request.Builder()
            .url(API_URL + "/portfolio/positions")
            .get()
            .header("Cookie", "limitless_session=" + sessionCookie)
            .build();

        try (Response response = client.newCall(request).execute()) {
            return mapper.readTree(response.body().string());
        }
    }

    // ========== Main Trading Function ==========

    public static JsonNode executeTrade(
        String marketSlug,
        String tokenType,
        int priceCents,
        int amount
    ) throws Exception {

        String privateKey = System.getenv("PRIVATE_KEY");
        if (privateKey == null) {
            throw new IllegalStateException("PRIVATE_KEY not set");
        }

        // 1. Authenticate
        System.out.println("Authenticating...");
        AuthResult auth = authenticate(privateKey);
        String makerAddress = auth.userData.get("account").asText();
        int ownerId = auth.userData.get("id").asInt();

        // 2. Get market data (cached per market)
        System.out.println("Fetching market: " + marketSlug);
        JsonNode market = getMarket(marketSlug);

        // Get venue exchange for EIP-712 signing (CRITICAL)
        String venueExchange = market.get("venue").get("exchange").asText();
        System.out.println("Using venue exchange: " + venueExchange);

        // Get token ID (positionIds[0] = YES, positionIds[1] = NO)
        JsonNode positionIds = market.get("positionIds");
        String tokenId = tokenType.equals("YES")
            ? positionIds.get(0).asText()
            : positionIds.get(1).asText();

        // 3. Calculate amounts
        double priceInDollars = priceCents / 100.0;
        double totalCost = priceInDollars * amount;
        long scalingFactor = 1_000_000L;

        long makerAmount = Math.round(totalCost * scalingFactor);
        long takerAmount = Math.round(amount * scalingFactor);

        System.out.printf("Buying %d %s at $%.2f%n", amount, tokenType, priceInDollars);
        System.out.printf("Total cost: $%.2f%n", totalCost);

        // 4. Create order payload
        Map<String, Object> order = new LinkedHashMap<>();
        order.put("salt", System.currentTimeMillis());
        order.put("maker", makerAddress);
        order.put("signer", makerAddress);
        order.put("taker", "0x0000000000000000000000000000000000000000");
        order.put("tokenId", tokenId);
        order.put("makerAmount", makerAmount);
        order.put("takerAmount", takerAmount);
        order.put("expiration", "0");
        order.put("nonce", 0);
        order.put("feeRateBps", 0);
        order.put("side", 0);  // BUY
        order.put("signatureType", 0);

        // 5. Sign order using venue's exchange address
        String signature = signOrder(privateKey, order, venueExchange);
        order.put("signature", signature);
        order.put("price", priceInDollars);

        // 6. Submit order
        System.out.println("Submitting order...");
        return createOrder(auth.sessionCookie, order, ownerId, marketSlug, "GTC");
    }

    // ========== Helper Classes ==========

    public static class AuthResult {
        public final String sessionCookie;
        public final JsonNode userData;

        public AuthResult(String sessionCookie, JsonNode userData) {
            this.sessionCookie = sessionCookie;
            this.userData = userData;
        }
    }

    // ========== Main ==========

    public static void main(String[] args) {
        System.out.println("=".repeat(80));
        System.out.println("LIMITLESS EXCHANGE TRADING - JAVA");
        System.out.println("=".repeat(80));

        try {
            JsonNode result = executeTrade(
                "btc-100k-2024",  // Market slug
                "YES",            // Token type
                65,               // Price in cents
                100               // Amount
            );

            System.out.println("\nOrder created successfully!");
            System.out.println(mapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(result));

        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

## Build and Run

```bash
# Build
./gradlew build

# Run
PRIVATE_KEY="0x..." ./gradlew run

# Create fat JAR
./gradlew fatJar
java -jar build/libs/limitless-trading-all.jar
```

## Additional build.gradle Configuration

```groovy
jar {
    manifest {
        attributes 'Main-Class': 'LimitlessTrader'
    }
}

task fatJar(type: Jar) {
    manifest {
        attributes 'Main-Class': 'LimitlessTrader'
    }
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
    from { configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) } }
    with jar
}
```

## Common Operations

### Get Best Prices

```java
public static Map<String, Double> getBestPrices(String slug) throws IOException {
    JsonNode orderbook = getOrderbook(slug);

    Double bestBid = null;
    Double bestAsk = null;

    JsonNode bids = orderbook.get("bids");
    if (bids != null && bids.size() > 0) {
        bestBid = bids.get(0).get("price").asDouble();
    }

    JsonNode asks = orderbook.get("asks");
    if (asks != null && asks.size() > 0) {
        bestAsk = asks.get(0).get("price").asDouble();
    }

    Map<String, Double> prices = new HashMap<>();
    prices.put("bid", bestBid);
    prices.put("ask", bestAsk);
    if (bestBid != null && bestAsk != null) {
        prices.put("spread", bestAsk - bestBid);
    }

    return prices;
}
```

### Calculate Portfolio Value

```java
public static double calculatePortfolioValue(String sessionCookie)
    throws IOException {

    JsonNode positions = getPositions(sessionCookie);
    long totalValue = 0;

    JsonNode clob = positions.get("clob");
    if (clob != null) {
        for (JsonNode pos : clob) {
            JsonNode posData = pos.get("positions");
            totalValue += posData.get("yes").get("marketValue").asLong();
            totalValue += posData.get("no").get("marketValue").asLong();
        }
    }

    return totalValue / 1_000_000.0;  // Convert to dollars
}
```

## Disclaimer

This code is for educational purposes only. Limitless Labs is not responsible for any losses. Always test with small amounts first.
