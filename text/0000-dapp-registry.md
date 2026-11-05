# Dapp Registry Standard

## Summary

[summary]: #summary

CEP PR: [casper-network/ceps#0000](https://github.com/casper-network/ceps/pull/0000)

A dapp on Casper is usually several contracts, which the chain sees as unrelated package hashes.
This CEP defines a **dapp registry**, a contract that describes a dapp (name, description, website
URL, icon URL) and lists the contracts that belong to it, and a **dapp contract** interface through
which every member points back at its registry.

Membership is verified from both sides: the registry adds a contract only after calling the
contract's `get_dapp_registry` entry point and receiving its own address back. A contract belongs
to at most one dapp. A member registered as a **registrar** may add other contracts, so a dapp
that creates contracts through a factory registers them at runtime without a signature from the
owner. All data lives in one dictionary under the `__dapp` named key, every item a whole value,
so explorers, wallets and indexers read it from global state without calling the contracts.

The standard fixes the entry points, the storage layout, the events and the error codes, and
leaves the authorization policy to the implementation. A reference implementation ships in
[Odra 2.10](https://github.com/odradev/odra/tree/release/2.10.0/modules/src/dapp).

## Motivation

[motivation]: #motivation

[CEP-96](0096-contract-metadata.md) lets one contract describe itself, but nothing on chain says
that a token, a vault and twenty pools created by a factory are one product. Every tool that wants
a dapp page, a "you are interacting with *Vault* of *My Dapp*" line in a signing dialog, or a
directory of dapps keeps its own off-chain list.

A one-sided, self-declared list would not do. A phishing dapp could list the real CSPR token among
"its" contracts, and a scam token could claim to be part of a known dapp. The standard therefore
requires a link both parties agreed to, and checks it when the link is made.

Many dapps create contracts at runtime through factories ([CEP-86](0086-factory-pattern.md)). The
owner cannot sign a registration for each of them, so the registry can delegate the right to add
contracts to selected members.

Finally, tools should answer "which dapp does this contract belong to?" from global state, the way
they read the `balances` of a CEP-18 token, so the storage layout is part of the standard.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The standard uses four terms:

- A **dapp** is a product made of one or more contracts.
- A **registry** is the contract that represents the dapp. It stores the dapp metadata and the
  list of its contracts, and it is itself a member of the dapp.
- A **member** (a *dapp contract*) is a contract that belongs to a dapp. It stores the address of
  its registry and exposes it through `get_dapp_registry`.
- A **registrar** is a member that may add other members, typically a contract that creates
  contracts through a factory.

The examples use the reference implementation: `OwnedDappRegistry` is a ready-to-deploy registry
owned by its deployer, `DappContractBase` the module a member composes.

### Describing the dapp and adding a member

```rust
let mut registry = OwnedDappRegistry::deploy(&env, NoArgs);
registry.set_dapp_metadata(DappMetadata {
    name: "My Dapp".to_string(),
    description: "Lending on Casper".to_string(),
    website_url: "https://mydapp.io".to_string(),
    icon_url: "https://mydapp.io/icon.png".to_string()
});
```

A member stores its registry address, usually in `init`, and exposes `get_dapp_registry`:

```rust
#[odra::module]
pub struct Vault {
    dapp: SubModule<DappContractBase>
}

#[odra::module]
impl Vault {
    pub fn init(&mut self, registry: Address) {
        self.dapp.set_dapp_registry(&registry);
    }

    delegate! {
        to self.dapp {
            fn get_dapp_registry(&self) -> Address;
        }
    }
}
```

The owner adds it. The registry calls `get_dapp_registry` on the vault, checks that the answer is
its own address, and records the vault:

```rust
let vault = Vault::deploy(&env, VaultInitArgs { registry: registry.address() });
registry.add_dapp_contract(&vault.address(), false);
assert_eq!(registry.get_dapp_contracts(), vec![registry.address(), vault.address()]);
```

A contract that points elsewhere is rejected with `DappRegistryMismatch`, and an account with
`NotAContract`.

### Contracts created by a factory

A contract that spawns other contracts is added with `is_registrar = true`. It then passes its
registry address to every contract it creates and adds it:

```rust
/// The spawner must be registered in the dapp as a registrar.
pub fn spawn(&mut self) -> Address {
    let factory = self.factory.get().unwrap_or_revert(self);
    let n = self.spawned.get_or_default();
    let mut factory = DappCounterFactoryContractRef::new(self.env(), factory);
    // The new counter stores the registry address in its `init`.
    let (address, _) =
        factory.new_contract(format!("DappCounter{}", n), self.dapp.register_address());
    self.spawned.set(n + 1);
    // Calls `add_dapp_contract(address, false)` on the registry.
    self.dapp.register(address);
    address
}
```

A spawner that is not a registrar is rejected with `CallerNotDappRegistrar`. The contracts it
spawns are members but not registrars.

An explorer that lands on a counter reads the `registry` item of its `__dapp` dictionary, then
`name`, `icon_url` and `contracts` from the registry's, and shows "part of My Dapp" with the icon
and the other contracts. No contract code runs.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Definitions use Rust syntax but are not Rust specific. A compliant registry exposes the registry
entry points with these names, argument names and CL types, keeps the storage layout, emits the
events and uses the error codes. A compliant member exposes `get_dapp_registry` and keeps the
`registry` item.

### Addresses and types

An *address* is a `Key`: a contract address is the contract **package** hash wrapped in
`Key::Hash`, an account address is `Key::Account`. Registries store package hashes, so upgrading a
member does not change its membership.

```rust
pub struct DappMetadata {
    pub name: String,
    pub description: String,
    pub website_url: String,
    pub icon_url: String
}
```

`DappMetadata` travels as one argument or return value of CL type `Any`, whose bytes are the
`bytesrepr` serialization of the four fields in order. The contract schema of the registry
describes the struct.

### Deployment forms

A registry is any contract that exposes the registry interface and keeps the storage layout. It
may be a dedicated contract, such as `OwnedDappRegistry`, or an existing contract of the dapp,
such as its token, that composes `DappRegistryBase` next to its own logic. The storage layout is
the same in both cases: the items live under the `__dapp` named key of the hosting contract,
apart from its other keys, and the entry point names carry the `dapp` prefix so they do not
collide with the host's own. A contract is a registry or a member, never both, since each role
answers `get_dapp_registry` differently.

### Registry interface

```rust
pub trait DappRegistry {
    /// Replaces all four metadata fields. Emits `DappMetadataChanged`.
    fn set_dapp_metadata(&mut self, metadata: DappMetadata);

    /// Returns the metadata; every field is an empty string until set.
    fn get_dapp_metadata(&self) -> DappMetadata;

    /// Adds a contract, as a registrar if `is_registrar`. See the steps below.
    fn add_dapp_contract(&mut self, dapp_contract: Address, is_registrar: bool);

    /// Removes a contract and its registrar right. See the steps below.
    fn remove_dapp_contract(&mut self, dapp_contract: Address);

    /// Returns the registry first, then the members in the order they were added.
    fn get_dapp_contracts(&self) -> Vec<Address>;

    /// Returns true if the contract belongs to the dapp. True for the registry itself.
    fn is_dapp_contract(&self, dapp_contract: Address) -> bool;

    /// Returns true if the contract is a registrar. False for the registry itself.
    fn is_dapp_registrar(&self, dapp_contract: Address) -> bool;

    /// The registry is a member of its own dapp: returns its own address.
    fn get_dapp_registry(&self) -> Address;
}
```

`add_dapp_contract(dapp_contract, is_registrar)` must, in this order:

1. Revert with `NotAContract` if `dapp_contract` is not a contract address.
2. Revert with `DappContractAlreadyRegistered` if `dapp_contract` already belongs to the dapp.
   The registry itself counts as registered.
3. Append `dapp_contract` to `contracts`, and to `registrars` if `is_registrar` is true. The
   writes happen before the call of the next step: a re-entrant attempt to add the same contract
   is rejected by step 2, and a revert in step 4 rolls the writes back.
4. Call `get_dapp_registry` on the current enabled version of the `dapp_contract` package and
   revert with `DappRegistryMismatch` unless it returns this registry's address. A contract
   without that entry point makes the host fail the transaction.
5. Emit `DappContractAdded { contract: dapp_contract, is_registrar, registrar: caller }`.

`remove_dapp_contract(dapp_contract)` must:

1. Revert with `CannotRemoveDappRegistry` if `dapp_contract` is the registry itself.
2. Revert with `DappContractNotRegistered` if `dapp_contract` is not in `contracts`.
3. Remove it from `contracts`, keeping the order of the other members, and from `registrars` if
   present. The removed contract still points to the registry, but the registry no longer confirms
   it, so it is not a member.
4. Emit `DappContractRemoved { contract: dapp_contract, registrar: caller }`.

The registrar right of a member cannot be changed in place: remove the member and add it again.

### Member interface

```rust
pub trait DappContract {
    /// Returns the dapp registry the contract belongs to.
    /// Reverts with `DappRegistryNotSet` if no registry has been set.
    fn get_dapp_registry(&self) -> Address;
}
```

`get_dapp_registry` must be an open entry point, since the registry calls it. Setting the
registry is not part of the public interface: a member sets it at installation, or through an
access-controlled entry point of its own. Setting it reverts with `NotAContract` for an account
address and emits `DappRegistryChanged { previous_registry, new_registry }`, including the first
time, with `previous_registry = None`. Re-pointing a member does not remove it from the old
registry, but the two sides no longer agree, so it is not a member of the old dapp.

A registry answers `get_dapp_registry` with its own address, so any contract of a dapp, including
the registry, can be asked which dapp it belongs to.

### Verification rule

A contract `C` is a member of the dapp of registry `R` if and only if:

- `R.is_dapp_contract(C)` is true (`C` is in `R`'s `contracts` item, or `C == R`), and
- `C.get_dapp_registry()` returns `R` (`C`'s `registry` item holds `R`).

The registry enforces the second condition when it adds a contract. Tools must check both when
they display a membership, since a member may have been re-pointed after it was added.

### Registrars

A registrar is a member that may call `add_dapp_contract`. The standard fixes what the flag means,
where it is stored (`registrars`) and how it is read (`is_dapp_registrar`). Who may add a
registrar, and what a registrar may add, is decided by the authorization policy. In the reference
implementation a registrar may add contracts that are not registrars, so a compromised registrar
cannot delegate further.

A registrar registers a contract in two steps of its own code: it creates the contract and passes
it the registry address, then it calls `add_dapp_contract(new_contract, false)`. The registry sees
the registrar as the caller and records it in the `registrar` field of `DappContractAdded`.

### Authorization

`set_dapp_metadata`, `add_dapp_contract` and `remove_dapp_contract` change the dapp. An
implementation must restrict them. The standard does not fix the policy, because dapps are
governed in different ways (one key, a multisig, roles, a DAO) and the policy is invisible to
tools. The read-only entry points are open.

The reference implementation, `OwnedDappRegistry`, applies this policy:

| Entry point            | Owner | Registrar                                   | Anyone else |
|------------------------|-------|---------------------------------------------|-------------|
| `set_dapp_metadata`    | yes   | `CallerNotTheOwner`                         | `CallerNotTheOwner` |
| `add_dapp_contract`    | yes   | yes, with `is_registrar = false` only; otherwise `CallerNotTheOwner` | `CallerNotTheOwner` for an account or if `is_registrar`; otherwise `CallerNotDappRegistrar` |
| `remove_dapp_contract` | yes   | `CallerNotTheOwner`                         | `CallerNotTheOwner` |

### Storage layout

All data lives in one dictionary whose seed `URef` is stored under the `__dapp` named key of the
contract. Item keys are plain UTF-8 strings, not hashed or encoded, so they can be typed into an
RPC query. Each item holds a whole value.

Registry:

| Item key      | CL type     | Value |
|---------------|-------------|-------|
| `name`        | `String`    | The name of the dapp. |
| `description` | `String`    | A short description of the dapp. |
| `website_url` | `String`    | The URL of the dapp's website. |
| `icon_url`    | `String`    | The URL of the dapp's icon. |
| `contracts`   | `List<Key>` | The members in the order they were added. The registry itself is **not** listed. |
| `registrars`  | `List<Key>` | The members that may add other contracts, in the order they were added. |

Member:

| Item key   | CL type | Value |
|------------|---------|-------|
| `registry` | `Key`   | The registry the contract belongs to. |

An absent item means "not set": an empty string for a metadata field, an empty list for
`contracts` and `registrars`, no registry for `registry`. A registry creates the dictionary at
installation, so its `__dapp` named key exists before anything is added. A member creates it on
its first write of `registry`.

The `__dapp` name follows the Casper Event Standard's `__events`, `__events_length` and
`__events_schema`: a reserved, double-underscored named key kept apart from the contract's own
keys. A registry that also implements [CEP-96](0096-contract-metadata.md) keeps `contract_name`
and the other CEP-96 keys next to `__dapp`; the two standards do not overlap.

### Events

Events follow the [Casper Event Standard](https://github.com/make-software/casper-event-standard).
Field types are CL types.

Emitted by a registry:

| Event                 | Fields |
|-----------------------|--------|
| `DappMetadataChanged` | `name: String`, `description: String`, `website_url: String`, `icon_url: String` |
| `DappContractAdded`   | `contract: Key`, `is_registrar: Bool`, `registrar: Key` (the caller: the owner or a registrar) |
| `DappContractRemoved` | `contract: Key`, `registrar: Key` (the caller) |

Emitted by a member:

| Event                 | Fields |
|-----------------------|--------|
| `DappRegistryChanged` | `previous_registry: Option<Key>`, `new_registry: Key` |

`DappMetadataChanged` lists the four fields one by one instead of carrying a `DappMetadata`
value, so each field keeps a plain CL type in the event schema. The schemas are part of the
standard: an upgrade cannot change the schema of an event a contract already emits.

### Error codes

| Error                           | Code  | When |
|---------------------------------|-------|------|
| `NotAContract`                  | 22000 | An account address is added to a dapp, or set as a registry. |
| `DappContractAlreadyRegistered` | 22001 | The contract, or the registry itself, is already in the dapp. |
| `DappContractNotRegistered`     | 22002 | A contract that is not in the dapp is removed. |
| `DappRegistryMismatch`          | 22003 | The contract's `get_dapp_registry` returns a different address. |
| `CallerNotDappRegistrar`        | 22004 | A contract that is not a registrar tries to add a contract. |
| `DappRegistryNotSet`            | 22005 | `get_dapp_registry` is called on a member before its registry is set. |
| `CannotRemoveDappRegistry`      | 22006 | The registry itself is removed. |

Errors are reverted as `ApiError::User` codes. The range 22000–22099 is reserved for this
standard. The authorization layer keeps its own codes; the reference implementation reuses
`CallerNotTheOwner` (20001) from its `Ownable` module.

### Reading a dapp from global state

A tool detects a compliant contract by the `__dapp` named key and reads the items it needs with a
dictionary query:

```
casper-client get-dictionary-item \
    --node-address http://NODE:7777 \
    --state-root-hash STATE_ROOT_HASH \
    --contract-hash hash-REGISTRY_CONTRACT_HASH \
    --dictionary-name __dapp \
    --dictionary-item-key contracts
```

- **From a contract to its dapp:** read its `registry` item, then the registry's `contracts`
  item, and show the registry's `name` and `icon_url` only if the contract is listed or is the
  registry. A signing dialog does the same for the transaction's target.
- **From a dapp to its contracts:** read the registry's `contracts` item, drop every entry whose
  `registry` item does not point back, and prepend the registry.
- **Keeping up to date:** follow the events of the registry and of the members. Membership and
  metadata are mutable, so cache per state root hash.

### Reference implementation

Odra 2.10 ships the standard as `odra_modules::dapp`
([source](https://github.com/odradev/odra/tree/release/2.10.0/modules/src/dapp),
[PR odradev/odra#677](https://github.com/odradev/odra/pull/677),
[tutorial](https://odra.dev/docs/tutorials/dapp-registry)):

- `DappRegistryBase` and `DappContractBase`: storage and logic of a registry and of a member. The
  read-only functions are entry points; the state-changing ones are plain methods with no caller
  check, guarded by the composing contract.
- `OwnedDappRegistry`: `DappRegistryBase` composed with `Ownable`, implementing the policy of the
  authorization table.
- `DappRegistry` and `DappContract`: the traits used to call a registry or a member by address.
- [`examples/src/factory/dapp.rs`](https://github.com/odradev/odra/blob/release/2.10.0/examples/src/factory/dapp.rs):
  a factory-created counter and a spawner registered as a registrar, tested on OdraVM and the
  Casper VM.

## Drawbacks

[drawbacks]: #drawbacks

- **Whole-list items.** Adding or removing a member rewrites the `contracts` list, membership
  checks deserialize it, and the value cannot grow past the host's limit on a stored value. The
  layout suits dapps of tens or hundreds of contracts, not registries of many thousands.
- **A cross-contract call on add.** The registry runs code of the contract it adds. Writing before
  calling keeps re-entrancy harmless, but a malicious contract can still burn the caller's gas, so
  an owner should add only contracts they control.
- **Self-declared metadata.** The standard proves that a set of contracts agreed to form a dapp,
  not that the dapp is who it says it is. Explorers still need their own "verified" process.
- **One dapp per contract.** A contract shared by two products of one team can belong to only one
  registry.
- **Mutable membership.** A removed or re-pointed contract leaves the dapp; cached memberships
  must be re-checked.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- **Two-sided verification.** A list kept only by the registry lets anyone claim foreign
  contracts; a pointer kept only by the member lets anyone attach to a known dapp. Requiring both
  makes a membership a fact both parties signed off on.
- **One dictionary under `__dapp`, not a named key per field.** CEP-96's key per field suits four
  constant values. A registry has lists that change, a member needs one value, and one reserved
  key keeps the contract's named keys clean. Hashed item keys were rejected so that `contracts`
  can be typed into a query.
- **Whole lists, not indexed items.** A `contracts_length` plus `contracts_0..n` layout scales
  further but needs `n + 1` reads in every tool. Dapps are small in practice; pagination is left
  to an extension.
- **Registrars, not a factory as owner.** The owner is the governance key; a spawner is a
  contract. A narrow right (add, but not registrars, and not remove) limits what a bug in it can
  do.
- **The registry is a member of its own dapp.** Tools treat all contracts uniformly, and the
  registry can carry the dapp's CEP-96 metadata.
- **Names prefixed with `dapp`.** The registry and member interfaces are meant to be embedded in
  contracts that already have `name`, `owner` or `transfer`, so every entry point name carries
  the prefix, and the storage items are scoped by the `__dapp` dictionary instead of by name.
- **Authorization left open.** Fixing `Ownable` would exclude dapps governed by roles or a DAO,
  and tools do not depend on the policy.
- **Metadata as one struct, empty string for unset.** The fields change together and rarely.
  `Option` fields as in CEP-96 were considered; tools handle an empty string the same way.
- **Discarded: calling registrars "factories".** The right is about who may add, not how
  contracts are created; a spawner that calls a factory holds it, the factory does not.
- **Discarded: self-registration.** A contract adding itself would let anyone spam a dapp.
- **Discarded: a global dapp directory contract.** It needs an authority to curate it. Per-dapp
  registries need none, and a directory can index them later.

Not doing this leaves every explorer and wallet with its own off-chain grouping, or none.

## Prior art

[prior-art]: #prior-art

- [CEP-96 Contract Metadata](0096-contract-metadata.md) describes one contract with named keys
  that tools read by name. This standard extends the idea to a set of contracts and adds
  verification; the two compose.
- [CEP-86 Factory Pattern](0086-factory-pattern.md) is why registrars exist.
- [CEP-18](0018-token-standard.md) established reading `name`, `symbol` and `balances` from named
  keys and dictionaries instead of calling entry points.
- The [Casper Event Standard](https://github.com/make-software/casper-event-standard) introduced
  the `__events` reserved keys and the event schemas used here.
- Ethereum has no on-chain dapp grouping standard.
  [ERC-1820](https://eips.ethereum.org/EIPS/eip-1820) is a global registry where contracts
  register the interfaces they implement, with a manager that may register on a contract's
  behalf; it inspired the registrar, while its global, self-registered model is what this
  standard avoids. [ERC-173](https://eips.ethereum.org/EIPS/eip-173) and OpenZeppelin's
  `Ownable` are the model for the reference authorization. Ethereum dapp directories (token
  lists, DappRadar, explorer labels) are off-chain and unverifiable from the chain.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- Should `set_dapp_metadata` take four `String` arguments instead of one `Any` value, for clients
  that do not consume contract schemas?
- Should metadata fields be `Option<String>` as in CEP-96, instead of empty strings?
- Should the standard define a paginated variant of `contracts`, or is that a separate extension?
- Should removing a member also clear the member's `registry` item through a callback? Today the
  member keeps pointing to the registry and tools rely on the registry's list.

## Future possibilities

[future-possibilities]: #future-possibilities

- **Dapp directories.** A directory contract, or an off-chain index, that lists registries and
  lets explorers browse dapps, adding curation on top of opt-in registries.
- **Website proof.** A well-known file under `website_url` naming the registry's package hash
  would let a tool confirm that the website claims the dapp.
- **Nested dapps.** A registry cannot be a member of another registry, since it answers
  `get_dapp_registry` with itself. A parent pointer would let a suite of products form a hierarchy.
- **Contract roles.** A per-member label (`token`, `vault`, `pool`) next to `registry` would let
  tools describe the dapp's structure, not only its membership.
- **Multiple dapps per contract.** A list of registries on the member side, if shared contracts
  turn out to be common.
