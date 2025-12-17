# Bug Report

## Description
The `CustomOperations` class in the Hyperliquid SDK has a critical bug where vault address is not properly extracted and used when trading on behalf of a vault. This causes `marketClose()` and `closeAllPositions()` methods to query the user's wallet address for positions instead of the vault address, resulting in incorrect trading behavior.

## Package Information
- Package version: "1.7.7"
- Installation method: npm

## Environment
- Node.js version: "16.0.0+" (supports all versions)
- Browser: All modern browsers supported

## Steps To Reproduce the bug
1. Initialize Hyperliquid SDK with a vault address:
```javascript
const sdk = new Hyperliquid({
  privateKey: 'your-private-key',
  vaultAddress: '0xYourVaultAddress123',
  testnet: false
});
```

2. Try to close a position using CustomOperations:
```javascript
await sdk.custom.marketClose('BTC-PERP');
```

3. The method fails or behaves incorrectly because it queries the user's wallet positions instead of the vault's positions.

## Expected Behavior
- When a vault address is configured, `marketClose()` and `closeAllPositions()` should query positions from the vault address
- The SDK should properly propagate the vault address to all components including CustomOperations
- Position-based operations should work correctly for vault trading

## Actual Behavior
- `marketClose()` and `closeAllPositions()` always query the user's wallet address for positions
- Vault address is not properly extracted in the `CustomOperations` constructor
- The constructor tries to get vault address using `exchangeOrParent.isAuthenticated().toString()` which returns a boolean converted to string, not an address
- Custom operations fail when trying to close vault positions because no positions are found in the user's wallet

## Code Sample
**Minimal code example that reproduces the issue:**
```javascript
const { Hyperliquid } = require('hyperliquid');

async function demonstrateVaultBug() {
  // Initialize with vault address
  const sdk = new Hyperliquid({
    privateKey: process.env.PRIVATE_KEY,
    vaultAddress: '0x1234567890123456789012345678901234567890',
    testnet: false
  });

  await sdk.connect();

  // This will fail because it queries user wallet instead of vault
  try {
    await sdk.custom.closeAllPositions();
    console.log('Should work but may fail or behave incorrectly');
  } catch (error) {
    console.log('Error:', error.message);
  }
}
```

## Error Messages
```
Error: No position found for BTC-PERP
```
Or the method may complete without error but not close the intended vault positions.

## Root Cause Analysis
1. **Incorrect Vault Address Extraction**: 
   - Line in `src/rest/custom.ts`: `this.walletAddress = exchangeOrParent.isAuthenticated() ? exchangeOrParent.isAuthenticated().toString() : null;`
   - `isAuthenticated()` returns a boolean, not an address

2. **Missing Vault Address Property**: 
   - `CustomOperations` class has no `vaultAddress` property
   - No mechanism to access vault address from parent instance

3. **Wrong Position Query Logic**:
   - `marketClose()` and `closeAllPositions()` always use `this.getUserAddress()` 
   - Should use vault address when available

## Possible Solution
The fix involves three main changes:

1. **Fix Constructor Logic**:
```typescript
// In CustomOperations constructor
if (exchangeOrParent instanceof Hyperliquid) {
  this.parent = exchangeOrParent;
  this.exchange = exchangeOrParent.exchange;
  this.infoApi = exchangeOrParent.info;
  this.symbolConversion = exchangeOrParent.symbolConversion;
  
  // FIXED - Properly extract wallet address
  this.walletAddress = (exchangeOrParent.exchange as any)?.wallet?.address || null;
  
  // FIXED - Properly extract vault address
  this.vaultAddress = (exchangeOrParent as any).vaultAddress || null;
}
```

2. **Add Position Query Logic**:
```typescript
private getPositionQueryAddress(): string {
  // If trading on behalf of a vault, query the vault's positions
  if (this.vaultAddress) {
    return this.vaultAddress;
  }
  // Otherwise query the user's own positions
  return this.getUserAddress();
}
```

3. **Update Position-Based Methods**:
```typescript
async marketClose(symbol: string, ...) {
  // Use vault address if available, wallet address otherwise
  const positionQueryAddress = this.getPositionQueryAddress();
  const positions = await this.infoApi.perpetuals.getClearinghouseState(positionQueryAddress);
  // ... rest of logic
}
```

## Impact
This bug affects:
- ✅ All vault trading operations using CustomOperations
- ✅ `marketClose()` method reliability
- ✅ `closeAllPositions()` method reliability
- ✅ Any position-based custom operations

## Additional Context
This is a critical bug for users who trade on behalf of vaults using the SDK's custom operations. It prevents proper vault management and could lead to trading errors or missed opportunities to close vault positions.