# Ethernaut-Solutions-2025-Foundry-new
### ntroduction

This repository contains solutions for all levels of [**Ethernaut**](https://ethernaut.openzeppelin.com/) Challenges. Each markdown file has in-depth walk through and real solution code (mostly Web3.js command and Solidity code) for a certain level. This can be a study source for beginners who want to start the journey of solidity smart contract security and auditing.

All of the code from the solution has been run and proven to solve the game and for some of the source contracts in the challenge that use Solidity version <= 0.6.0, you can remove the source contract in your project, create an interface, and still use a higher version compiler. 

### Usage

[**Foundry**](https://github.com/foundry-rs/foundry) is the primary tool that we are going to use in this solution series. In most of the cases, the attack logic will finally be wrapped into the Solidity script. To first initialize the project, you can use command:

```bash
forge init
```

Then create files and copy the code from the markdown solution. Run the script using the command:

```bash
forge script $script_path$ --rpc-url $your_rpc_url$ --account $your_account$ --broadcast
```

### Ref

[https://github.com/OpenZeppelin/ethernaut/](https://github.com/OpenZeppelin/ethernaut/)

[https://github.com/Ching367436/ethernaut-motorbike-solution-after-decun-upgrade](https://github.com/Ching367436/ethernaut-motorbike-solution-after-decun-upgrade)

[https://github.com/piatoss3612/Ethernaut](https://github.com/piatoss3612/Ethernaut)

