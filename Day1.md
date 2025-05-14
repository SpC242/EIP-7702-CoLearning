## Day 1

## ERC-4337
An account abstraction proposal which completely avoids the need for consensus-layer protocol changes, instead relying on a separate mempool of **UserOperation** objects and miners either running custom code or connecting to a bundle marketplace.

- **UserOperation**
    | Field | Type | Description
    | - | - | - |
    | `sender` | `Address` | The wallet making the operation |
    | `nonce` | `uint256` | Anti-replay parameter; also used as the salt for first-time wallet creation |
    | `initCode` | `bytes` | The initCode of the wallet (only needed if the wallet is not yet on-chain and needs to be created) |
    | `callData` | `bytes` | The data to pass to the `sender` during the main execution call |

- Users send UserOperation objects that package up the user’s intent along with signatures and other data for verification. Either miners or bundlers using services such as Flashbots can package up a set of UserOperation objects into a single “bundle transaction”, which then gets included into an Ethereum block.


## EIP-3074 : Introduce New Opcodes
The two new EVM instructions(Opcodes), **AUTH** and **AUTHCALL** ,together enable a smart contract (known as an “invoker”) to take over the address of a EOA.

- **AUTH (0xf6)** takes a user's signature based on ECDSA and intended action (commit) and verifies that it was signed properly, allowing a smart contract to temporarily assume the identity of an EOA. It sets a variable that states the originating address of the transaction.

- **AUTHCALL (0xf7)** calls the target contract with the originator's address as the caller rather than the actual msg.sender.

- **workflow** 
    1. User signs a message (off-chain, not a tx).
    2. User or sponsor sends the message to an invoker contract as a tx.
    3. Invoker uses AUTHCALL to verify and call each target contract with the user's address as the sender.

- **benefits**
    - Batch transactions in the same signature.
    - Pay with any ERC-20 tokens as gas, not just ETH.
    - Have transactions sponsored by a third party, e.g. dApps.
    - Users can delegate control of their EOA to other keys which might have different security properties.

- **risks**
    - Invokers need to be fully audited, non-upgradeable, and trustless; otherwise, users' funds can easily be stolen. 
    - phishing — an attacker could trick users into delegating permissions/ownership to fraudulent invoker contracts.

- **batch call demo**
    ``` Solidity
    /**
    * @notice Authenticate and send the provided transaction payload(s) in the context of the signer. This function
    *  reverts if the signature is invalid, the nonce is incorrect, or one of the calls failed.
    * @param signature The signature of the transactions to verify.
    * @param transaction The nonce and payload(s) to send.
    */
    function invoke(Signature calldata signature, Transaction calldata transaction) external payable {
        require(transaction.payload.length > 0, "No transaction payload");

        address signer = authenticate(signature, transaction);
        require(signer != address(0), "Invalid signature");
        require(transaction.nonce == nonces[signer], "Invalid nonce");

        nonces[signer] += 1;

        for (uint256 i = 0; i < transaction.payload.length; i++) {
        bool success = call(transaction.payload[i]);
        require(success, "Transaction failed");
        }

        // To ensure that the caller does not send more funds than used in the transaction payload, we check if the contract
        // balance is zero here.
        require(address(this).balance == 0, "Invalid balance");
    }

    /**
    * @notice Authenticate based on the signature and transaction. This will calculate the EIP-712 message hash and use
    *  that as commit for authentication.
    * @param signature The signature to authenticate with.
    * @param transaction The transaction that was signed.
    * @return signer The recovered signer, or `0x0` if the signature is invalid.
    */
    function authenticate(Signature calldata signature, Transaction calldata transaction)
        private
        view
        returns (address signer)
    {
        bytes32 commit = getCommitHash(transaction);

        uint256 r = signature.r;
        uint256 s = signature.s;
        bool v = signature.v;

        // solhint-disable-next-line no-inline-assembly
        assembly {
        signer := auth(commit, v, r, s)
        }
    }

    /**
    * @notice Send an authenticated call to the address provided in the payload.
    * @dev Currently this function does not return the call data.
    * @param payload The payload to send.
    * @return success Whether the call succeeded.
    */
    function call(TransactionPayload calldata payload) private returns (bool success) {
        uint256 gasLimit = payload.gasLimit;
        address to = payload.to;
        uint256 value = payload.value;
        bytes memory data = payload.data;

        // solhint-disable-next-line no-inline-assembly
        assembly {
        success := authcall(gasLimit, to, value, 0, add(data, 0x20), mload(data), 0, 0)
        }
    }
    ```