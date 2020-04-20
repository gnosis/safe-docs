---
id: contracts_details
title: Architecture
sidebar_label: Architecture
---

A graphical overview of the smart contracts can be seen via [Piet](https://piet.slock.it/?gitHubUser=gnosis&gitHubRepo=safe-contracts&subDir=contracts).

## Gnosis Safe Transactions
A Safe transaction has the following parameters: A destination address, an Ether value, a data payload as a bytes array, <span style="color:#DB3A3D">`operation`</span> and <span style="color:#DB3A3D">`nonce`</span>.

The operation type specifies if the transaction is executed as a <span style="color:#DB3A3D">`CALL` </span>,<span style="color:#DB3A3D">`DELEGATECALL` </span>  or `CREATE` operation. While most wallet contracts only support <span style="color:#DB3A3D">`CALL` </span> operations, adding <span style="color:#DB3A3D">`DELEGATECALL` </span>operations allows to enhance the functionality of the wallet without updating the wallet code. As a <span style="color:#DB3A3D">`DELEGATECALL` </span>is executed in the context of the wallet contract, it can potentially mutate the state of the wallet (like changing owners) and therefore should only be used with known, trusted contracts. The <span style="color:#DB3A3D">`CREATE` </span> operation allows to create new contracts with bytecode sent from the wallet itself.

More information on delegate calls can be found in the [solidity docs](https://solidity.readthedocs.io/en/latest/introduction-to-smart-contracts.html#delegatecall-callcode-and-libraries)

The nonce prevents replay attacks. Each transaction should have a different nonce and once a transaction with a specific nonce has been executed it should not be possible to execute this transaction again. The concrete replay protection mechanism depends on the version of the Gnosis Safe and will be explained later.

## Contract Creations
As the creation of new contracts is a very gas consuming operation, Safe contracts use a proxy pattern where only one master copy of a contract is deployed once and all its copies are deployed as minimal proxy contracts pointing to the master copy contract. This pattern also allows to update the contract functionality later on by updating the address of the master copy in the proxy contract.

As contract constructors can only be executed once at the time the master copy is deployed, constructor logic has to be moved into an additional persistent setup function, which can be called to setup all copies of the master copy. This setup function has to be implemented in a way it can only be executed once. It is important to note that the master copy contract has to be persistent and there should be no possibility to execute a <span style="color:#DB3A3D">`selfdestruct`</span> call on the master copy contract.

It is **important** to know that it is possible to "hijack" a contract if the proxy creation and setup method are done in separate transactions. To avoid this it is possible to pass the initialisation data to the [ProxyFactory](https://github.com/gnosis/safe-contracts/blob/v1.0.0/contracts/proxies/ProxyFactory.sol) or the [DelegatingConstructorProxy](https://github.com/gnosis/safe-contracts/blob/v1.0.0/contracts/proxies/DelegateConstructorProxy.sol).

For more information about Proxy contracts read our blog post about [Solidity DelegateProxy Contracts](https://blog.gnosis.pm/solidity-delegateproxy-contracts-e09957d0f201).

## Contracts


### Base Contracts
#### SelfAuthorized.sol
The self authorized contract implements the <span style="color:#DB3A3D">`authorized()`</span> modifier so that only the contract itself is authorized to perform actions.

Multiple contracts use the <span style="color:#DB3A3D">`authorized()`</span> modifier. This modifier should be overwritten by a contract to implement the desired logic to check access to the protected methods.

#### Proxy.sol
The proxy contract implements only two functions: The constructor setting the address of the master copy and the fallback function forwarding all transactions sent to the proxy via a <span style="color:#DB3A3D">`DELEGATECALL` </span> to the master copy and returning all data returned by the <span style="color:#DB3A3D">`DELEGATECALL` </span>.

#### DelegateConstructorProxy.sol
This is an extension to the proxy contract that allows further initialization logic to be passed to the constructor.

#### PayingProxy.sol
This is an extension to the delegate constructor proxy contract that pays a specific amount to a target address after initialization.

#### ProxyFactory.sol
The proxy factory allows to create new proxy contracts pointing to a master copy and executing a function in the newly deployed proxy in one transaction. This additional transaction can for example execute the setup function to initialize the state of the contract.

#### MasterCopy.sol
The master copy contract defines the master copy field and has simple logic to change it. The master copy class should always be defined first if inherited.

#### EtherPaymentFallback.sol
Base contract with a fallback function to receive Ether payments.

#### Executor.sol
The executor implements logic to execute calls, delegatecalls and create operations.

#### ModuleManager.sol
The module manager is an executor implementation which allows the management (add, remove) of modules. These modules can execute transactions via the module manager.

A linked list is used to store the enabled modules in the smart contract. To modify the list with minimal gas usage it is required to specify the module that should be modified and the module that points to this module. This is important when disabling a module. If multiple transactions disabling modules are submitted at once it is important to note that the module that points to the module that should be disabled might have changed. The linked list requires a sentinel (start and end pointer). This sentinel is the <span style="color:#DB3A3D">`0x1`</span> address. Therefore this address cannot be used as a module.

#### OwnerManager.sol
The owner manager allows the management (addition, removal, replacement) of owners. It also specifies a threshold that can be used for all actions that require the confirmation of a specific amount of owners.

For managing the owners a linked list is used as well (see ModuleManager.sol). Modifying transactions that require to specify the owner pointing to the owner that should be modified include <span style="color:#DB3A3D">`removeOwner`</span> and <span style="color:#DB3A3D">`swapOwner`</span>. Also here the sentinel is the `0x1` and therefore it is not possible that this address becomes an owner.

#### BaseSafe.sol
The Gnosis Safe contract implements all basic multisignature functionality. It allows to execute Safe transactions and interact with Safe modules from internal methods. The contract provides no methods to interact with the Safe contract and also has no functionality to check if any interaction was approved by the required amount of owners. This logic and the methods to interact with the Gnosis Safe need to be implemented by the sub-contracts.

Safe transactions can be used to configure the wallet like managing owners, updating the master copy address or whitelisting of modules. All configuration functions can only be called via transactions sent from the Safe itself. This assures that configuration changes require owner confirmations.

Before a Safe transaction can be executed, the transaction has to be confirmed by the required number of owners.

There are multiple implementations of the Gnosis Safe contract with different methods to check if a transaction has been confirmed by the required owners.


### Gnosis Safe
#### GnosisSafe.sol
This contract implements verification of approvals when execution transactions via the contract.

To execute a transaction the method `execTransaction` can be used. To approve a transaction it is necessary to generate and encode the required signatures.

There are different types of signatures:

1. ECDSA signatures generated by Externally Owned Accounts
1. EIP-1271 based contract signatures
1. Pre-validated transaction hash signatures

For more information on signature types including how to generate and encode them, see [Signatures](./signatures.html).

### Modules
Modules allow to execute transactions from the Safe without the requirement of multiple signatures. For this Modules that have been added to a Safe can use the `execTransactionFromModule` function. Modules define their own requirements for execution. Modules need to implement their own replay protection.

Modules are smart contracts which implement a concrete Safe's functionality separating its logic from the Safe's contract. Keep in mind that modules allow the execution of transactions without needing confirmations, while this allows the implementation of many advanced use cases it also introduces additional attack vectors.

Modules can be included in the Safe according to owners' requirements, making the process of creating Safes more gas efficient (not all Safes should include all modules). They also enable developers to include their own features without compromising a Safe's core functionality, having all benefits of developing an isolated smart contract. Modules that are used on a Safe should always be reviewd and audited in a similar manner as the core fuctionality of the Safe, to make sure that no exploits are introduced.

#### StateChannelModule.sol
This module is meant to be used with state channels. It is a module similar to the base contract, but without the payment option (it has the same method for execution named `execTransaction`, but with different parameters). Furthermore this version doesn't store the nonce in the contract but for each transaction a nonce needs to be specified.

#### DailyLimitModule.sol
The Daily Limit Modules allows an owner to withdraw specified amounts of specified ERC20 tokens on a daily basis without confirmation by other owners. The daily limit is reset at midnight UTC. Ether is represented with the token address 0. Daily limits can be set via Safe transactions.

#### SocialRecoveryModule.sol
The Social Recovery Modules allows to recover a Safe in case access to owner accounts was lost. This is done by defining a minimum of 3 friends’ addresses as trusted parties. If all required friends confirm that a Safe owner should be replaced with another address, the Safe owner is replaced and access to the Safe can be restored. Every owner address can be replaced only once.

#### WhitelistModule.sol
The Whitelist Modules allows an owner to execute arbitrary transactions to specific addresses without confirmation by other owners. The whitelist can be maintained via Safe transactions.

### Libraries
Libraries can be called from the Safe via a <span style="color:#DB3A3D">`DELEGATECALL` </span>. They should not implement their own storage as this storage won’t be accessible via a <span style="color:#DB3A3D">`DELEGATECALL` </span>.

#### MultiSend.sol
This library allows to batch transactions and execute them at once. This is useful if user interactions require more than one transaction for one UI interaction like approving an amount of ERC20 tokens and calling a contract consuming those tokens. Each sub-transaction of the multi-send contract has an operation. With this it is possible to perform <span style="color:#DB3A3D">`CALL` </span>s and <span style="color:#DB3A3D">`DELEGATECALL` </span>s.

#### CreateAndAddModules.sol
This library allows to create new Safe modules and whitelist these modules for the Safe in one single transaction.
