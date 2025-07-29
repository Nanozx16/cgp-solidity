# cgp-solidity

Build
We recommend using the latest Node.js LTS version.

npm ci

npm run build

npm run test
Pre-compiled bytecodes can be found under Releases. Furthermore, pre-compiled bytecodes and ABI are shipped in the npm package and can be imported via:

npm i @axelar-network/axelar-cgp-solidity
const IAxelarGateway = require('@axelar-network/axelar-cgp-solidity/artifacts/interfaces/IAxelarGateway.json');

const AxelarGateway = require('@axelar-network/axelar-cgp-solidity/artifacts/contracts/AxelarGateway.sol/AxelarGateway.json');
Deploying contracts
See the Axelar contract deployments repository for relevant deployment/upgrade scripts:

Gateway deployment
Gateway upgrade
Gas service and deposit service deployment and upgrades
Live network testing
Check if the contract deployments repository supports the chain you will be using. Supported chains can be found here. If the chain is not already supported, proceed to steps 2-4, otherwise you may skip to step 5.
Navigate to the contract deployments repo here and clone the repository locally.
Within the contract deployments repo, edit the environment specific file inside the axelar-chains-config/info folder to add the chain you'll be testing. The following values need to be provided:
{
    "chains": {
        "example": {
            "name": "Example",
            "axelarId": "example",
            "chainId": 123,
            "rpc": "PROVIDER_RPC",
            "tokenSymbol": "EXM",
            "gasOptions": {
                "gasLimit": 8000000
            },
            "confirmations": 1
        }
    }
}
gasLimit override will skip auto gas estimation (which might be unreliable on certain chains for certain txs). confirmations indicates the number of block confirmations to wait for. axelarId is the unique id used to reference the chain on Axelar.

Return to the axelar-cgp-solidity repository. Once there, in the root directory of this repository, navigate to the hardhat.config.js file and modify the chains import line as shown below:
const chains = require(`/path/to/axelar-contract-deployments/axelar-chains-config/info/${env}.json`);
Create a keys.json file in this repo that contains the private keys for your accounts that will be used for testing. For some tests, such as the AxelarGateway tests, you need to provide at least two private keys (you can refer the test to find the number of accounts needed). At this point the keys.json file should resemble the example file below (chains can be left empty):
{
    "chains": {},
    "accounts": ["PRIVATE_KEY1", "PRIVATE_KEY2"]
}
Ensure that your accounts corresponding to the private keys provided have sufficient gas tokens on the chain.
Run
npm ci

npx hardhat test --network example
To run specific tests you may modify the test scripts by adding .only to describe and/or it blocks as shown below or grep the specific test names:
describe.only();
it.only();
There are two basic test groups that should be checked for the EVM-compatible chains:

npx hardhat test --network example --grep 'AxelarGateway'
npx hardhat test --network example --grep 'RpcCompatibility'
Debugging Steps
Explicitly pass getGasOptions() using utils.js file for some spceific transactions. See the code below for example
await sourceChainGateway
    .execute(
        await getSignedWeightedExecuteInput(await getTokenDeployData(false), [operatorWallet], [1], 1, [operatorWallet]),
        getGasOptions()
    )
    .then((tx) => tx.wait(network.config.confirmations));
Using the most up to date and fast rpc can help in tests execution runtime. Make sure the rate limit for the rpc is not exceeded.

Make sure that the account being used to broadcast transactions has enough native balance. The maximum gasLimit for a chain should be fetched from an explorer and set it in config file. You may also need to update the confirmations required for a transaction to be successfully included in a block in the config here depending on the network.

Note that certain tests can require upto 3 accounts.

Transactions can fail if previous transactions are not mined and picked up by the provide, therefore wait for a transaction to be mined after broadcasting. See the code below for example

await testToken.mint(userWallet.address, 1e9).then((tx) => tx.wait(network.config.confirmations));

// Or

const txExecute = await interchainGovernance.execute(commandIdGateway, governanceChain, governanceAddress, payload, getGasOptions());
const receiptExecute = await txExecute.wait(network.config.confirmations);
The changeEtherBalance check expects one tx in a block so change in balances might need to be tested explicitly for unit tests using changeEtherBalance.
Example flows
See Axelar examples for concrete examples.

Token transfer
Setup: A wrapped version of Token A is deployed (AxelarGateway.deployToken()) on each non-native EVM chain as an ERC-20 token (BurnableMintableCappedERC20.sol).
Given the destination chain and address, Axelar network generates a deposit address (the address where DepositHandler.sol is deployed, BurnableMintableCappedERC20.depositAddress()) on source EVM chain.
User sends their token A at that address, and the deposit contract locks the token at the gateway (or burns them for wrapped tokens).
Axelar network validators confirm the deposit Transfer event using their RPC nodes for the source chain (using majority voting).
Axelar network prepares a mint command, and validators sign off on it.
Signed command is now submitted (via any external relayer) to the gateway contract on destination chain AxelarGateway.execute().
Gateway contract authenticates the command, and mint's the specified amount of the wrapped Token A to the destination address.
Token transfer via AxelarDepositService
User wants to send wrapped token like WETH from chain A back to the chain B and to be received in native currency like Ether.
The un-wrap deposit address is generated by calling AxelarDepositService.addressForNativeUnwrap().
The token transfer deposit address for specific transfer is generated by calling AxelarDepositService.addressForTokenDeposit() with using the un-wrap address as a destination.
User sends the wrapped token to that address on the source chain A.
Axelar microservice detects the token transfer to that address and calls AxelarDepositService.sendTokenDeposit().
AxelarDepositService deploys DepositReceiver to that generated address which will call AxelarGateway.sendToken().
Axelar network prepares a mint command, and it gets executed on the destination chain gateway.
Wrapped token gets minted to the un-wrap address on the destination chain B.
Axelar microservice detects the token transfer to the un-wrap address and calls AxelarDepositService.nativeUnwrap().
AxelarDepositService deploys DepositReceiver which will call IWETH9.withdraw() and transfer native currency to the recipient address.
Cross-chain smart contract call
Setup:
Destination contract implements the IAxelarExecutable.sol interface to receive the message.
If sending a token, source contract needs to call ERC20.approve() beforehand to allow the gateway contract to transfer the specified amount on behalf of the sender/source contract.
Smart contract on source chain calls AxelarGateway.callContractWithToken() with the destination chain/address, payload and token.
An external service stores payload in a regular database, keyed by the hash(payload), that anyone can query by.
Similar to above, Axelar validators confirm the ContractCallWithToken event.
Axelar network prepares an AxelarGateway.approveContractCallWithMint() command, signed by the validators.
This is submitted to the gateway contract on the destination chain, which records the approval of the payload hash and emits the event ContractCallApprovedWithMint.
Any external relayer service listens to this event on the gateway contract, and calls the IAxelarExecutable.executeWithToken() on the destination contract, with the payload and other data as params.
executeWithToken of the destination contract verifies that the contract call was indeed approved by calling AxelarGateway.validateContractCallAndMint() on the gateway contract.
As part of this, the gateway contract records that the destination address has validated the approval, to not allow a replay.
The destination contract uses the payload for its own application.
References
Network resources: https://docs.axelar.dev/resources

Deployed contracts: https://docs.axelar.dev/resources/mainnet

General Message Passing Usage: https://docs.axelar.dev/dev/gmp

Example cross-chain token swap app: https://app.squidrouter.com

EVM module of the Axelar network that prepares commands for the gateway: https://github.com/axelarnetwork/axelar-core/blob/main/x/evm/keeper/msg_server.go
