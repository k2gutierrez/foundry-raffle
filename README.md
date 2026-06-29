<div align="center">
  <h1>🎟️ Automated Decentralized Raffle Application</h1>
  <p><b>A high-performance, secure, and fully autonomous lottery smart contract powered by Chainlink VRF v2.5 and Chainlink Automation</b></p>
</div>

## 📖 About the Project

The **Automated Decentralized Raffle Application** is a production-grade Web3 smart contract project built with **Solidity** and engineered using the **Foundry** development framework. At its core, the project solves the trust and automation problems inherent in traditional lottery systems. By leveraging decentralized oracle networks, it removes human intervention entirely from the equation, guaranteeing absolute fairness and predictability.

The system addresses two major design challenges in decentralized game mechanics:
1. **Verifiable Randomness:** Traditional on-chain data (like `block.timestamp` or `blockhash`) can be manipulated by miners or validators. This contract integrates **Chainlink VRF v2.5 (Verifiable Random Function)** to request and consume cryptographically secure, tamper-proof random numbers.
2. **Autonomous Execution:** Instead of requiring manual transactions to close rounds and pick winners, the architecture utilizes **Chainlink Automation** (`Keeper` network). The node network autonomously monitors the contract's state off-chain and executes the payout logic exactly when preconditions are met.

### Key Technical Highlights:
* **Solidity `^0.8.19`:** Harnesses strict state layouts, custom error codes (e.g., `Raffle__SendMoreToEnterRaffle`), and explicit type layouts for optimal gas management and security boundaries.
* **Chainlink VRF v2.5 Integration:** Implements the latest `VRFConsumerBaseV2Plus` standards, decoupling the subscription-funding model while ensuring safe callback verification via the `fulfillRandomWords` protocol.
* **CEI Pattern (Checks, Effects, Interactions):** Strict adherence to defensive programming models within state changes to completely eliminate reentrancy vulnerabilities before moving external funds.
* **Comprehensive Test Suite:** Designed with mock-driven environments (`VRFCoordinatorV2_5Mock`) allowing for full end-to-end local integration testing and state assertions.

---

## ⚙️ How It Works

The system operates via an automated state loop: **Open**, **Calculating**, and **Payout/Reset**. 

1. **Raffle Entry:** Users join the lottery pool by calling `enterRaffle()` and sending an amount equal to or greater than the configured `i_entranceFee`. Their addresses are pushed to the tracking array `s_players`.
2. **Upkeep Verification:** Chainlink Automation nodes continuously evaluate the off-chain view function `checkUpkeep()`. The upkeep is considered needed only when the contract has funds, has active players, a designated time interval has elapsed, and the raffle is currently `OPEN`.
3. **Randomness Request:** When upkeep is due, the automated node calls `performUpkeep()`. The contract flips its internal state to `CALCULATING` to lock further entries, and issues an asynchronous request to the Chainlink VRF Coordinator.
4. **Fulfillment & Payout:** The VRF Coordinator returns a verifiably random word to the contract's internal callback `fulfillRandomWords()`. The contract maps this number against the player pool via a modulo operation, securely transfers the entire native contract balance to the selected winner, resets the player pool, and updates the state back to `OPEN`.

![Project Raffle Diagram](./images/diagram.png)

---

## 🛠️ Technical Docs

### Main Project Contracts
* [Raffle.sol](src/Raffle.sol) — Core raffle mechanism, state management, and oracle integration.
* [DeployRaffle.s.sol](script/DeployRaffle.s.sol) — Automated deployment script configuring orchestrations across environments.
* [HelperConfig.s.sol](script/HelperConfig.s.sol) — Environment abstraction layer handling both local Anvil mocks and Live Testnet parameters.
* [Interactions.s.sol](script/Interactions.s.sol) — Automated scripts for managing VRF subscription creation, funding, and consumer attachment.

### Core Architecture Functions

#### `enterRaffle`
Allows participants to enter the raffle by paying the entry fee. Enforces state validation to ensure entries are accepted only when the raffle is open.
```solidity
    function enterRaffle() external payable {
        if (msg.value < i_entranceFee) {
            revert Raffle__SendMoreToEnterRaffle();
        }
        if (s_raffleState != RaffleState.OPEN) {
            revert Raffle__RaffleNotOpen();
        }
        s_players.push(payable(msg.sender));
        emit RaffleEntered(msg.sender);
    }
```

