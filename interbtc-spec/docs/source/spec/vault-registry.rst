.. _Vault-registry:

Vault Registry
==============

Overview
~~~~~~~~

The vault registry is the central place to manage vaults. Vaults can register themselves here, update their collateral, or can be liquidated.
Similarly, the issue, redeem, refund, and replace protocols call this module to assign vaults during issue, redeem, refund, and replace procedures.
Moreover, vaults use the registry to register public key for the :ref:`okd` and register addresses for the :ref:`op-return` scheme.

Data Model
~~~~~~~~~~

Constants
---------

Scalars
-------

MinimumCollateralVault
......................

The minimum collateral a vault needs to provide to participate in the issue process. 

.. note:: This is a protection against spamming the protocol with very small collateral amounts. Vaults are still able to withdraw the collateral after registration, but at least it requires an additional transaction fee, and it provides protection against accidental registration with very low amounts of collateral.

.. _punishmentDelay:

PunishmentDelay
...............

Time period in which a Vault cannot participate in issue, redeem or replace requests.

- Measured in Parachain blocks
- Initial value: 1 day (Parachain constant)

.. _SecureCollateralThreshold:

SecureCollateralThreshold
.........................

Determines the over-collateralization rate for collateral locked by Vaults, necessary for issuing tokens. 

The Vault can take on issue requests depending on the collateral it provides and under consideration of the ``SecureCollateralThreshold``.
The maximum amount of interBTC a vault is able to support during the issue process is based on the following equation:

:math:`\text{max(interBTC)} = \text{collateral} * \text{ExchangeRate} / \text{SecureCollateralThreshold}`.

* The Secure Collateral Threshold MUST be greater than the Liquidation Threshold.
* The Secure Collateral Threshold MUST be greater than the Premium Redeem Threshold.

.. note:: As an example, assume we use ``DOT`` as collateral, we issue ``interBTC`` and lock ``BTC`` on the Bitcoin side. Let's assume the ``BTC``/``DOT`` exchange rate is ``80``, i.e., one has to pay 80 ``DOT`` to receive 1 ``BTC``. Further, the ``SecureCollateralThreshold`` is 200%, i.e., a vault has to provide two-times the amount of collateral to back an issue request. Now let's say the vault deposits 400 ``DOT`` as collateral. Then this vault can back at most 2.5 interBTC as: :math:`400 * (1/80) / 2 = 2.5`.


.. _PremiumCollateralThreshold:

PremiumRedeemThreshold
......................

Determines the rate for the collateral rate of Vaults, at which users receive a premium, allocated from the Vault's collateral, when performing a :ref:`redeem-protocol` with this Vault. 

* The Premium Redeem Threshold MUST be greater than the Liquidation Threshold.

.. _LiquidationThreshold:

LiquidationThreshold
....................

Determines the lower bound for the collateral rate in issued tokens. If a Vault’s collateral rate drops below this, automatic liquidation is triggered.

* The Liquidation Threshold MUST be greater than 100% for any collateral asset.

LiquidationVault
.................

Account identifier of an artificial vault maintained by the VaultRegistry to handle interBTC balances and DOT collateral of liquidated Vaults. That is, when a vault is liquidated, its balances are transferred to ``LiquidationVault`` and claims are later handled via the ``LiquidationVault``.


.. note:: A Vault's token balances and DOT collateral are transferred to the ``LiquidationVault`` as a result of automated liquidations and :ref:`reportVaultTheft`.


Maps
----

.. _vaults:

Vaults
......

Mapping from accounts of Vaults to their struct. ``<Account, Vault>``.


Structs
-------

Vault
.....

Stores the information of a Vault.

.. tabularcolumns:: |l|l|L|

=========================  ==================  ========================================================
Parameter                  Type                Description
=========================  ==================  ========================================================
``wallet``                 Wallet<BtcAddress>  A set of Bitcoin address(es) of this vault, used for theft detection. Additionally, it contains the btcPublicKey used for generating deposit addresses in the issue process. 
``status``                 VaultStatus         Current status of the vault (Active, Liquidated, CommittedTheft)
``bannedUntil``            BlockNumber         Block height until which this vault is banned from being used for Issue, Redeem (except during automatic liquidation) and Replace . 
``toBeIssuedTokens``       interBTC            Number of interBTC tokens currently requested as part of an uncompleted issue request.
``issuedTokens``           interBTC            Number of interBTC tokens actively issued by this Vault.
``toBeRedeemedTokens``     interBTC            Number of interBTC tokens reserved by pending redeem and replace requests. 
``toBeReplacedTokens``     interBTC            Number of interBTC tokens requested for replacement.
``replaceCollateral``      DOT                 Griefing collateral to be used for accepted replace requests.
``liquidatedCollateral``   DOT                 Any collateral that is locked for remaining to_be_redeemed on liquidation.
=========================  ==================  ========================================================

