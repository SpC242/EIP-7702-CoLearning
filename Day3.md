## Day3

## EIP-7702
Add a new EIP-2718 transaction type that allows EOAs to set the code in their account.
EIP-7702 is better version of EIP-3074.
- **features**
    - **Batching**: allowing multiple operations from the same user in one atomic transaction.
    - **Sponsorship**: account X pays for a transaction on behalf of account Y. Account X could be paid in some other ERC-20 for this service, or it could be an application operator including the transactions of its users for free.
    - **Privilege de-escalation**: users can sign sub-keys and give them specific permissions that are much weaker than global access to the account. 
    - **Social recovery**: EIP-7702 supports social recovery for EOA users.
    - **Session keys**: Users can delegate session keys with lifecycle and scoped permissions to key custody services, allowing subscription models. 

- **new transaction type** 
    - **SET_CODE_TX_TYPE (0x04)**
        **TransactionPayload**
        ```solidity
        rlp([chain_id,
             nonce,
             max_priority_fee_per_gas,
             max_fee_per_gas, gas_limit,
             destination,
             value,
             data,
             access_list,
             authorization_list,
             signature_y_parity, signature_r, signature_s])
        ```
    - **authorization_list**
        ```solidity
        authorization_list = [[chain_id, address, nonce, y_parity, r, s], ...]
        ```
        The authorization_list is a list of tuples that store the address to code which the signer desires to execute in the context of their EOA. The transaction is considered invalid if the length of authorization_list is zero.

    - **workflow**
        1. Verify the chain id is either 0 or the chain’s current ID.
        2. Verify the **nonce** is less than 2**64 - 1.
        3. **authority = ecrecover(keccak(0x05 || rlp([chain_id, address, nonce])), y_parity, r, s)**
            - s value must be less or equal than secp256k1n/2, as specified in EIP-2.
        4. Add **authority** to **accessed_addresses** (as defined in EIP-2929.)
        5. Verify the code of **authority** is either empty or already delegated.
        6. Verify the nonce of **authority** is equal to **nonce**. In case authority does not exist in the trie, verify that nonce is equal to 0.
        7. Add **PER_EMPTY_ACCOUNT_COST - PER_AUTH_BASE_COST** gas to the global refund counter if authority exists in the trie.
        8. Set the code of **authority** to be **0xef0100 || address**. This is a **delegation designation**.
            - As a special case, if address is 0x0000000000000000000000000000000000000000 do not write the designation. Clear the accounts code and reset the account’s code hash to the empty hash 0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470.
        9. Increase the nonce of **authority** by one.
