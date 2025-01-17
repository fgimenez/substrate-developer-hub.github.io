---
title: Building a Custom Front End
---

This tutorial is not really about writing user interfaces, but we do want to
show you how easy it can be with Substrate. If you have made it this far, that
means you _should_ have a brand new blockchain with custom functionality up and
running.

We will give you a custom react component that you can add to your
`substrate-front-end-template` meant for interacting with your node.

## Add Your Custom React Component

In the `substrate-front-end-template` project, edit the `TemplateModule.js`
file in the `/src/` folder:

```
substrate-front-end-template
|
+-- src
|   |
|   +-- index.js
|   |
|   +-- App.js
|   |
|   +-- TemplateModule.js  <-- Edit this file
|   |
|   +-- ...
+-- ...
```

Delete the contents of that file, and instead use the following component.

<div style="max-height: 20em; overflow: auto; margin-bottom: 1em;">

```js
// React and Semantic UI elements.
import React, { useState, useEffect } from 'react';
import { Form, Input, Grid, Message } from 'semantic-ui-react';
// Pre-built Substrate front-end utilities for connecting to a node
// and making a transaction.
import { useSubstrate } from './substrate-lib';
import { TxButton } from './substrate-lib/components';
// Polkadot-JS utilities for hashing data.
import { blake2AsHex } from '@polkadot/util-crypto';

// Our main Proof Of Existence Component which is exported.
export default function ProofOfExistence (props) {
  // Establish an API to talk to our Substrate node.
  const { api } = useSubstrate();
  // Get the "selected user" from the `AccountSelector` component.
  const { accountPair } = props;
  // React hooks for all the state variables we track.
  // Learn more at: https://reactjs.org/docs/hooks-intro.html
  const [status, setStatus] = useState('');
  const [digest, setDigest] = useState('');
  const [owner, setOwner] = useState('');
  const [block, setBlock] = useState(0);

  // Our `FileReader()` which is accessible from our functions below.
  let fileReader;

  // Takes our file, and creates a digest using the Blake2 256 hash function.
  const bufferToDigest = () => {
    // Turns the file content to a hexadecimal representation.
    const content = Array.from(new Uint8Array(fileReader.result))
      .map(b => b.toString(16).padStart(2, '0'))
      .join('');

    const hash = blake2AsHex(content, 256);
    setDigest(hash);
  };

  // Callback function for when a new file is selected.
  const handleFileChosen = file => {
    fileReader = new FileReader();
    fileReader.onloadend = bufferToDigest;
    fileReader.readAsArrayBuffer(file);
  };

  // React hook to update the "Owner" and "Block Number" information for a file.
  useEffect(() => {
    let unsubscribe;

    // Polkadot-JS API query to the `proofs` storage item in our module.
    // This is a subscription, so it will always get the latest value,
    // even if it changes.
    api.query.templateModule
      .proofs(digest, result => {
        // Our storage item returns a tuple, which is represented as an array.
        setOwner(result[0].toString());
        setBlock(result[1].toNumber());
      })
      .then(unsub => {
        unsubscribe = unsub;
      });

    return () => unsubscribe && unsubscribe();
  // This tells the React hook to update whenever the file digest changes
  // (when a new file is chosen), or when the storage subscription says the
  // value of the storage item has updated.
  }, [digest, api.query.templateModule]);

  // We can say a file digest is claimed if the stored block number is not 0.
  function isClaimed () {
    return block !== 0;
  }

  // The actual UI elements which are returned from our component.
  return (
    <Grid.Column>
      <h1>Proof Of Existence</h1>
      {/* Show warning or success message if the file is or is not claimed. */}
      <Form success={!!digest && !isClaimed()} warning={isClaimed()}>
        <Form.Field>
          {/* File selector with a callback to `handleFileChosen`. */}
          <Input
            type='file'
            id="file"
            label="Your File"
            onChange={e => handleFileChosen(e.target.files[0])}
          />
          {/* Show this message if the file is available to be claimed */}
          <Message success header="File Digest Unclaimed" content={digest} />
          {/* Show this message if the file is already claimed. */}
          <Message
            warning
            header="File Digest Claimed"
            list={[digest, `Owner: ${owner}`, `Block: ${block}`]}
          />
        </Form.Field>
        {/* Buttons for interacting with the component. */}
        <Form.Field>
          {/* Button to create a claim. Only active if a file is selected,
          and not already claimed. Updates the `status`. */}
          <TxButton
            accountPair={accountPair}
            label={'Create Claim'}
            setStatus={setStatus}
            type="TRANSACTION"
            attrs={{ params: [digest], tx: api.tx.templateModule.createClaim }}
            disabled={isClaimed() || !digest}
          />
          {/* Button to revoke a claim. Only active if a file is selected,
          and is already claimed. Updates the `status`. */}
          <TxButton
            accountPair={accountPair}
            label="Revoke Claim"
            setStatus={setStatus}
            type="TRANSACTION"
            attrs={{ params: [digest], tx: api.tx.templateModule.revokeClaim }}
            disabled={!isClaimed() || owner !== accountPair.address}
          />
        </Form.Field>
        {/* Status message about the transaction. */}
        <div style={{ overflowWrap: 'break-word' }}>{status}</div>
      </Form>
    </Grid.Column>
  );
}
```
</div>

We won't walk you step by step through the creation of this component, but do
look over the code comments to learn what each part is doing.

## Submit a Proof

Your front-end should have automatically restarted with this new component
included. Select any file on your computer, and you will see that you can create
a claim with it's file digest:

![Proof Of Existence Component](assets/poe-component.png)

If you press "Create Claim", a transaction will be dispatched to your custom
Proof of Existence module, where this digest and the selected user account will
be stored on chain.

![Claimed File](assets/poe-claimed.png)

If all went well, you should see a new `ClaimCreated` event appear in the Events
component. The front-end can automatically recognize that your file is now
claimed, and even gives you the option to revoke the claim if you want.

Remember, only the owner can revoke the claim! If you select another user
account at the top, and you will see that the revoke option is disabled!

## Next Steps

This is the end of our journey into creating a Proof of Existence blockchain.

You have seen first hand how simple it can be to develop a brand new runtime
module and launch a custom blockchain using Substrate. Furthermore, we have
shown you that the Substrate ecosystem provides you with the tools to quickly
create responsive front-end experiences so users can interact with your
blockchain.

This tutorial chose to omit some of the specific details around development in
order to keep this experience short and satisfying. However, we want you to keep
learning!

If you are interested in learning how to program your own runtime module on
Substrate, please try our [Substrate Collectables
Workshop](https://substrate.dev/substrate-collectables-workshop/). This
comprehensive tutorial will teach you step by step how to program your own
custom runtime modules. Furthermore, it will start to introduce you to advance
runtime development concepts and best practices when building on Substrate.

It would also be a good time to call out that your success on the Substrate
framework will ultimately be limited on your ability to program in Rust. The
[Rust Book](https://doc.rust-lang.org/book/) is the best starting point for
developers of any experience.

If you experienced any issues with this tutorial or want to provide feedback,
feel free to [open a GitHub
issue](https://github.com/substrate-developer-hub/substrate-developer-hub.github.io/issues/new)
with your thoughts.

We can't wait to see what you build next!