.. note:: This specification currently assumes for simplicity that a vault will reuse the same BTC address, even after multiple redeem requests. **[Future Extension]**: For better security, Vaults may desire to generate new BTC addresses each time they execute a redeem request. This can be handled by pre-generating multiple BTC addresses and storing these in a list for each Vault. Caution is necessary for users which execute issue requests with "old" vault addresses - these BTC must be moved to the latest address by Vaults. 

External Functions
~~~~~~~~~~~~~~~~~~


.. _registerVault:

registerVault
-------------

Registers a new Vault. The vault locks up some amount of collateral, and provides a public key which is used for the :ref:`okd`.

Specification
.............

*Function Signature*

``registerVault(vault, collateral, btcPublicKey)``

*Parameters*

* ``vault``: The account of the vault to be registered.
* ``collateral``: to-be-locked collateral.
* ``btcPublicKey``: public key used to derive deposit keys with the :ref:`okd`.


*Events*

* :ref:`registerVaultEvent`

*Preconditions*

* The function call MUST be signed by ``vaultId``.
* The BTC Parachain status in the :ref:`security` component MUST NOT be ``SHUTDOWN:2``.
* The vault MUST NOT be registered yet
* The vault MUST have sufficient funds to lock the collateral
* ``collateral > MinimumCollateralVault``, i.e., the vault MUST provide sufficient collateral (above the spam protection threshold).

*Postconditions*

* The vault's free balance MUST decrease by ``collateral``.
* The vault's reserved balance MUST increase by ``collateral``.
* The new vault MUST be created as follows:

    * ``vault.wallet``: MUST be empty.
    * ``vault.status``: MUST be set to ``active=true``.
    * ``vault.bannedUntil``: MUST be empty.
    * ``vault.toBeIssuedTokens``: MUST be zero.
    * ``vault.issuedTokens``: MUST be zero.
    * ``vault.toBeRedeemedTokens``: MUST be zero.
    * ``vault.toBeReplacedTokens``: MUST be zero.
    * ``vault.replaceCollateral``: MUST be zero.
    * ``vault.liquidatedCollateral``: MUST be zero.

* The new vault MUST be inserted into :ref:`vaults` using their account identifier as key.

.. _registerAddress:

registerAddress
---------------

Add a new BTC address to the vault's wallet. Typically this function is called by the vault client to register a return-to-self address, prior to making redeem/replace payments. If a vault makes a payment to an address that is not registered, nor is a valid redeem/replace payment, it will be marked as theft.

Specification
.............

*Function Signature*

``registerAddress(vaultId, address)``

*Parameters*

* ``vaultId``: the account of the vault.
* ``address``: a valid BTC address.

*Events*

* :ref:`registerAddressEvent`



Precondition

* The function call MUST be signed by ``vaultId``.
* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST NOT be registered.

*Postconditions*

* ``address`` MUST be added to the vault's wallet.

 
.. _updatePublicKey:

updatePublicKey
---------------

Changes a vault's public key that is used for the :ref:`okd`.

Specification
.............

*Function Signature*

``updatePublicKey(vaultId, publicKey)``

*Parameters*

* ``vaultId``: the account of the vault.
* ``publicKey``: the new BTC public key of the vault.

*Events*

* :ref:`updatePublicKeyEvent`


*Preconditions*

* The function call MUST be signed by ``vaultId``.
* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.

*Postconditions*

* The vault's public key MUST be set to ``publicKey``.

.. _depositCollateral:

depositCollateral
------------------------

The vault locks additional collateral as a security against stealing the Bitcoin locked with it. 

Specification
.............

*Function Signature*

``lockCollateral(vaultId, collateral)``

*Parameters*

* ``vaultId``: The account of the vault locking collateral.
* ``collateral``: to-be-locked collateral.

*Events*

* :ref:`depositCollateralEvent`

Precondition
............

* The function call MUST be signed by ``vaultId``.
* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* The vault MUST have sufficient unlocked collateral to lock.

*Postconditions*

* Function :ref:`staking_depositStake` MUST complete successfully - parameterized by ``vaultId`` and ``collateral``.

.. _withdrawCollateral:

withdrawCollateral
------------------

A vault can withdraw its *free* collateral at any time, as long as the collateralization ratio remains above the ``SecureCollateralThreshold``. Collateral that is currently being used to back issued interBTC remains locked until the vault is used for a redeem request (full release can take multiple redeem requests).


Specification
.............

*Function Signature*

``withdrawCollateral(vaultId, withdrawAmount)``

*Parameters*

* ``vaultId``: The account of the vault withdrawing collateral.
* ``withdrawAmount``: To-be-withdrawn collateral.

*Events*

* :ref:`withdrawCollateralEvent`

*Preconditions*

