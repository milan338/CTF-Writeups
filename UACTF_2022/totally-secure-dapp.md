A quick look at the app source which we have access to reveals a `next.config.js` file, indicating we are dealing with a nextjs app. In the `pages/api` directory, we can find a user-facing API - secret.ts - that will presumably provide us with the flag. Looking through `pages/api/secret.ts`, we can see the flag is given by an API request, but for the server to actually give us the flag, the deployed smart contract is queried, and three conditions must be met - the `_owner` property of the contract must be the same as our Ethereum address, the `_flagCaptured` property must be true, and the contract must have a balance > 0.005 ether.

The first part of this challenge requires us to gain ownership of the smart contract. We're given the contract source code, in which we can notice the solidity compiler version is on `0.4.24` - that means the `length` property of arrays is *not* read-only, and the contract will not revert state on numerical overflow / underflow. We can see the `editPost` function accepts a `uint256 index` and uses this to set `_authors[index] = msg.sender`, where _authors is an `address[] public` in storage. So, we need to somehow get this function to do an out-of-bounds write and replace _owner with our address, but it won't work just yet because solidity makes sure that the index we're accessing is less than the array length. Luckily, we can see the `removePost` function also takes in an arbitrary `uint256 index` parameter and decrements the _authors array length by one, without ever checking if the array is empty. Because of the compiler version, we can then call this function with index as 0 without ever having added to the _authors array, and the contract will happily decrement the array length of 0 by one, causing an underflow and wrapping the array length to $2^{256} - 1$.

Now that we can out-of-bounds write, we need to determine at what array index we need to write. Keep in mind that solidity stores data in storage contiguously in 32-byte slots, starting from slot 0 - for those unfamiliar with solidity storage layouts, this insight could be arrived at through searching for documentation on solidity variable layouts, finding sources such as [this](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html). If we look at the storage layout of the contract, we can see the two booleans from the inherited contract (which are uint8 under the hood) will be packed together into the 0th slot. Then there's the _contractId string which will have the 1st slot to itself, which means `_owner` will be in `slot 2`. Next up is the `_authors` array we're using, and while the length will be stored in `slot 3`, the actual array contents will be stored one after another starting from `slot keccak256(uint256(3))` (where 3 is the slot number of its length). So if we wanted to write to slot 2, which is before the array, first we'd need to get to slot 0 - if we try to write to slot $2^{256}$, we'd overflow uint256 and start writing at slot 0, and this happens at array index `2^256 - keccak(uint256(3))`. From here, we just add 2 to get to the owner slot. If we then called `editPost(2^256-keccak(uint256(3))+2, '', '')`, we would successfully overwrite _owner with our own address.

Next, we are required to send more than 0.005 ether to the contract to get the flag. We can't just send the ether to the contract, because the contract does not define any regular payable functions, meaning the fallback function will be executed on a transfer, but all that function does is call `revert`, so it'll just reject the ether. Instead, we have to exploit a quirk of solidity - the `selfdestruct` function. This function, when called by a contract, removes the deployed bytecode of that contract, but it also does something weird - it accepts a parameter of type address, and when the selfdestruct happens, all the contract's ether are *forcibly* sent to that specified address, bypassing even the fallback function. So, all we need to do is deploy a simple contract with a function to accept ether payment then immediately selfdestruct, sending that ether to the contract we're attacking. A simple implementation might look something like this

```solidity
pragma solidity 0.4.24;
contract Attacker {
    function attack(address contractToAttack) external payable {
        selfdestruct(contractToAttack);
    }
}
```
This could be quickly compiled and its ABI and bytecode extracted using the online Remix IDE.

When the above two attacks are chained together, we can finally call `captureFlag` on the contract, then we're able to get the flag. The secret API endpoint requires the request body to include the `userAddress`, the `contractAddress`, and the `userID` - digging through `components/connector/ConnectModal.tsx`, we can see in `getContractAddr` that the client receives this ID from a different API endpoint and stores it in the browser's localStorage, so we can find this using browser devtools. The final request might look something like

```http
GET http://website.com/api/secret HTTP/1.1
Content-Type: application/json
{
    "userAddress": "0x0000000000000000000000000000000000000000",
    "contractAddress": "0x0000000000000000000000000000000000000000",
    "userId": "B1LuFx7Yo2Npb8sRaggUH"
}
```

An example of the actual attack written in Typescript using the ethers.js library, using the provided contract ABI, might look something like

```typescript
import { Contract, Wallet, BigNumber, getDefaultProvider, ContractFactory } from 'ethers';
import { parseEther, solidityKeccak256 } from 'ethers/lib/utils';
async function main() {
    const wallet = new Wallet('0x0000000000000000000000000000000000000000');
    const provider = getDefaultProvider('ropsten');
    const account = wallet.connect(provider);
    const totallySecureDapp = new Contract('0x0000000000000000000000000000000000000000', abi, account);
    // First exploit - array length underflow - overwrite contract owner
    const i = BigNumber.from(2)
            .pow(256)
            .sub(solidityKeccak256(['uint256'], [3]))
            .add(2);
    await totallySecureDapp.removePost(0);
    await totallySecureDapp.editPost(i, '', '');
    // Second exploit - forcibly send ether to contract - gain access to captureFlag()
    const Attacker = new ContractFactory(attackerAbi, attackerBytecode, account);
    const attacker = await Attacker.deploy();
    await attacker.deployed();
    await attacker.attack(totallySecureDapp.address, { value: parseEther('0.006') })
    // Capture the flag
    await totallySecureDapp.captureFlag();
}
```
