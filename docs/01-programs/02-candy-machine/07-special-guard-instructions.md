---
description: "Explains how to execute guard-specific instructions."
---

import { Accordion, AccordionItem } from '/src/accordion.jsx';

# Special Guard Instructions

## Introduction

As we’ve seen on the previous pages, guards are a powerful way to customize the minting process of your Candy Machines. But did you know guards can even provide their own custom instructions?

## The Route Instruction

The Candy Guard program ships with a special instruction called **the “Route” instruction**.

This instruction allows us to **select a specific guard** from our Candy Machine and **run a custom instruction** that is specific to this guard. We call it the “Route” instruction because it will route our request to the selected guard.

This feature makes guards even more powerful as they can ship with their own program logic. It enables guards to:

- Decouple the verification process from the minting process for heavy operations.
- Provide custom features that would otherwise require the deployment of a custom program.

To call a route instruction, we must specify which guard we want to route that instruction to as well as **provide the route settings it expects**. Note that if we try to execute the “route” instruction by selecting a guard that does not support it, the transaction will fail.

Since there can only be one “route” instruction per registered guard on a Candy Guard program, it is common to provide a **Path** attribute in the route settings to distinguish between multiple features offered by the same guard.

For instance, a guard adding support for Frozen NFTs — that can only be thawed once minting is over — could use their route instruction to initialize the treasury escrow account as well as allow anyone to thaw a minted NFT under the right conditions. We could distinguish these two features by using a **Path** attribute equal to “init” for the former and “thaw” for the latter.

You will find a detailed explanation of the route instruction of each guard that supports it and their underlying paths [on their respective pages](/programs/candy-machine/available-guards).

Let’s take a minute to illustrate how the route instruction works by providing an example. The [**Allow List**](/programs/candy-machine/available-guards/allow-list) guard, for instance, supports the route instruction in order to verify that the minting wallet is part of the preconfigured list of wallets.