* The function call MUST be signed by ``vaultId``.
* The BTC Parachain status in the :ref:`security` component MUST be set to ``RUNNING:0``.
* A vault with id ``vaultId`` MUST be registered.
* The collatalization rate of the vault MUST remain above ``SecureCollateralThreshold`` after the withdrawal of ``withdrawAmount``.
* After the withdrawal, the vault's ratio of nominated collateral to own collateral must remain above the value returned by :ref:`getMaxNominationRatio`.

*Postconditions*

* Function :ref:`staking_withdrawStake` MUST complete successfully - parameterized by ``vaultId`` and ``withdrawAmount``.
* The vault's free balance MUST increase by ``withdrawAmount``.







Internal Functions
~~~~~~~~~~~~~~~~~~

.. _tryIncreaseToBeIssuedTokens:

tryIncreaseToBeIssuedTokens
---------------------------

During an issue request function (:ref:`requestIssue`), a user must be able to assign a vault to the issue request. As a vault can be assigned to multiple issue requests, race conditions may occur. To prevent race conditions, a Vault's collateral is *reserved* when an ``IssueRequest`` is created - ``toBeIssuedTokens`` specifies how much interBTC is to be issued (and the reserved collateral is then calculated based on :ref:`getPrice`).

Specification
.............

*Function Signature*

``tryIncreaseToBeIssuedTokens(vaultId, tokens)``

*Parameters*

* ``vaultId``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC to be locked.

*Events*

* :ref:`increaseToBeIssuedTokensEvent`


*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST be set to ``RUNNING:0``.
* A vault with id ``vaultId`` MUST be registered.
* The vault MUST have sufficient collateral to remain above the ``SecureCollateralThreshold`` after issuing ``tokens``.
* The vault status MUST be `Active(true)`
* The vault MUST NOT be banned

*Postconditions*

* The vault's ``toBeIssuedTokens`` MUST be increased by an amount of ``tokens``.

.. _decreaseToBeIssuedTokens:

decreaseToBeIssuedTokens
------------------------

A Vault's committed tokens are unreserved when an issue request (:ref:`cancelIssue`) is cancelled due to a timeout (failure!). If the vault has been liquidated, the tokens are instead unreserved on the liquidation vault.

Specification
.............

*Function Signature*

``decreaseToBeIssuedTokens(vaultId, tokens)``

*Parameters*

* ``vaultId``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC to be unreserved.

*Events*

* :ref:`decreaseToBeIssuedTokensEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* If the vault is not liquidated, it MUST have at least ``tokens`` ``toBeIssuedTokens``. 
* If the vault *is* liquidated, it MUST have at least ``tokens`` ``toBeIssuedTokens``.

*Postconditions*

* If the vault is *not* liquidated, its ``toBeIssuedTokens`` MUST be decreased by an amount of ``tokens``. 
* If the vault *is* liquidated, the liquidation vault's ``toBeIssuedTokens`` MUST be decreased by an amount of ``tokens``. 

.. _issueTokens:

issueTokens
-----------

The issue process completes when a user calls the :ref:`executeIssue` function and provides a valid proof for sending BTC to the vault. At this point, the ``toBeIssuedTokens`` assigned to a vault are decreased and the ``issuedTokens`` balance is increased by the ``amount`` of issued tokens.

Specification
.............

*Function Signature*

``issueTokens(vaultId, amount)``

*Parameters*

* ``vaultId``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC that were just issued.


*Events*

* :ref:`issueTokensEvent`


*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* If the vault is *not* liquidated, its ``toBeIssuedTokens`` MUST be greater than or equal to ``tokens``.
* If the vault *is* liquidated, the ``toBeIssuedTokens`` of the liquidation vault MUST be greater than or equal to ``tokens``.

*Postconditions*

* If the vault is *not* liquidated, its ``toBeIssuedTokens`` MUST be decreased by ``tokens``, while its ``issuedTokens`` MUST be increased by ``tokens``.
* If the vault is *not* liquidated, function :ref:`reward_depositStake` MUST complete successfully - parameterized by ``vaultId`` and ``tokens``.
* If the vault *is* liquidated, the ``toBeIssuedTokens`` of the liquidation vault MUST be decreased by ``tokens``, while its ``issuedTokens`` MUST be increased by ``tokens``.


.. _tryIncreaseToBeRedeemedTokens:

tryIncreaseToBeRedeemedTokens
-----------------------------

Add an amount of tokens to the ``toBeRedeemedTokens`` balance of a vault. This function serves as a prevention against race conditions in the redeem and replace procedures.
If, for example, a vault would receive two redeem requests at the same time that have a higher amount of tokens to be issued than his ``issuedTokens`` balance, one of the two redeem requests should be rejected.

Specification
.............

*Function Signature*

``tryIncreaseToBeRedeemedTokens(vaultId, tokens)``

*Parameters*

* ``vaultId``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC to be redeemed.

*Events*

* :ref:`increaseToBeRedeemedTokensEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* The vault MUST NOT be liquidated.
* The vault MUST have sufficient tokens to reserve, i.e. ``tokens`` must be less than or equal to ``issuedTokens - toBeRedeemedTokens``.