### checkUpkeep
The evaluation endpoint called off-chain by Chainlink Automation nodes to verify if all execution constraints have been satisfied.
```Solidity
    function checkUpkeep(
        bytes memory /* calldata */
    ) public view returns (bool upkeepNeeded, bytes memory /* performData */) {
        bool timeHasPassed = ((block.timestamp - s_lastTimeStamp) >= i_interval);
        bool isOpen = RaffleState.OPEN == s_raffleState;
        bool hasBalance = address(this).balance > 0;
        bool hasPlayers = s_players.length > 0;
        upkeepNeeded = (timeHasPassed && isOpen && hasBalance && hasPlayers);
        return (upkeepNeeded, "0x");
    }
```

### performUpkeep  
Triggered by the automation framework when checkUpkeep returns true. It locks entries and dispatches the request for verified randomness.
```Solidity
    function performUpkeep(bytes calldata /* performData */) external {
        (bool upkeepNeeded, ) = checkUpkeep("");
        if (!upkeepNeeded) {
            revert Raffle_UpkeepNotNeeded(
                address(this).balance,
                s_players.length,
                uint256(s_raffleState)
            );
        }
        s_raffleState = RaffleState.CALCULATING;

        VRFV2PlusClient.RandomWordsRequest memory request = VRFV2PlusClient.RandomWordsRequest({
            keyHash: i_gasLane,
            subId: i_subscriptionId,
            requestConfirmations: REQUEST_CONFIRMATIONS,
            callbackGasLimit: i_callbackGasLimit,
            numWords: NUM_WORDS,
            extraArgs: VRFV2PlusClient._argsToBytes(VRFV2PlusClient.ExtraArgsV1({nativePayment: false}))
        });
        uint256 requestId = s_vrfCoordinator.requestRandomWords(request);
        emit RequestedRaffleWinner(requestId);
    }
```

### fulfillRandomWords
The secure, internal callback triggered exclusively by the Chainlink VRF Coordinator. Selects the winner, transfers funds, and resets the contract context.
```Solidity
    function fulfillRandomWords(
        uint256 /*requestId*/,
        uint256[] calldata randomWords
    ) internal override {
        uint256 indexOfWinner = randomWords[0] % s_players.length;
        address payable recentWinner = s_players[indexOfWinner];
        s_recentWinner = recentWinner;

        s_raffleState = RaffleState.OPEN;
        s_players = new address payable[](0);
        s_lastTimeStamp = block.timestamp;

        emit WinnerPicked(s_recentWinner);

        (bool success,) = recentWinner.call{value: address(this).balance}("");
        if (!success){
            revert Raffle__TransferFailed();
        }
    }
```

📊 Execution Example  
- Step 1: Deployment & Subscription Setup
The contract is deployed using the DeployRaffle.s.sol script. If running on a local network, a mock VRF coordinator is deployed, a subscription is programmatically generated, funded with dummy LINK tokens, and the newly deployed Raffle contract is authorized as a valid consumer.  

- Step 2: Entering the Pool
Multiple users call the enterRaffle() function. For instance, User A, User B, and User C each deposit 0.01 ETH. The contract hold balance becomes 0.03 ETH, and s_players contains [UserA, UserB, UserC].

- Step 3: Upkeep Trigger
The time interval (e.g., 30 seconds) passes. A Chainlink Automation node notices that checkUpkeep() now evaluates to true (since the interval passed, balance > 0, players > 0, and the raffle state is OPEN).

- Step 4: Requesting Randomness
The automation node signs and broadcasts a transaction calling performUpkeep(). The contract state changes to CALCULATING (blocking any incoming entries) and emits RequestedRaffleWinner.

- Step 5: Fulfilling Randomness & Settlement
The Chainlink VRF node listens for the event, computes a cryptographically secure random value, and feeds it back by executing fulfillRandomWords(). The contract applies the modulo operation on the array length to select the index, sends the full 0.03 ETH prize pool to the winning address, wipes the players list clean, resets the timestamp tracker, and sets the raffle state back to OPEN for the next cycle.

📥 Installation
Ensure you have Foundry installed on your local environment. Then, install the project dependencies (Chainlink Brownie Contracts and Forge Standard Library) using the command below:
```Bash
forge install smartcontractkit/chainlink-brownie-contracts --no-commit && forge install foundry-rs/forge-std --no-commit
```

🧪 Testing
```Bash
forge test
```

📉 Coverage
```Bash
forge coverage
```

📜 Contract Address
- Provide your deployed contract addresses below upon testnet/mainnet deployment:
