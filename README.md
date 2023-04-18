# Lines of code
https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/MintingHub.sol#L208-L211
## Impact

     if (challenge.position.tryAvertChallenge(challenge.size, _bidAmountZCHF)) { //L208
        zchf.transferFrom(msg.sender, challenge.challenger, _bidAmountZCHF);
        challenge.position.collateral().transfer(msg.sender, challenge.size); //L211
        emit ChallengeAverted(address(challenge.position), _challengeNumber);
        delete challenges[_challengeNumber];
    }

L208-L211. In both cases, the function is called before a funds transfer and the function call is subject to a potential reentrancy vulnerability. The problem occurs when an attacker can reenter the function before the previous transaction is completed, allowing the attacker to manipulate the function's state and steal funds.
In the first case, 'tryAvertChallenge()' is a function that is called within a loop that can be executed multiple times. The function is designed to prevent a challenge from occurring if a participant offers an amount of ZCHF that meets the requirements of the offer. However, if this function allows the attacker to call an external function that results in a callback, the attacker can execute the function again before the previous transaction is completed, potentially modifying the function's state.
In the second case, the funds transfer is made after the challenge has been resolved and the winning participant has been determined. If this funds transfer is executed before verifying whether the previous function call has completed, an attacker can call back the function and manipulate the function's state before the previous transaction is completed.
In both cases, if the reentrancy vulnerability is successfully exploited, there can be a loss of funds or manipulation of the function's state, which can be used to obtain funds illicitly.
## Proof of Concept
https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/MintingHub.sol#L208-L211

To illustrate how the reentrancy vulnerability can be exploited, i create a malicious contract that takes advantage of the flaw in the 'MintingHub.sol' contract. Consider the following code:

pragma solidity ^0.8.0;

contract MaliciousContract {
    address public target;
    uint256 public amount;

    constructor(address _target, uint256 _amount) {
        target = _target;
        amount = _amount;
    }

    function attack() public {
        (bool success, ) = target.call(abi.encodeWithSignature("tryAvertChallenge(uint256,uint256)", amount, amount));
        require(success, "attack failed");
    }

    function () external payable {
        if (address(this).balance >= amount) {
            (bool success, ) = target.call(abi.encodeWithSignature("tryAvertChallenge(uint256,uint256)", amount, amount));
            require(success, "attack failed");
        }
    }
}

This malicious contract is designed to exploit the reentrancy vulnerability in the 'MintingHub.sol' contract. It has two functions: 'attack()' and 'fallback()'.

The 'attack()' function is the main function that executes the exploitation. It calls the 'tryAvertChallenge()' function of the 'MintingHub.sol' contract with a sufficiently large value in order to generate a challenge. However, instead of waiting for the function to finish executing, the malicious contract takes advantage of the reentrancy vulnerability and calls the 'tryAvertChallenge()' function again before the first call has been completed.

The 'fallback()' function is the function that allows for reentrancy. When the 'MintingHub.sol' contract makes a fund transfer to this contract, the 'fallback()' function is executed. If the malicious contract has sufficient funds, it will call the 'tryAvertChallenge()' function again, which will allow the attacker to manipulate the state of the contract.

The exploit is similar on line - 211

challenge.position.collateral().transfer(msg.sender, challenge.size);

The malicious contract can be similar to the previous example, but instead of calling the 'tryAvertChallenge()' function, it calls the function that transfers funds to 'msg.sender' and tries to exploit the reentrancy vulnerability during the transfer.