*Postconditions*

* The vault's ``toBeRedeemedTokens`` MUST be increased by ``tokens``.

.. _decreaseToBeRedeemedTokens:

decreaseToBeRedeemedTokens
--------------------------

Subtract an amount tokens from the ``toBeRedeemedTokens`` balance of a vault. This function is called from :ref:`cancelRedeem`.

Specification
.............

*Function Signature*

``decreaseToBeRedeemedTokens(vaultId, tokens)``

*Parameters*

* ``vaultId``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC not to be redeemed.


*Events*

* :ref:`decreaseToBeRedeemedTokensEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component must not be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* If the vault is *not* liquidated, its ``toBeRedeemedTokens`` MUST be greater than or equal to ``tokens``.
* If the vault *is* liquidated, the ``toBeRedeemedTokens`` of the liquidation vault MUST be greater than or equal to ``tokens``.

*Postconditions*

* If the vault is *not* liquidated, its ``toBeRedeemedTokens`` MUST be decreased by ``tokens``.
* If the vault *is* liquidated, the ``toBeRedeemedTokens`` of the liquidation vault MUST be decreased by ``tokens``.


.. _decreaseTokens:

decreaseTokens
--------------

Decreases both the ``toBeRedeemed`` and ``issued`` tokens, effectively burning the tokens. This is called from :ref:`cancelRedeem`.

Specification
.............

*Function Signature*

``decreaseTokens(vaultId, user, tokens)``

*Parameters*

* ``vaultId``: The BTC Parachain address of the Vault.
* ``userId``: The BTC Parachain address of the user that made the redeem request.
* ``tokens``: The amount of interBTC that were not redeemed.


*Events*

* :ref:`decreaseTokensEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component must not be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* If the vault is *not* liquidated, its ``toBeRedeemedTokens`` and ``issuedTokens`` MUST be greater than or equal to ``tokens``.
* If the vault *is* liquidated, the ``toBeRedeemedTokens`` and ``issuedTokens`` of the liquidation vault MUST be greater than or equal to ``tokens``.

*Postconditions*

* If the vault is *not* liquidated, its ``toBeRedeemedTokens`` and ``issuedTokens`` MUST be decreased by ``tokens``.
* If the vault *is* liquidated, the ``toBeRedeemedTokens`` and ``issuedTokens`` of the liquidation vault MUST be decreased by ``tokens``.


.. _redeemTokens:

redeemTokens
------------

Reduces the to-be-redeemed tokens when a redeem request completes

Specification
.............

*Function Signature*

``redeemTokens(vaultId, tokens, premium, redeemerId)``

*Parameters*


* ``vaultId``: the id of the vault from which to redeem tokens
* ``tokens``: the amount of tokens to be decreased
* ``premium``: amount of collateral to be rewarded to the redeemer if the vault is not liquidated yet
* ``redeemerId``: the id of the redeemer


*Events*

One of:

* :ref:`redeemTokensEvent`
* :ref:`redeemTokensPremiumEvent`
* :ref:`redeemTokensLiquidatedVaultEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* If the vault is *not* liquidated:
   * The vault's ``toBeRedeemedTokens`` must be greater than or equal to ``tokens``.
   * If ``premium > 0``, then the vault's ``backingCollateral`` (as calculated via :ref:`computeStakeAtIndex`) must be greater than or equal to ``premium``.
* If the vault *is* liquidated, then the liquidation vault's ``toBeRedeemedTokens`` must be greater than or equal to ``tokens``
  
*Postconditions*

* If the vault is *not* liquidated:
   * The vault's ``toBeRedeemedTokens`` MUST be decreased by ``tokens``, and its ``issuedTokens`` MUST increase by the same amount.
   * If ``premium = 0``, then the ``RedeemTokens`` event is emitted
   * If ``premium > 0``, then ``premium`` is transferred from the vault's collateral to the redeemer. The ``RedeemTokensPremium`` event is emitted.
   * Function :ref:`reward_withdrawStake` MUST complete successfully - parameterized by ``vaultId`` and ``tokens``.
* If the vault *is* liquidated, then the liquidation vault's ``toBeRedeemedTokens`` MUST be decreased by ``tokens``, and its ``issuedTokens`` MUST increase by the same amount. The ``RedeemTokensLiquidatedVault`` event is emitted.

.. _redeemTokensLiquidation:

redeemTokensLiquidation
------------------------

Handles redeem requests which are executed against the LiquidationVault. Reduces the issued token of the LiquidationVault and slashes the
corresponding amount of collateral.

Specification
.............

*Function Signature*

``redeemTokensLiquidation(redeemerId, tokens)``

*Parameters*

* ``redeemerId`` : The account of the user redeeming interBTC.
* ``tokens``: The amount of interBTC to be burned, in exchange for collateral.

*Events*

* :ref:`redeemTokensLiquidationEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* The liquidation vault MUST have sufficient tokens, i.e. ``tokens`` MUST be less than or equal to its ``issuedTokens - toBeRedeemedTokens``.

