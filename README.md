
# Arrakis contest details

- Join [Sherlock Discord](https://discord.gg/MABEWyASkp)
- Submit findings using the issue page in your private contest repo (label issues as med or high)
- [Read for more details](https://docs.sherlock.xyz/audits/watsons)

# Q&A

### Q: On what chains are the smart contracts going to be deployed?
Ethereum, Arbitrum, Gnosis
___

### Q: If you are integrating tokens, are you allowing only whitelisted tokens to work with the codebase or any complying with the standard? Are they assumed to have certain properties, e.g. be non-reentrant? Are there any types of <a href="https://github.com/d-xo/weird-erc20" target="_blank" rel="noopener noreferrer">weird tokens</a> you want to integrate?
- standard ERC-20 Tokens & USDC, USDT, DAI, BNB
- rebasing tokens, in this case, the Valantis Sovereign pool should be configured accordingly at deployment (setting the immutable variables)
___

### Q: Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED? If these integrations are trusted, should auditors also assume they are always responsive, for example, are oracles trusted to provide non-stale information, or VRF providers to respond within a designated timeframe?
TRUSTED
___

### Q: Are there any protocol roles? Please list them and provide whether they are TRUSTED or RESTRICTED, or provide a more comprehensive description of what a role can and can't do/impact.
Admin/Owner: TRUSTED

In HOT contract.
- Manager: TRUSTED
- liquidityProvider: TRUSTED
- signer: TRUSTED

In Arrakis modular :
- Executor of ArrakisStandardManager is RESTRICTED during rebalance action on vaults.
- Public vault owner is RESTRICTED
___

### Q: For permissioned functions, please list all checks and requirements that will be made before calling the function.
HOT.setPause - can be called immediately by manager, to ensure that HOT is shut down during emergency scenarios.
HOT.setFeeds - This allows liquidityProvider to accept oracle price feeds proposed by manager.
HOT.{setManager, setSigner, proposeFeeds, setMaxTokenVolumes, setHotFeeInBips, setMaxAllowedQuotes, setMaxOracleDeviationBips} - Only callable by manager and should have a timelock.

Arrakis Modular :
ArrakisMetaVaultFactory.{pause, unpause, setManager, whitelistDeployer, blacklistDeployer} - Only callable by ArrakisMetaVaultFactory owner.
ArrakisMetaVaultPrivate.{withdraw, whitelistDepositors, blacklistDepositors, whitelistModules, blacklistModules} - Only callable by ArrakisMetaVaultPrivate owner.
ArrakisMetaVaultPrivate.{deposit} - Only callable by depositors.
ArrakisMetaVaultPublic.{whitelistModules, blacklistModules} - Only callable by ArrakisMetaVaultPublic owner.
ArrakisStandardManager.{setDefaultReceiver, setReceiverByToken, decreaseManagerFeePIPS, finalizeIncreaseManagerFeePIPS, submitIncreaseManagerFeePIPS, withdrawManagerBalance} - Only callable by ArrakisStandardManager owner.
ArrakisStandardManager.{pause, unpause} - Only callable by guardian.
ArrakisStandardManager.{rebalance, setModule} - Only callable by vault's executor.
ArrakisStandardManager.{initManagement} - Only callable by ArrakisMetaVaultFactory.
ArrakisStandardManager.{updateVaultInfo} - Only callable by vault's owner.
ArrakisStandardManager.{announceStrategy} - Only callable by vault's stratAnnouncer.
Guardian.{setPauser} - Only callable by Guardian owner.
ModulePrivateRegistry/ModulePubleRegistry.{whitelistBeacons, blacklistBeacons} - Only callable by ModulePrivateRegistry/ModulePublicRegistry owner.
PALMVaultNFT.{mint} - Only callable by PALMVaultNFT owner.
RouterSwapExecutor.{swap} - Only callable by router.
ValantisModulePrivate.{fund, initializePosition, withdraw} - Only callable by metaVault.
ValantisModulePrivate.{pause, unpause} - Only callable by guardian.
ValantisModulePrivate.{setALMAndManagerFees} - Only callable by metaVault owner.
ValantisModulePrivate.{setManagerFeePIPS, setPriceBounds, swap} - Only callable by manager.
ValantisModulePrivate.{setManagerFeePIPS, setPriceBounds, swap} - Only callable by manager.
___

### Q: Is the codebase expected to comply with any EIPs? Can there be/are there any deviations from the specification?
HOT contract implements EIP-712 (strictly compliant)
___

### Q: Are there any off-chain mechanisms or off-chain procedures for the protocol (keeper bots, arbitrage bots, etc.)?
There will be an arbitrage bot, mostly to get developers and searchers to interact with the Valantis Sovereign Pool and HOT contract. If HOT quotes are enabled, arbitrage should not be necessary since signer can frequently update the AMM state (and in case of liveness failure, signer can be replaced). In AMM-only mode, arbitraging the AMM's spot price would be important.
___

### Q: Are there any hardcoded values that you intend to change before (some) deployments?
No
___

### Q: If the codebase is to be deployed on an L2, what should be the behavior of the protocol in case of sequencer issues (if applicable)? Should Sherlock assume that the Sequencer won't misbehave, including going offline?
yes, it should be assumed the Sequencer wont misbehave.
___

### Q: Should potential issues, like broken assumptions about function behavior, be reported if they could pose risks in future integrations, even if they might not be an issue in the context of the scope? If yes, can you elaborate on properties/invariants that should hold?
No
___

### Q: Please discuss any design choices you made.
In HOT.withdrawLiquidity, there is an extra condition to ensure that effective AMM liquidity cannot increase. Because the HOT AMM uses both passive and active reserves (where active reserves mean the portion that gets utilized in the constant product AMM), we added this extra layer of protection to ensure that a combination of swaps and withdrawals would not cause more reserves to be utilised by the AMM, which could lead to round-trip swap attacks. Moreover, we allow liquidityProvider to optionally pass in expected bounds for the spot price, in case further protection against price manipulation is needed. Unlike with HOT.depositLiquidity, we do not check for deviation between spot and oracle prices, since an oracle failure would lead to funds getting stuck.

HOT.depositLiquidity will check that the spot price is close enough to the oracle price (up to the configured price bounds), in case oracle price feeds are set. This is to help mitigate spot price manipulation attacks. Moreover, we allow liquidityProvider to optionally pass in expected bounds for the spot price, in case further protection against price manipulation is needed.

https://github.com/ValantisLabs/valantis-hot/blob/main/src/HOT.sol#L844-L848 This condition is relevant in case of rebase tokens. In case of negative rebases, effective AMM liquidity gets reduced, hence why we update it if the re-calculated value is lower than the existing one. For conventional constant-product AMMs with non-rebase/non-fee-on-transfer tokens it should always remain constant. Importantly, we cannot allow it to increase during AMM swaps, since this would lead to round-trip attacks.
___

### Q: Please list any known issues/acceptable risks that should not result in a valid finding.
HOT contract:
- signer role being compromised, which would allow it to swap against the HOT LPs, by manipulating AMM's spot price and/or fee parameters. We provide several recommendations in the docs about how to minimise damage in this scenario, as well as relying on manager role to pause the HOT contract. In general, there is a delicate trade-off between giving signer enough flexibility to use HOT quotes to update the AMM, and the level of trust required.
- liquidityProvider role trading against its own passive LPs by using a combination of deposits, swaps, setPriceBounds and/or withdrawals. liquidityProvider, in case it manages funds from passive LPs, must have sufficient internal protections (e.g. timelock, check spot price against an oracle, slippage protections, etc).
- Arbitrage against the AMM becomes possible after HOT quotes perform stale updates to AMM spot price and/or dynamic fee parameters. It is the responsibility of the signer to set these properly, as well as manager to set sensible bounds for these. 
- Attack scenarios that rely on parameters such as maxOracleDeviationBipsLower/Upper,  maxAllowedQuotes or minAmmFee being set too loose. We know that certain types of attacks, particularly with round-trip swaps, are possible if these are not configured appropriately. It is the responsibility of manager and liquidityProvider to set these in a way where they have enough flexibility to run their liquidity management strategies, without leaking significant amounts of value per-block in worst case scenarios (for example, the signer being compromised).

For arrakis-modular :
Any chainsecurity's findings that has not been treated is not a valid finding, see link below to chainsecurity report.
___

### Q: We will report issues where the core protocol functionality is inaccessible for at least 7 days. Would you like to override this value?
No
___

### Q: Please provide links to previous audits (if any).
Reports for valantis-hot are not yet available. These will be put into valantis-hot main branch.
Intermediary report of chainsecurity for arrakis-modular codebase : https://ipfs.io/ipfs/QmS3BkpyUbfw1DCW2KjYHBHc4uBvf2U77YWeEiHMgR4f9e
___

### Q: Please list any relevant protocol resources.
https://docs.valantis.xyz/
https://github.com/ArrakisFinance/arrakis-modular/tree/main/docs
___

### Q: Additional audit information.
We have taken SwapMath and LiquidityAmounts from UniV3, the branch that has Solidity 0.8 versions, without modifications.
___



# Audit scope


[arrakis-modular @ f150eb247d924187aa8e23c2684fb576cdbcdb9c](https://github.com/ArrakisFinance/arrakis-modular/tree/f150eb247d924187aa8e23c2684fb576cdbcdb9c)
- [arrakis-modular/src/ArrakisMetaVaultFactory.sol](arrakis-modular/src/ArrakisMetaVaultFactory.sol)
- [arrakis-modular/src/ArrakisMetaVaultPrivate.sol](arrakis-modular/src/ArrakisMetaVaultPrivate.sol)
- [arrakis-modular/src/ArrakisMetaVaultPublic.sol](arrakis-modular/src/ArrakisMetaVaultPublic.sol)
- [arrakis-modular/src/ArrakisPublicVaultRouter.sol](arrakis-modular/src/ArrakisPublicVaultRouter.sol)
- [arrakis-modular/src/ArrakisStandardManager.sol](arrakis-modular/src/ArrakisStandardManager.sol)
- [arrakis-modular/src/CreationCodePrivateVault.sol](arrakis-modular/src/CreationCodePrivateVault.sol)
- [arrakis-modular/src/CreationCodePublicVault.sol](arrakis-modular/src/CreationCodePublicVault.sol)
- [arrakis-modular/src/Guardian.sol](arrakis-modular/src/Guardian.sol)
- [arrakis-modular/src/ModulePrivateRegistry.sol](arrakis-modular/src/ModulePrivateRegistry.sol)
- [arrakis-modular/src/ModulePublicRegistry.sol](arrakis-modular/src/ModulePublicRegistry.sol)
- [arrakis-modular/src/PALMVaultNFT.sol](arrakis-modular/src/PALMVaultNFT.sol)
- [arrakis-modular/src/RouterSwapExecutor.sol](arrakis-modular/src/RouterSwapExecutor.sol)
- [arrakis-modular/src/TimeLock.sol](arrakis-modular/src/TimeLock.sol)
- [arrakis-modular/src/abstracts/ArrakisMetaVault.sol](arrakis-modular/src/abstracts/ArrakisMetaVault.sol)
- [arrakis-modular/src/abstracts/ModuleRegistry.sol](arrakis-modular/src/abstracts/ModuleRegistry.sol)
- [arrakis-modular/src/abstracts/ValantisHOTModule.sol](arrakis-modular/src/abstracts/ValantisHOTModule.sol)
- [arrakis-modular/src/modules/HOTOracleWrapper.sol](arrakis-modular/src/modules/HOTOracleWrapper.sol)
- [arrakis-modular/src/modules/ValantisHOTModulePrivate.sol](arrakis-modular/src/modules/ValantisHOTModulePrivate.sol)
- [arrakis-modular/src/modules/ValantisHOTModulePublic.sol](arrakis-modular/src/modules/ValantisHOTModulePublic.sol)
- [arrakis-modular/src/structs/SManager.sol](arrakis-modular/src/structs/SManager.sol)
- [arrakis-modular/src/structs/SPermit2.sol](arrakis-modular/src/structs/SPermit2.sol)
- [arrakis-modular/src/structs/SRouter.sol](arrakis-modular/src/structs/SRouter.sol)

[valantis-hot @ 9bfa0817521c71db1b932a28fada56bc7834631f](https://github.com/ValantisLabs/valantis-hot/tree/9bfa0817521c71db1b932a28fada56bc7834631f)
- [valantis-hot/src/HOT.sol](valantis-hot/src/HOT.sol)
- [valantis-hot/src/HOTOracle.sol](valantis-hot/src/HOTOracle.sol)
- [valantis-hot/src/libraries/AlternatingNonceBitmap.sol](valantis-hot/src/libraries/AlternatingNonceBitmap.sol)
- [valantis-hot/src/libraries/HOTConstants.sol](valantis-hot/src/libraries/HOTConstants.sol)
- [valantis-hot/src/libraries/HOTParams.sol](valantis-hot/src/libraries/HOTParams.sol)
- [valantis-hot/src/libraries/ReserveMath.sol](valantis-hot/src/libraries/ReserveMath.sol)
- [valantis-hot/src/libraries/utils/TightPack.sol](valantis-hot/src/libraries/utils/TightPack.sol)
- [valantis-hot/src/structs/HOTStructs.sol](valantis-hot/src/structs/HOTStructs.sol)




[arrakis-modular @ f150eb247d924187aa8e23c2684fb576cdbcdb9c](https://github.com/ArrakisFinance/arrakis-modular/tree/f150eb247d924187aa8e23c2684fb576cdbcdb9c)
- [arrakis-modular/src/ArrakisMetaVaultFactory.sol](arrakis-modular/src/ArrakisMetaVaultFactory.sol)
- [arrakis-modular/src/ArrakisMetaVaultPrivate.sol](arrakis-modular/src/ArrakisMetaVaultPrivate.sol)
- [arrakis-modular/src/ArrakisMetaVaultPublic.sol](arrakis-modular/src/ArrakisMetaVaultPublic.sol)
- [arrakis-modular/src/ArrakisPublicVaultRouter.sol](arrakis-modular/src/ArrakisPublicVaultRouter.sol)
- [arrakis-modular/src/ArrakisStandardManager.sol](arrakis-modular/src/ArrakisStandardManager.sol)
- [arrakis-modular/src/CreationCodePrivateVault.sol](arrakis-modular/src/CreationCodePrivateVault.sol)
- [arrakis-modular/src/CreationCodePublicVault.sol](arrakis-modular/src/CreationCodePublicVault.sol)
- [arrakis-modular/src/Guardian.sol](arrakis-modular/src/Guardian.sol)
- [arrakis-modular/src/ModulePrivateRegistry.sol](arrakis-modular/src/ModulePrivateRegistry.sol)
- [arrakis-modular/src/ModulePublicRegistry.sol](arrakis-modular/src/ModulePublicRegistry.sol)
- [arrakis-modular/src/PALMVaultNFT.sol](arrakis-modular/src/PALMVaultNFT.sol)
- [arrakis-modular/src/RouterSwapExecutor.sol](arrakis-modular/src/RouterSwapExecutor.sol)
- [arrakis-modular/src/TimeLock.sol](arrakis-modular/src/TimeLock.sol)
- [arrakis-modular/src/abstracts/ArrakisMetaVault.sol](arrakis-modular/src/abstracts/ArrakisMetaVault.sol)
- [arrakis-modular/src/abstracts/ModuleRegistry.sol](arrakis-modular/src/abstracts/ModuleRegistry.sol)
- [arrakis-modular/src/abstracts/ValantisHOTModule.sol](arrakis-modular/src/abstracts/ValantisHOTModule.sol)
- [arrakis-modular/src/modules/HOTOracleWrapper.sol](arrakis-modular/src/modules/HOTOracleWrapper.sol)
- [arrakis-modular/src/modules/ValantisHOTModulePrivate.sol](arrakis-modular/src/modules/ValantisHOTModulePrivate.sol)
- [arrakis-modular/src/modules/ValantisHOTModulePublic.sol](arrakis-modular/src/modules/ValantisHOTModulePublic.sol)
- [arrakis-modular/src/structs/SManager.sol](arrakis-modular/src/structs/SManager.sol)
- [arrakis-modular/src/structs/SPermit2.sol](arrakis-modular/src/structs/SPermit2.sol)
- [arrakis-modular/src/structs/SRouter.sol](arrakis-modular/src/structs/SRouter.sol)

