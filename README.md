# Lotos Finance Audit Report




## Audit Summary

This report presents findings from a security audit conducted for Lotus Finance's smart contract suite. The audit examined the codebase to identify vulnerabilities and provide recommendations for improving the security posture of the DeFi protocol.

## Findings Summary

The audit identified several vulnerabilities of varying severity across the protocol's contracts. This report details these issues, their potential impact, and recommended remediation steps to enhance the security and reliability of the protocol.

### High Findings

#### Improper AdminCap Handling

**Description:**  

The LotusLPFarm contract does not properly handle the LotusLPFarmCap capability object. The new<LP>() function creates a capability but merely returns it without transferring it to the transaction sender:

**POC:**  
``` rust
    public fun new<LP>(ctx: &mut TxContext): (LotusLPFarm<LP>, LotusLPFarmCap) {
        let mut farm = LotusLPFarm<LP> { /* ... */ };
        let farm_cap = LotusLPFarmCap {
            id: object::new(ctx),
            farm_id: object::id(&farm)
        };
        
        // Emit event
        event::emit(CreateLPFarmEvent {
            farm_id: object::id(&farm),
            farm_cap_id: object::id(&farm_cap),
        });

        (farm, farm_cap)
    }
```

This violates the capability pattern in Sui Move, where capabilities should be explicitly transferred to the intended owner. The same issue appears in several other capability-based contracts in the codebase.

**Impact:**

Without proper ownership transfer, there are several risks:

1. The capability could be lost or not properly accounted for
2. Administrative functions may become inaccessible if the capability is lost
3. The security model of the contract is compromised since anyone who gets a reference to the capability can use i

**Recommendation:**  
Modify the new<LP>() function to automatically transfer the capability to the transaction sender:

``` rust
public fun new<LP>(ctx: &mut TxContext): LotusLPFarm<LP> {
    let mut farm = LotusLPFarm<LP> { /* ... */ };
    let farm_cap = LotusLPFarmCap {
        id: object::new(ctx),
        farm_id: object::id(&farm)
    };
    
    // Emit event
    event::emit(CreateLPFarmEvent {
        farm_id: object::id(&farm),
        farm_cap_id: object::id(&farm_cap),
    });

    // Transfer capability to transaction sender using private transfer
    transfer::transfer(farm_cap, tx_context::sender(ctx));

    farm
}
```
This ensures the capability is properly owned and follows the standard capability pattern in Sui Move.

### Medium Findings

#### Deposit Function Type Mismatch

**Description:**  

The module documentation states that it "only supports single token (USDC) deposit," but the implementation of the deposit function accepts any coin type as a generic parameter T. This creates a mismatch between the documented behavior and the actual implementation.

**POC:**

```rust
#[test]
fun test_deposit_any_coin_type() {
    let mut test = begin(User1);
    let dummy_id = object::id_from_address(DUMMY_ADDRESS);
    
    let (vault, vault_cap) = lotus_db_vault::new<LOTUS_DB_VAULT_TESTS>(dummy_id, test.ctx());
    let vault_id = object::id(&vault);
    transfer::public_share_object(vault);
    transfer::public_share_object(vault_cap);
    
    test.next_tx(User1);
    {
        let mut vault = test.take_shared_by_id<LotusDBVault<LOTUS_DB_VAULT_TESTS>>(vault_id);
        
        // mint test coins of different types
        let usdc_coin: Coin<USDC> = coin::mint_for_testing<USDC>(1000000, test.ctx());
        let sui_coin: Coin<SUI> = coin::mint_for_testing<SUI>(1000000, test.ctx());
        
        // despite docs saying only USDC is supported, we can deposit any coin type
        vault.deposit(usdc_coin, test.ctx()); 
        vault.deposit(sui_coin, test.ctx());  
        
        // verify both deposits succeeded
        debug::print(&vault.bm_balance<LOTUS_DB_VAULT_TESTS, USDC>());
        debug::print(&vault.bm_balance<LOTUS_DB_VAULT_TESTS, SUI>());
        
        test_scenario::return_shared(vault);
    };
    end(test);
}
```
**Impact:**

1. Allows deposits of unsupported tokens which could break protocol assumptions
2. Creates potential for incorrect economic calculations and accounting
3. May lead to integration issues with other components expecting only USDC

**Recommendation:**

Modify the deposit function to restrict deposits to USDC only

```rust 
public fun deposit<LP>(
    self: &mut LotusDBVault<LP>,
    to_deposit: Coin<USDC>,
    ctx: &mut TxContext,
) {
    // Implementation for USDC only
}
```

### Low Findings

#### Duplicate Trade Cap Function

**Description:**  

The lotus_db_vault module contains an unused private function mint_trade_cap that would create a duplicate TradeCap for the same BalanceManager

**POC:**

``` rust
    fun mint_trade_cap<LP>(self: &mut LotusDBVault<LP>, ctx: &mut TxContext): TradeCap {
        event::emit(ActivateBotEvent {
            vault_id: object::id(self),
            trade_cap_id: object::id(&self.balance_manager),
            balance_manager_id: object::id(&self.balance_manager),
        });
        self.balance_manager.mint_trade_cap(ctx)
    }
```

This is problematic because a TradeCap is already created in the constructor:
``` rust
    let trade_cap = balance_manager.mint_trade_cap(ctx);
```

**Impact:**

Currently, this function is private and never called, so there is no immediate security risk. However, if this function were made public or called in future updates, it would create multiple trade capabilities for the same balance manager

**Recommendation:**

Either utilize this function properly in the constructor to emit the ActivateBotEvent, or remove it entirely

#### Unused Fee Collection and Storage Fields

**Description:**  

The LotusDBVault contract includes a performance fee mechanism and unused storage that are never utilized in the code. The contract defines performance_fee and collected_fees fields, but there is no implementation of fee calculation or collection anywhere in the module. Similarly, a free_balances Bag is initialized but never used throughout the contract.

**POC:**

```rust
public struct LotusDBVault<phantom LP> has key, store {
    // ...other fields...
    free_balances: Bag,                
    performance_fee: u64,              
    collected_fees: Bag,                
}
```

**Impact:**

Creates confusion about protocol economics and fee collection

**Recommendation:**

Either implement the fee collection mechanism or remove the unused fields