*Postconditions*

* The liquidation vault's ``issuedTokens`` MUST decrease by ``tokens``.
* The redeemer MUST have received an amount of collateral equal to ``(tokens / liquidationVault.issuedTokens) * liquidationVault.backingCollateral``.

.. _increaseToBeReplacedTokens:

increaseToBeReplacedTokens
--------------------------

Increases the toBeReplaced tokens of a vault, which indicates how many tokens other vaults can replace in total.

Specification
.............

*Function Signature*

``increaseToBeReplacedTokens(oldVault, tokens, collateral)``

*Parameters*

* ``vaultId``: Account identifier of the vault to be replaced.
* ``tokens``: The amount of interBTC replaced.
* ``collateral``: The extra collateral provided by the new vault as griefing collateral for potential accepted replaces. 

*Returns*

* A tuple of the new total ``toBeReplacedTokens`` and ``replaceCollateral``.

*Events*

* :ref:`increaseToBeReplacedTokensEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.
* The vault MUST NOT be liquidated.
* The vault's increased ``toBeReplaceedTokens`` MUST NOT exceed ``issuedTokens - toBeRedeemedTokens``.

*Postconditions*

* The vault's ``toBeReplaceTokens`` MUST be increased by ``tokens``.
* The vault's ``replaceCollateral`` MUST be increased by ``collateral``.



.. _decreaseToBeReplacedTokens:

decreaseToBeReplacedTokens
-----------------------------

Decreases the toBeReplaced tokens of a vault, which indicates how many tokens other vaults can replace in total.

Specification
.............

*Function Signature*

``decreaseToBeReplacedTokens(oldVault, tokens)``

*Parameters*

* ``vaultId``: Account identifier of the vault to be replaced.
* ``tokens``: The amount of interBTC replaced.

*Returns*

* A tuple of the new total ``toBeReplacedTokens`` and ``replaceCollateral``.

*Events*

* :ref:`decreaseToBeReplacedTokensEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``vaultId`` MUST be registered.

*Postconditions*

* The vault's ``replaceCollateral`` MUST be decreased by ``(min(tokens, toBeReplacedTokens) / toBeReplacedTokens) * replaceCollateral``.
* The vault's ``toBeReplaceTokens`` MUST be decreased by ``min(tokens, toBeReplacedTokens)``.
  
.. note:: the ``replaceCollateral`` is not actually unlocked - this is the responsibility of the caller. It is implemented this way, because in :ref:`requestRedeem` it needs to be unlocked, whereas in :ref:`requestReplace` it must remain locked.  




.. _replaceTokens:

replaceTokens
-------------

When a replace request successfully completes, the ``toBeRedeemedTokens`` and the ``issuedToken`` balance must be reduced to reflect that removal of interBTC from the ``oldVault``.Consequently, the ``issuedTokens`` of the ``newVault`` need to be increased by the same amount.

Specification
.............

*Function Signature*

``replaceTokens(oldVault, newVault, tokens, collateral)``

*Parameters*

* ``oldVault``: Account identifier of the vault to be replaced.
* ``newVault``: Account identifier of the vault accepting the replace request.
* ``tokens``: The amount of interBTC replaced.
* ``collateral``: The collateral provided by the new vault. 


*Events*

* :ref:`replaceTokensEvent`

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``oldVault`` MUST be registered.
* A vault with id ``newVault`` MUST be registered.
* If ``oldVault`` is *not* liquidated, its ``toBeRedeemedTokens`` and ``issuedTokens`` MUST be greater than or equal to ``tokens``.
* If ``oldVault`` *is* liquidated, the liquidation vault's ``toBeRedeemedTokens`` and ``issuedTokens`` MUST be greater than or equal to ``tokens``.
* If ``newVault`` is *not* liquidated, its ``toBeIssuedTokens`` MUST be greater than or equal to ``tokens``.
* If ``newVault`` *is* liquidated, the liquidation vault's ``toBeIssuedTokens`` MUST be greater than or equal to ``tokens``.

*Postconditions*

* If ``oldVault`` is *not* liquidated:
   * The ``oldVault``'s ``toBeRedeemedTokens`` and ``issuedTokens`` MUST be decreased by the amount ``tokens``.
   * The ``oldVault``'s collateral free balance MUST be increased by ``tokens / toBeRedeemed``.
