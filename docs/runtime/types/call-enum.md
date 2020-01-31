---
title: "Call Enum"
---
In Substrate, the `Call` enum lists the dispatchable functions exposed by your runtime modules. 

Each module will have its own `Call` enum, which contains the function names and parameters for that module. Then when constructing the runtime, an _outer_ `Call` enum is generated as an aggregate of each module's specific `Call`, which references these individual `Call` types from a per-module enumeration.

## Module Call Enum

At the individual module level, a `Call` enum is generated in the `decl_module!` macro. For example, here is the `decl_module!` macro defined in the SRML Sudo module (`srml-sudo`):

```rust
decl_module! {
    // Simple declaration of the `Module` type. Lets the macro know what its working on.
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        fn deposit_event<T>() = default;
        /// Authenticates the sudo key and dispatches a function call with `Root` origin
        ///
        /// The dispatch origin for this call must be _Signed_.
        fn sudo(origin, proposal: Box<T::Proposal>) {...}
        /// Authenticates the current sudo key and sets the given AccountId as the new sudo key
        ///
        /// The dispatch origin for this call must be _Signed_.
        fn set_key(origin, new: <T::Lookup as StaticLookup>::Source) {...}
    }
}
```

When the `decl_module!` macro expands, the following `Call` enum is generated:

```rust
pub enum Call<T: Trait> {
    sudo(Box<T::Proposal>),
    set_key(<T::Lookup as StaticLookup>::Source),
}
```

Here you can see that it has enumerated the list of dispatchable functions available in the module and the parameters needed to call them. `origin` is excluded as it is implicitly required for all dispatchable function calls.

> Note that `deposit_event` did not receive an entry in the `Call` enum because it is not really a dispatchable function, and [the `decl_module!` macro handles that](runtime/macros/decl_module.md#deposit_event).

## Runtime _Outer_ Call Enum

A `Call` enum is generated for each module included in your runtime. This enum is then passed to the `construct_runtime!` macro to generate an _outer_ `Call` enum which lists all of your runtime modules and references their individual `Call` objects.

For example, in the default Substrate `node-template` runtime, we find the following declaration:

```rust
construct_runtime!(
	pub enum Runtime with Log(InternalLog: DigestItem<Hash, AuthorityId, AuthoritySignature>) where
		Block = Block,
		NodeBlock = opaque::Block,
		UncheckedExtrinsic = UncheckedExtrinsic
	{
		System: system::{default, Log(ChangesTrieRoot)},
		Timestamp: timestamp::{Module, Call, Storage, Config<T>, Inherent},
		Consensus: consensus::{Module, Call, Storage, Config<T>, Log(AuthoritiesChange), Inherent},
		Aura: aura::{Module},
		Indices: indices,
		Balances: balances,
		Sudo: sudo,
		TemplateModule: template::{Module, Call, Storage, Event<T>},
	}
);
```

> Note that if you do not include any types with your module declaration, it uses the [default set](runtime/macros/construct_runtime.md#no-types-or-default), which includes `Call`.

This will then generate the following _outer_ `Call` enum:

```rust
pub enum Call {
    Timestamp(::srml_support::dispatch::CallableCallFor<Timestamp>),
    Consensus(::srml_support::dispatch::CallableCallFor<Consensus>),
    Indices(::srml_support::dispatch::CallableCallFor<Indices>),
    Balances(::srml_support::dispatch::CallableCallFor<Balances>),
    Sudo(::srml_support::dispatch::CallableCallFor<Sudo>),
    TemplateModule(::srml_support::dispatch::CallableCallFor<TemplateModule>),
}
```

The _outer_ `Call` enum has collected all the modules from the `construct_runtime!` macro that expose a `Call` enum, since only such modules expose dispatchable functions. Hence, it defines the full set of exposed dispatchable functions in your blockchain.

## Exposing Call via the Metadata Endpoint

Finally, when you run a Substrate node, it will automatically generate a `getMetadata` endpoint containing the objects generated by your runtime.

For example, the SRML Sudo module (`srml-sudo`) lists the following `calls` when the response of the `getMetadata` API is converted to JSON.

```json
"modules": [
    {
        "name": "sudo",
        "prefix": "Sudo",
        "calls": [
            {
                "name": "sudo",
                "args": [
                    {
                        "name": "proposal",
                        "type": "Proposal"
                    }
                ],
                "docs": [
                    "Authenticates the sudo key and dispatches a function call with `Root` origin",
                    "",
                    "The dispatch origin for this call must be _Signed_."
                ]
            },
            {
                "name": "set_key",
                "args": [
                    {
                        "name": "new",
                        "type": "Address"
                    }
                ],
                "docs": [
                    "Authenticates the current sudo key and sets the given AccountId as the new sudo key",
                    "",
                    "The dispatch origin for this call must be _Signed_."
                ]
            }
        ],
        ...
    },
    ...
]
```

This may then be used to generate JavaScript functions which allow you to dispatch calls to the runtime.