It does that using [Merkle Trees](https://en.m.wikipedia.org/wiki/Merkle_tree) which means we need to create a hash of the entire list of allowed wallets and store that hash — known as the **Merkle Root** — on the guard settings. For a wallet to prove it is on the allowed list, it must provide a list of hashes — known as the **Merkle Proof** — that allows the program to compute the Merkle Root and ensure it matches the guard’s settings.

Therefore, the Allow List guard **uses its route instruction to verify the Merkle Proof of a given wallet** and, if successful, creates a small PDA account on the blockchain that acts as verification proof for the mint instruction.

![CandyMachinesV3-SpecialGuardInstructions1.png](/assets/candy-machine-v3/CandyMachinesV3-SpecialGuardInstructions1.png#radius)

So why can’t we just verify the Merkle Proof directly within the mint instruction? That’s simply because, for big allow lists, Merkle Proofs can end up being pretty lengthy. After a certain size, it becomes impossible to include it within the mint transaction that already contains a decent amount of instructions. By separating the validation process from the minting process, we make it possible for allow lists to be as big as we need them to be.

<Accordion>
<AccordionItem title="JavaScript — Umi library (recommended)" open={true}>
<div className="accordion-item-padding">

You may use the `route` function to call the route instruction of a guard using the Umi library. You will need to pass the guard’s name via the `guard` attribute and its route settings via the `routeArgs` attribute.

Here is an example using the Allow List guard which validates the wallet’s Merkle Proof before minting.

```ts
import {
  create,
  route,
  getMerkleProof,
  getMerkleRoot,
} from "@metaplex-foundation/mpl-candy-machine";

// Prepare the allow list.
// Let's assume the first wallet on the list is the Metaplex identity.
const allowList = [
  "GjwcWFQYzemBtpUoN5fMAP2FZviTtMRWCmrppGuTthJS",
  "2vjCrmEFiN9CLLhiqy8u1JPh48av8Zpzp3kNkdTtirYG",
  "AT8nPwujHAD14cLojTcB1qdBzA1VXnT6LVGuUd6Y73Cy",
];
const merkleRoot = getMerkleRoot(allowList);

// Create a Candy Machine with an Allow List guard.
await create(umi, {
  // ...
  guards: {
    allowList: some({ merkleRoot }),
  },
}).sendAndConfirm(umi);

// If we try to mint now, it will fail because
// we did not verify our Merkle Proof.

// Verify the Merkle Proof using the route instruction.
await route(umi, {
  candyMachine: candyMachine.publicKey,
  guard: "allowList",
  routeArgs: {
    path: "proof",
    merkleRoot,
    merkleProof: getMerkleProof(
      allowList,
      "GjwcWFQYzemBtpUoN5fMAP2FZviTtMRWCmrppGuTthJS"
    ),
  },
}).sendAndConfirm(umi);

// If we try to mint now, it will succeed.
```

API References: [route](https://mpl-candy-machine-js-docs.vercel.app/functions/route.html), [DefaultGuardSetRouteArgs](https://mpl-candy-machine-js-docs.vercel.app/types/DefaultGuardSetRouteArgs.html)

</div>
</AccordionItem>
<AccordionItem title="JavaScript — SDK">
<div className="accordion-item-padding">

You may use the `callGuardRoute` operation to call the route instruction of a guard using the JS SDK. You must pass the guard’s name via the `guard` attribute and the route settings via the `settings` attribute.
Here is an example using the Allow List guard which validates the wallet’s Merkle Proof before minting.

```tsx
import { getMerkleProof, getMerkleRoot } from "@metaplex-foundation/js";

// Prepare the allow list.
// Let's assume the first wallet on the list is the Metaplex identity.
const allowList = [
  "GjwcWFQYzemBtpUoN5fMAP2FZviTtMRWCmrppGuTthJS",
  "2vjCrmEFiN9CLLhiqy8u1JPh48av8Zpzp3kNkdTtirYG",
  "AT8nPwujHAD14cLojTcB1qdBzA1VXnT6LVGuUd6Y73Cy",
];

// Create a Candy Machine with an Allow List guard.
const { candyMachine } = await metaplex.candyMachines().create({
  // ...
  guards: {
    allowList: { merkleRoot: getMerkleRoot(allowList) },
  },
});

// If we try to mint now, it will fail because
// we did not verify our Merkle Proof.

// Verify the Merkle Proof using the route instruction.
await metaplex.candyMachines().callGuardRoute({
  candyMachine,
  guard: "allowList",
  settings: {
    path: "proof",
    merkleProof: getMerkleProof(
      allowList,
      "GjwcWFQYzemBtpUoN5fMAP2FZviTtMRWCmrppGuTthJS"
    ),
  },
});

// If we try to mint now, it will succeed.
```

API References: [Operation](https://metaplex-foundation.github.io/js/classes/js.CandyMachineClient.html#callGuardRoute), [Input](https://metaplex-foundation.github.io/js/types/js.CallCandyGuardRouteInput.html), [Output](https://metaplex-foundation.github.io/js/types/js.CallCandyGuardRouteOutput.html), [Transaction Builder](https://metaplex-foundation.github.io/js/classes/js.CandyMachineBuildersClient.html#callGuardRoute), [Default Route Settings](https://metaplex-foundation.github.io/js/types/js.DefaultCandyGuardRouteSettings.html).

</div>
</AccordionItem>
</Accordion>

## Route Instruction With Groups

When calling the route instruction whilst using guard groups, it is important to **specify the group label** of the guard we wish to select. This is because we may have multiple guards of the same type across different groups and the program needs to know which one it should use for the route instruction.

For instance, say we had an **Allow List** of handpicked VIP wallets in one group and another **Allow List** for the winners of a raffle in another group. Then saying we want to verify the Merkle Proof for the Allow List guard is not enough, we also need to know for which group we should perform that verification.

<Accordion>
<AccordionItem title="JavaScript — Umi library (recommended)" open={true}>
<div className="accordion-item-padding">

When using groups, the `route` function of the Umi library accepts an additional `group` attribute of type `Option<string>` which must be set to the label of the group we want to select.

```ts
import {
  create,
  route,
  getMerkleProof,
  getMerkleRoot,
} from "@metaplex-foundation/mpl-candy-machine";
import { base58PublicKey, some } from "@metaplex-foundation/umi";

// Prepare the allow lists.
const allowListA = [...];
const allowListB = [...];

// Create a Candy Machine with two Allow List guards.
await create(umi, {
  // ...
  groups: [
    {
      label: "listA",
      guards: {
        allowList: some({ merkleRoot: getMerkleRoot(allowListA) }),
      },
    },
    {
      label: "listB",
      guards: {
        allowList: some({ merkleRoot: getMerkleRoot(allowListB) }),
      },
    },
  ],
}).sendAndConfirm(umi);

// Verify the Merkle Proof by specifying which group to select.
await route(umi, {
  candyMachine: candyMachine.publicKey,
  guard: 'allowList',
  group: some('listA'), // <- We are veryfing using "allowListA".
  routeArgs: {
    path: 'proof',
    merkleRoot: getMerkleRoot(allowListA),
    merkleProof: getMerkleProof(
      allowListA,
      base58PublicKey(umi.identity),
    ),
  },
}).sendAndConfirm(umi);
```

API References: [route](https://mpl-candy-machine-js-docs.vercel.app/functions/route.html), [DefaultGuardSetRouteArgs](https://mpl-candy-machine-js-docs.vercel.app/types/DefaultGuardSetRouteArgs.html)

</div>
</AccordionItem>
<AccordionItem title="JavaScript — SDK">
<div className="accordion-item-padding">

When using groups in the JS SDK, the `callGuardRoute` operation accepts an additional `group` attribute which must be set to the label of the group we want to select.

```tsx
import { getMerkleProof, getMerkleRoot } from '@metaplex-foundation/js';

// Prepare the allow lists.
const allowListA = [...];
const allowListB = [...];

// Create a Candy Machine with two Allow List guards.
const { candyMachine } = await metaplex.candyMachines().create({
  // ...
  groups: [
    {
      label: "listA",
      guards: {
        allowList: { merkleRoot: getMerkleRoot(allowListA) },
      },
    },
    {
      label: "listB",
      guards: {
        allowList: { merkleRoot: getMerkleRoot(allowListB) },
      },
    },
  ],
});

// Verify the Merkle Proof by specifying which group to select.
await metaplex.candyMachines().callGuardRoute({
  candyMachine,
  guard: 'allowList',
  group: 'listA', // <- We are veryfing using "allowListA".
  settings: {
    path: 'proof',
    merkleProof: getMerkleProof(
      allowListA,
      metaplex.identity().publicKey.toBase58(),
    ),
  },
});
```

API References: [Operation](https://metaplex-foundation.github.io/js/classes/js.CandyMachineClient.html#callGuardRoute), [Input](https://metaplex-foundation.github.io/js/types/js.CallCandyGuardRouteInput.html), [Output](https://metaplex-foundation.github.io/js/types/js.CallCandyGuardRouteOutput.html), [Transaction Builder](https://metaplex-foundation.github.io/js/classes/js.CandyMachineBuildersClient.html#callGuardRoute), [Default Route Settings](https://metaplex-foundation.github.io/js/types/js.DefaultCandyGuardRouteSettings.html).

</div>
</AccordionItem>
</Accordion>

## Conclusion

The route instruction makes guards even more powerful by allowing them to ship with their own custom program logic. Check out the dedicated pages of [all available guards](/programs/candy-machine/available-guards) to see the full feature set of each guard.

Now that we know everything there is to know about setting up Candy Machines and their guards, it’s about time we talk about minting. See you on [the next page](/programs/candy-machine/minting)!