* If ``oldVault`` *is* liquidated, the liquidation vault's ``toBeRedeemedTokens`` and ``issuedTokens`` are decrease by the amount ``tokens``.
* If ``newVault`` is *not* liquidated, its ``toBeIssuedTokens`` is decreased by ``tokens``, while its ``issuedTokens`` is increased by the same amount.
* If ``newVault`` *is* liquidated, the liquidation vault's  ``toBeIssuedTokens`` is decreased by ``tokens``, while its ``issuedTokens`` is increased by the same amount.



.. _cancelReplaceTokens:

cancelReplaceTokens
-------------------

Cancels a replace: decrease the old-vault's to-be-redeemed tokens, and the new-vault's to-be-issued tokens. If one or both of the vaults has been liquidated, the change is forwarded to the liquidation vault.

Specification
.............

*Function Signature*

``cancelReplaceTokens(oldVault, newVault, tokens)``

*Parameters*

* ``oldVault``: Account identifier of the vault to be replaced.
* ``newVault``: Account identifier of the vault accepting the replace request.
* ``tokens``: The amount of interBTC replaced.

*Preconditions*

* The BTC Parachain status in the :ref:`security` component MUST NOT be set to ``SHUTDOWN: 2``.
* A vault with id ``oldVault`` MUST be registered.
* A vault with id ``newVault`` MUST be registered.
* If ``oldVault`` is *not* liquidated, its ``toBeRedeemedTokens`` MUST be greater than or equal to ``tokens``.
* If ``oldVault`` *is* liquidated, the liquidation vault's ``toBeRedeemedTokens`` MUST be greater than or equal to ``tokens``.
* If ``newVault`` is *not* liquidated, its ``toBeIssuedTokens`` MUST be greater than or equal to ``tokens``.
* If ``newVault`` *is* liquidated, the liquidation vault's ``toBeIssuedTokens`` MUST be greater than or equal to ``tokens``.

*Postconditions*

* If ``oldVault`` is *not* liquidated, its ``toBeRedeemedTokens`` MUST be decreased by ``tokens``.
* If ``oldVault`` *is* liquidated, the liquidation vault's ``toBeRedeemedTokens`` MUST be decreased by ``tokens``.
* If ``newVault`` is *not* liquidated, its ``toBeIssuedTokens`` MUST be decreased by ``tokens``.
* If ``newVault`` *is* liquidated, the liquidation vault's  ``toBeIssuedTokens`` MUST be decreased by ``tokens``.


.. _liquidateVault:

liquidateVault
--------------

Liquidates a vault, transferring token balances to the ``LiquidationVault``, as well as collateral.

Specification
.............

*Function Signature*

``liquidateVault(vault)``

*Parameters*

* ``vault``: Account identifier of the vault to be liquidated.


*Events*

* :ref:`liquidateVaultEvent`

*Preconditions*

*Postconditions*

* Function :ref:`reward_withdrawStake` MUST complete successfully - parameterized by ``vault`` and ``vault.issuedTokens``.
* The liquidation vault's ``issuedTokens``, ``toBeIssuedTokens`` and ``toBeRedeemedTokens`` MUST be increased by the respective amounts in the vault.
* The vault's ``issuedTokens`` and ``toBeIssuedTokens`` MUST be set to 0.
* Collateral MUST be moved from the vault to the liquidation vault: an amount of ``confiscatedCollateral - confiscatedCollateral * (toBeRedeemedTokens / (toBeIssuedTokens + issuedTokens))`` is moved, where ``confiscatedCollateral`` is the minimum of the vault's ``backingCollateral`` (as calculated via :ref:`computeStakeAtIndex`) and ``SecureCollateralThreshold`` times the equivalent worth of the amount of tokens it is backing.

.. note:: If a vault successfully executes a replace after having been liquidated, it receives some of its confiscated collateral back.

.. _getMaxNominationRatio:

getMaxNominationRatio
----------------------

Returns the nomination ratio, denoting the maximum amount of collateral that can be nominated to a particular Vault.

- ``MaxNominationRatio = (SecureCollateralThreshold / PremiumRedeemThreshold) - 1)``

*Example*

- ``SecureCollateralThreshold = 1.5 (150%)``
- ``PremiumRedeemThreshold = 1.2 (120%)``
- ``MaxNominationRatio = (1.5 / 1.2) - 1 = 0.25 (25%)``

In this example, a Vault with 10 DOT locked as collateral can only receive 2.5 DOT through nomination.

Events
~~~~~~

.. _registerVaultEvent:

RegisterVault
-------------

Emit an event stating that a new vault (``vault``) was registered and provide information on the Vault’s collateral (``collateral``).

*Event Signature*

``RegisterVault(vault, collateral)``

*Parameters*

* ``vault``: The account of the vault to be registered.
* ``collateral``: to-be-locked collateral in DOT.

*Functions*

* :ref:`registerVault`

.. _depositCollateralEvent:

DepositCollateral
------------------------

Emit an event stating how much new (``newCollateral``), total collateral (``totalCollateral``) and freely available collateral (``freeCollateral``) the vault calling this function has locked.

*Event Signature*

``DepositCollateral(vault, newCollateral, totalCollateral, freeCollateral)``

*Parameters*

* ``vault``: The account of the vault locking collateral.
* ``newCollateral``: to-be-locked collateral in DOT.
* ``totalCollateral``: total collateral in DOT.
* ``freeCollateral``: collateral not "occupied" with interBTC in DOT.

*Functions*

* :ref:`depositCollateral`

.. _withdrawCollateralEvent:

WithdrawCollateral
------------------

Emit emit an event stating how much collateral was withdrawn by the vault and total collateral a vault has left.

*Event Signature*

``WithdrawCollateral(vault, withdrawAmount, totalCollateral)``

*Parameters*

* ``vault``: The account of the vault locking collateral.
* ``withdrawAmount``: To-be-withdrawn collateral in DOT.
* ``totalCollateral``: total collateral in DOT.

*Functions*

* :ref:`withdrawCollateral`

.. _registerAddressEvent:

RegisterAddress
---------------

Emit an event stating that a vault (``vault``) registered a new address (``address``).

*Event Signature*

``RegisterAddress(vault, address)``

*Parameters*

* ``vault``: The account of the vault to be registered.
* ``address``: The added address

*Functions*

* :ref:`registerAddress`

.. _updatePublicKeyEvent:

UpdatePublicKey
---------------

Emit an event stating that a vault (``vault``) registered a new address (``address``).

*Event Signature*

``UpdatePublicKey(vault, publicKey)``

*Parameters*

* ``vault``: the account of the vault.
* ``publicKey``: the new BTC public key of the vault.

*Functions*

* :ref:`updatePublicKey`

.. _increaseToBeIssuedTokensEvent:

IncreaseToBeIssuedTokens
------------------------

Emit 

*Event Signature*

``IncreaseToBeIssuedTokens(vaultId, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC to be locked.


*Functions*

* :ref:`tryIncreaseToBeIssuedTokens`

.. _decreaseToBeIssuedTokensEvent:

DecreaseToBeIssuedTokens
------------------------

Emit 

*Event Signature*

``DecreaseToBeIssuedTokens(vaultId, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC to be unreserved.


*Functions*

* :ref:`decreaseToBeIssuedTokens`

.. _issueTokensEvent:

IssueTokens
-----------

Emit an event when an issue request is executed.

*Event Signature*

``IssueTokens(vault, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC that were just issued.

*Functions*

* :ref:`issueTokens`

.. _increaseToBeRedeemedTokensEvent:

IncreaseToBeRedeemedTokens
--------------------------

Emit an event when a redeem request is requested.

*Event Signature*

``IncreaseToBeRedeemedTokens(vault, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC to be redeemed.

*Functions*

* :ref:`tryIncreaseToBeRedeemedTokens`

.. _decreaseToBeRedeemedTokensEvent:

DecreaseToBeRedeemedTokens
--------------------------

Emit an event when a replace request cannot be completed because the vault has too little tokens committed.

*Event Signature*

``DecreaseToBeRedeemedTokens(vault, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC not to be redeemed.

*Functions*

* :ref:`decreaseToBeRedeemedTokens`

.. _increaseToBeReplacedTokensEvent:

IncreaseToBeReplacedTokens
--------------------------

Emit an event when the ``toBeReplacedTokens`` is increased.

*Event Signature*

``IncreaseToBeReplacedTokens(vault, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC to be replaced.

*Functions*

* :ref:`increaseToBeReplacedTokens`

.. _decreaseToBeReplacedTokensEvent:

DecreaseToBeReplacedTokens
--------------------------

Emit an event when the ``toBeReplacedTokens`` is decreased.

*Event Signature*

``DecreaseToBeReplacedTokens(vault, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC not to be replaced.

*Functions*

* :ref:`decreaseToBeReplacedTokens`

.. _decreaseTokensEvent:

DecreaseTokens
--------------

Emit an event if a redeem request cannot be fulfilled.

*Event Signature*

``DecreaseTokens(vault, user, tokens, collateral)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``user``: The BTC Parachain address of the user that made the redeem request.
* ``tokens``: The amount of interBTC that were not redeemed.
* ``collateral``: The amount of collateral assigned to this request.

*Functions*

* :ref:`decreaseTokens`

.. _redeemTokensEvent:

RedeemTokens
------------

Emit an event when a redeem request successfully completes.

*Event Signature*

``RedeemTokens(vault, tokens)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC redeemed.

*Functions*

* :ref:`redeemTokens`

.. _redeemTokensPremiumEvent:

RedeemTokensPremium
-------------------

Emit an event when a user is executing a redeem request that includes a premium.

*Event Signature*

``RedeemTokensPremium(vault, tokens, premiumDOT, redeemer)``

*Parameters*

* ``vault``: The BTC Parachain address of the Vault.
* ``tokens``: The amount of interBTC redeemed.
* ``premiumDOT``: The amount of DOT to be paid to the user as a premium using the Vault's released collateral.
* ``redeemer``: The user that redeems at a premium.

*Functions*

* :ref:`redeemTokens`

.. _redeemTokensLiquidationEvent:

RedeemTokensLiquidation
-----------------------

Emit an event when a redeem is executed under the ``LIQUIDATION`` status.

*Event Signature*

``RedeemTokensLiquidation(redeemer, redeemDOTinBTC)``

*Parameters*

* ``redeemer`` : The account of the user redeeming interBTC.
* ``redeemDOTinBTC``: The amount of interBTC to be redeemed in DOT with the ``LiquidationVault``, denominated in BTC.

*Functions*

* :ref:`redeemTokensLiquidation`

.. _redeemTokensLiquidatedVaultEvent:

RedeemTokensLiquidatedVault
---------------------------

Emit an event when a redeem is executed on a liquidated vault.

*Event Signature*

``RedeemTokensLiquidation(redeemer, tokens, unlockedCollateral)``

*Parameters*

* ``redeemer`` : The account of the user redeeming interBTC.
* ``tokens``: The amount of interBTC that have been refeemed.
* ``unlockedCollateral``: The amount of collateral that has been unlocked for the vault for this redeem.


*Functions*

* :ref:`redeemTokens`

.. _replaceTokensEvent:

ReplaceTokens
-------------

Emit an event when a replace requests is successfully executed.

*Event Signature*

``ReplaceTokens(oldVault, newVault, tokens, collateral)``

*Parameters*

* ``oldVault``: Account identifier of the vault to be replaced.
* ``newVault``: Account identifier of the vault accepting the replace request.
* ``tokens``: The amount of interBTC replaced.
* ``collateral``: The collateral provided by the new vault. 

*Functions*

* :ref:`replaceTokens`

.. _liquidateVaultEvent:

LiquidateVault
--------------

Emit an event indicating that the vault with ``vault`` account identifier has been liquidated.

*Event Signature*

``LiquidateVault(vault)``

*Parameters*

* ``vault``: Account identifier of the vault to be liquidated.

*Functions*

* :ref:`liquidateVault`


Error Codes
~~~~~~~~~~~

``InsufficientVaultCollateralAmount``

* **Message**: "The provided collateral was insufficient - it must be above ``MinimumCollateralVault``."
* **Cause**: The vault provided too little collateral, i.e. below the MinimumCollateralVault limit.

``VaultNotFound``

* **Message**: "The specified vault does not exist."
* **Cause**: vault could not be found in ``Vaults`` mapping.

``ERR_INSUFFICIENT_FREE_COLLATERAL``

* **Message**: "Not enough free collateral available."
* **Cause**: The vault is trying to withdraw more collateral than is currently free. 

``ERR_EXCEEDING_VAULT_LIMIT``

* **Message**: "Issue request exceeds vault collateral limit."
* **Cause**: The collateral provided by the vault combined with the exchange rate forms an upper limit on how much interBTC can be issued. The requested amount exceeds this limit.

``ERR_INSUFFICIENT_TOKENS_COMMITTED``

* **Message**: "The requested amount of ``tokens`` exceeds the amount available to vault."
* **Cause**: A user requests a redeem with an amount exceeding the vault's tokens, or the vault is requesting replacement for more tokens than it has available.

``ERR_VAULT_BANNED``

* **Message**: "Action not allowed on banned vault."
* **Cause**: An illegal operation is attempted on a banned vault, e.g. an issue or redeem request.

``ERR_ALREADY_REGISTERED``

* **Message**: "A vault with the given accountId is already registered."
* **Cause**: A vault tries to register a vault that is already registered.

``ERR_RESERVED_DEPOSIT_ADDRESS``

* **Message**: "Deposit address is already registered."
* **Cause**: A vault tries to register a deposit address that is already in the system.

``ERR_VAULT_NOT_BELOW_LIQUIDATION_THRESHOLD``

* **Message**: "Attempted to liquidate a vault that is not undercollateralized."
* **Cause**: A vault has been reported for being undercollateralized, but at the moment of execution, it is no longer undercollateralized.

``ERR_INVALID_PUBLIC_KEY``

* **Message**: "Deposit address could not be generated with the given public key."
* **Cause**: An error occurred while attempting to generate a new deposit address for an issue request.

.. note:: These are the errors defined in this pallet. It is possible that functions in this pallet return errors defined in other pallets.