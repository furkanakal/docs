import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Auth Methods

Authentication methods are ways of asigning Programmable Key Pairs (PKP) to a specific account resource. This requires individuals to authenticate before performing operations requiring a PKP. This is a powerful feature of the Lit network as it means users can sign up for a wallet the same way they sign up for other types of digital resources, thus lowering the barrier to accessing web3 enabled applications.

## What is authentication?

An authentication method refers to the specific credential (i.e a wallet address, Google oAuth, or Discord account) that is programmatically tied to the PKP and used to control the underlying key-pair. Only the auth method associated with a particular PKP has the ability to combine the underlying shares. You can read more about how authentication works with PKPs on our [blog](https://spark.litprotocol.com/how-authentication-works-with-pkps/).

Right now, there are two main ways to do auth with Lit Actions. We will dive into these two methods below.

## Using Lit Auth Directly

Several auth methods are supported by Lit directly. These include methods configured using the [PKPPermissions](https://github.com/LIT-Protocol/LitNodeContracts/blob/main/contracts/PKPPermissions.sol) contract, the user holding the PKP NFT, or assigned via a Lit Action with permission to sign using the PKP. If you use Lit auth directly, you are limited to the auth methods that we support. We provide an easy to use SDK to help you add auth methods to a PKP. You can find the SDK [here](https://github.com/LIT-Protocol/js-sdk/tree/master/packages/lit-auth-client).

### Existing supported auth methods

| Auth Method Name     | Auth Method Type Number | Description                                                                                                                                                                                                                                                                                         |
| -------------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| NULLMETHOD           | 0                       | Don't use this one, it's just a placeholder                                                                                                                                                                                                                                                         |
| ADDRESS              | 1                       | An Ethereum address. As long as the user presents an AuthSig with this address, they can sign using the PKP.                                                                                                                                                                                        |
| ACTION               | 2                       | A Lit Action. This is the IPFS CID of the Javascript that is your Lit Action, base58 decoded. As long as the user is calling a Lit Action with this CID, the Lit Action can sign using this PKP. It's very important to only authorize actions that you trust, because they can sign using the PKP. |
| WEBAUTHN             | 3                       | A WebAuthn Public Key. Take a look at our [WebAuthn example](https://github.com/LIT-Protocol/oauth-pkp-signup-example/tree/main) to learn more.                                                                                                                                                     |
| DISCORD              | 4                       | Discord Oauth Login                                                                                                                                                                                                                                                                                 |
| GOOGLE               | 5                       | Google Oauth Login. You should try to use the Google JWT Oauth Login below if you can, since it's more efficient and secure.                                                                                                                                                                        |
| GOOGLE_JWT           | 6                       | Google Oauth Login, except where Google provides a JWT. This is the most efficient way to use Google Oauth with Lit because the Lit nodes only need to check the JWT signature against the Google certificates, and don't need to make HTTP requests to the Google servers to verify the token.     |
| APPLE_JWT            | 8                       | Sign in with Apple Login                                                                                                                                                                                                                                                                            |
| STYTCH_JWT           | 9                       | Stytch Login using the Stytch user. Can use any supported Stytch auth method but note - the Stytch account admin can change underlying identifiers like phone number. To prevent this attack, use one of the explicit Stytch auth methods below                                                     |
| STYTCH_EMAIL_OTP     | 10                      | Stytch Login using the Stytch user's email address. This is a one-time password (OTP) sent to the user's email address.                                                                                                                                                                             |
| STYTCH_SMS_OTP       | 11                      | Stytch Login using the Stytch user's phone number. This is a one-time password (OTP) sent to the user's phone number.                                                                                                                                                                               |
| STYTCH_WHATS_APP_OTP | 12                      | Stytch Login using the Stytch user's WhatsApp number. This is a one-time password (OTP) sent to the user's WhatsApp account.                                                                                                                                                                        |
| STYTCH_TOTP          | 13                      | Stytch Login using the Stytch user's TOTP. This is a one-time password (OTP) generated by the user's authenticator app.                                                                                                                                                                             |

Check out the implementation details within the SDK section [here](../../sdk/authentication/session-sigs/auth-methods/overview).

### Auth Method Scopes

Auth methods support scoping, which permits what they can be used for within Lit. These scopes are passed in to the "scopes" array as numbers when adding an auth method, or minting a PKP with PKPHelper. An overview of minting with scopes is provided in this [section](../wallets/minting). The scopes are as follows:

| Scope Name         | Scope Number | Description                                                                                                                                                                                                                                               |
| ------------------ | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sign Anything      | 1            | This scope allows signing any data                                                                                                                                                                                                                        |
| Only Sign Messages | 2            | This scope only allows signing messages using the [EIP-191 scheme](https://eips.ethereum.org/EIPS/eip-191) which prefixes "Ethereum Signed Message" to the data to be signed. This prefix prevents creating signatures that can be used for transactions. |

You can also set scopes: [] which will mean that the auth method can only be used for authentication, but not authorization. This means that the auth method can be used to prove that the user is who they say they are, but cannot be used to sign transactions or messages.

Any auth methods (regardless of scope) passed in to a Lit Action will be resolved/checked and put into the Lit.Auth object which is available inside the Lit Action. However, when you try to sign something using signEcdsa(), you'll find that it checks the scopes of the auth methods passed in, and will only sign if the appropriate scope is present.

Using this strategy, you could have a Lit Action that governs all signing for a user, and then add many auth methods with scopes: [], so that they cannot be used on their own without the Lit Action. You would then also use addPermittedAction() with scopes: [1] on the PKP to permit that action to sign. Then, inside the action, you can check if the auth methods resolved in Lit.Auth are authorized to sign, and if so, sign the data.

Using this strategy, you could implement your own MFA, where the user must present 2 or more auth methods to sign something, for example.

**Adding permitted scopes to existing PKPs**
1. Verify the scopes:
```js
import { LitAuthClient } from '@lit-protocol/lit-auth-client';
import { LitContracts } from '@lit-protocol/contracts-sdk';
import { AuthMethodScope, AuthMethodType } from '@lit-protocol/constants';

const authMethod = {
  authMethodType: AuthMethodType.EthWallet,
  accessToken: ...,
};

const authId = LitAuthClient.getAuthIdByAuthMethod(authMethod);

const scopes = await contractClient.pkpPermissionsContract.read.getPermittedAuthMethodScopes(
  tokenId,
  AuthMethodType.EthWallet,
  authId,
  3 // there are only 2 scope numbers atm. and index 0 doesn't count
);

// -- validate both scopes should be false
if (scopes[1] !== false) {
  return fail('scope 1 (sign anything) should be false');
}

if (scopes[2] !== false) {
  return fail('scope 2 (only sign messages) should be false');
}
```
2. Set the scopes:
```js
import { LitAuthClient } from '@lit-protocol/lit-auth-client';
import { LitContracts } from '@lit-protocol/contracts-sdk';
import { AuthMethodScope, AuthMethodType } from '@lit-protocol/constants';

const authMethod = {
  authMethodType: xx,
  accessToken: xxx,
};

const authId = LitAuthClient.getAuthIdByAuthMethod(authMethod);

const setScopeTx =
  await contractClient.pkpPermissionsContract.write.addPermittedAuthMethodScope(
    tokenId,
    AuthMethodType.EthWallet,
    authId,
    AuthMethodScope.SignAnything
  );

await setScopeTx.wait();
```

**Demos**: 
1. [Minting a PKP with an auth method and permitted scopes (Easy)](https://github.com/LIT-Protocol/js-sdk/blob/feat/SDK-V3/e2e-nodejs/group-contracts/test-contracts-write-mint-a-pkp-and-set-scope-1-2-easy.mjs)

2. [Minting a PKP with an auth method and permitted scopes (Advanced)](https://github.com/LIT-Protocol/js-sdk/blob/feat/SDK-V3/e2e-nodejs/group-contracts/test-contracts-write-mint-a-pkp-and-set-scope-1-advanced.mjs)

3. [Minting a PKP with no permissions, then add permitted scopes](https://github.com/LIT-Protocol/js-sdk/blob/feat/SDK-V3/e2e-nodejs/group-contracts/test-contracts-write-mint-a-pkp-then-set-scope-1.mjs)

4. [Minting a PKP using the relayer, adding permitted scopes, and getting session sigs](https://github.com/LIT-Protocol/js-sdk/tree/feat/SDK-V3/e2e-nodejs/group-pkp-session-sigs)

### Adding a Permitted Address

You can use the [PKPPermissions contract](https://github.com/LIT-Protocol/LitNodeContracts/blob/main/contracts/PKPPermissions.sol#L418) to add additional permitted auth methods and addresses to your PKP. Note that any permitted users will be able to execute transactions, authorized Lit Actions, and additional functionality associated with that PKP.

### Sending the PKP to itself

Sending a PKP to itself is possible, because the PKP is an NFT and also a wallet. This is useful if you want to make sure that only the PKP itself can change it's auth methods. You can use our handy auth helper contract [here](https://github.com/LIT-Protocol/LitNodeContracts/blob/main/contracts/PKPHelper.sol) and use that contract there is a parameter called `sendPkpToItself` in the `mintNextAndAddAuthMethods` function that you can set to true to send the PKP to itself.

### Obtaining the PKP Public Key

After a PKP is generated and assigned an auth method, you can pass the AuthMethodType and AuthMethodId into this [function](https://github.com/LIT-Protocol/LitNodeContracts/blob/ed2adf77e63f371ef864846dc9e92fef42f0ebb1/contracts/PKPPermissions.sol#L99) to obtain the PKP ID. The PKP ID can be used to fetch the PKP's public key by passing it into this [function](https://github.com/LIT-Protocol/LitNodeContracts/blob/ed2adf77e63f371ef864846dc9e92fef42f0ebb1/contracts/PKPPermissions.sol#L78).

The PKP public key is required to initialize a new 'wallet' object when using [Lit and WalletConnect](https://github.com/LIT-Protocol/pkp-walletconnect/blob/main/components/CallRequest.js#L44) together.

You will also need the PKP public key in order to generate a [sessionSig](../../sdk/authentication/session-sigs/intro) which is required to communicate with the Lit nodes, as seen in this [example](https://github.com/LIT-Protocol/oauth-pkp-signup-example/blob/main/src/App.tsx#L422).

## Custom Auth

If you would like further customization over your PKP auth methods, you can do auth yourself with a Lit Action, using the auth helpers we provide (see below). In this scenario, after you give your Lit Action permission to use the PKP, the typical flow is to burn the PKP NFT or send it to itself. It is important to note, if you do decide to burn the PKP, you will be unable to add additional auth methods in the future. If you go this route, your auth basically looks like a bunch of if statements inside the Lit Action.

If you decide to use your own auth, you can still use the PKPPermissions contract to define your method(s) of choice, or deploy your own access control contract. If you use the deployed Lit PKPPermissions contract, then it is important to pick a unique authMethodType that isn't used by anyone else, ever. Since it can be a uint256, you should do something like `sha256("some unique or random string")` to pick a unique authMethodType number. You can find the current methods being used [here](https://github.com/LIT-Protocol/LitNodeContracts/blob/main/contracts/PKPPermissions.sol#L25). If this is the route you choose, you could check the PKPPermissions contract in your Lit Action using [this function](https://github.com/LIT-Protocol/LitNodeContracts/blob/main/contracts/PKPPermissions.sol#L25).

## How does authentication differ from authorization?

Authorization refers to an [auth signature](../../sdk/authentication/auth-sig), which is **always required** to communicate with the Lit nodes and make a request to the network. It doesn't matter if you are decrypting a piece of data or calling a Lit Action, an auth sig will always be required.

In the case that a user doesn’t own a wallet (and therefore cannot produce a valid AuthSig), they can present their alternative auth method to the Lit SDK which will convert it into a “compliant” AuthSig. This is documented in our [docs](../../sdk/authentication/session-sigs/auth-methods/overview). The flow is as follows:

1. Present a PKP public key and an auth token from an authorized auth method (like a Google OAuth JWT), as well as a session public key for a local key-pair that is generated and stored locally.
2. The PKP is used to sign a SIWE signature which authorizes the session key-pair going forward.
3. The Lit SDK will use the session key to sign future requests. So instead of signing the session key-pair with a wallet, you can sign it using the PKP by communicating with the Lit nodes and presenting proof that you are authorized.

## Authentication Helpers

Inside of your Lit Actions, there is an object called `Lit.Auth` that will be pre-populated with the resolved Auth Methods, and a few other items. For example, if you pass a Google Oauth Token, then the Lit Nodes will resolve the Oauth Token into a user ID and application ID and those will be available to you in `Lit.Auth`. `Lit.Auth` has the following members:

- actionIpfsIds: An array of IPFS IDs that are being called by this Lit Action. This will typically only have a single item, but if you call multiple Lit Actions from inside your Lit Action, they will all be included here. For example, if you have two Lit Actions, A, and B, and A calls B, then the first item in the array will be A and the last item will be B. Therefore, the last item in the array is always the IPFS ID of the Lit Action that is currently running.
- authSigAddress: A verified wallet address, if one was passed in. This is the address that was used to sign the AuthSig.
- authMethodContexts: An array of auth method contexts. Each entry will contain the following items: `userId`, `appId`, and `authMethodType`. A list of AuthMethodTypes can be found [here](#existing-supported-auth-methods) in the docs and is used [here](https://github.com/LIT-Protocol/LitNodeContracts/blob/main/contracts/PKPPermissions.sol#L25) in the PKPPermissions Contract.

Important to note on Authentication Helpers: authorization is not included. This means that a user can present a Google oAuth JWT as an auth method to be resolved and validated by your Lit Action. The Action will then stick the result inside the Lit.Auth object. In this case, the result would be the users verified google account info like their user id, email address, and more.

## Example: Setting Auth Context with Lit Actions

This example shows how to assign different auth methods to a PKP using a Lit Action.

```js
import * as LitJsSdk from "@lit-protocol/lit-node-client";

// this code will be run on the node
const litActionCode = `
const go = async () => {
  Lit.Actions.setResponse({response: JSON.stringify({"Lit.Auth": Lit.Auth})})
};

go();
`;

// you need an AuthSig to auth with the nodes
// normally you would obtain an AuthSig by calling LitJsSdk.checkAndSignAuthMessage({chain})
const authSig = {
  sig: "0x2bdede6164f56a601fc17a8a78327d28b54e87cf3fa20373fca1d73b804566736d76efe2dd79a4627870a50e66e1a9050ca333b6f98d9415d8bca424980611ca1c",
  derivedVia: "web3.eth.personal.sign",
  signedMessage:
    "localhost wants you to sign in with your Ethereum account:\n0x9D1a5EC58232A894eBFcB5e466E3075b23101B89\n\nThis is a key for Partiful\n\nURI: https://localhost/login\nVersion: 1\nChain ID: 1\nNonce: 1LF00rraLO4f7ZSIt\nIssued At: 2022-06-03T05:59:09.959Z",
  address: "0x9D1a5EC58232A894eBFcB5e466E3075b23101B89",
};

const runLitAction = async () => {
  const litNodeClient = new LitJsSdk.LitNodeClient({
    alertWhenUnauthorized: false,
    litNetwork: "localhost",
    debug: true,
  });
  await litNodeClient.connect();
  const results = await litNodeClient.executeJs({
    code: litActionCode,
    authSig,
    authMethods: [
      // {
      //   // discord oauth
      //   accessToken: "M1Y1WnYnavzmSaZ6p1LBLsNFn2iiu0",
      //   authMethodType: 4,
      // },
      // {
      //   // google oauth
      //   accessToken:
      //     "ya29.a0Aa4xrXMCyLStBQzLhC8il8YRPXIkEEgno9nB4PKvjCi6oIu-uIjeIoyfQoR99TcZf0IUMPfJfjRIJyIXtLk_kXLa5BmdUyJcJGP8SB4-UjlebOILidfItC8KR1sQR9LSFX55cw3_GTa5IqCOCTXME38z5ZMZaCgYKATASARASFQEjDvL9HinQH3Mk1UclCD011YbLfQ0163",
      //   authMethodType: 5,
      // },
      // {
      // email / sms
      //   accessToken: "eyJhbGciOiJzZWNwMjU2azEiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJMSVQtUHJvdG9jb2wiLCJzdWIiOiJMSVQtT1RQIiwiaWF0IjoxNjgzMjIzNjIyMDg5LCJleHAiOjE2ODMyMjU0MjIwODksIm9yZ0lkIjoiTElUIiwicm9sZSI6InVzZXIiLCJleHRyYURhdGEiOiIrMTIwMTQwNzIwNzN8MjAyMy0wNS0wNFQxODowNzowMi4wODkxODgrMDA6MDAifQ.eyJyIjoiOTRiOWE1ODkyODFlYzdlYmZlZTdjOGRjMjU0YTk1NGY5NjY1N2IzZmRkNmFlMWIwZThmMmY1OWIxMWYwNTU1YSIsInMiOiI0NWNlNTA0YTBkZjFlZWFkMWYxMGIyYTQ1MjU4ZjlhOTI5ZTY5ODYzYjIzNDdlZGViMmRkODMxM2Y4NDVhNDA1In0"
      //   authMethodType: 7,
      // }
      {
        // google oauth JWT
        accessToken:
          "eyJhbGciOiJSUzI1NiIsImtpZCI6Ijc3Y2MwZWY0YzcxODFjZjRjMGRjZWY3YjYwYWUyOGNjOTAyMmM3NmIiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJhenAiOiI0MDc0MDg3MTgxOTIuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJhdWQiOiI0MDc0MDg3MTgxOTIuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJzdWIiOiIxMDg5OTYwNTQyNzMzNjA1NjgxMzIiLCJlbWFpbCI6ImdldmVuc3RlZUBnbWFpbC5jb20iLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXRfaGFzaCI6IlVYV1Z1eEJsdGswcEhKclllOEFXTUEiLCJpYXQiOjE2NjcxNjgyMTUsImV4cCI6MTY2NzE3MTgxNX0.ejZu5bADJ6cUsovV7otHAafy0mqWZBAtN860jvBdVe38XUi0v-eB5WWBPMD5zXcJxbXFvaPWCX8nTaE6S24cNNHJw0hq15irjRZeg9D2i7ToitR1LZSQ3rPCDQZPX4xYn7G-FH7C1DQ-7NEDMmr9ge4B6Qs4pT5Mj8ESVlA29yZjKCfk-zL7F5b6W0EOIA6G9rj6-3HgtazkHfIGHAtfBz4dqHjC4HJncHJzqIm9Y8eSBBnN-ZhYUr3cWxGCuFIw3yrGccv5_khfhbbk6TqdSeMO9YNWN3otiVB8Nwu2sb9VsllFoHIE0uGSzVZVbJgSK1GsGbJZe76ubLuObI5YFw",
        authMethodType: 6,
      },
    ],
    // all jsParams can be used anywhere in your litActionCode
    jsParams: {
      // this is the string "Hello World" for testing
      toSign: ethers.utils.arrayify(
        ethers.utils.keccak256([
          72, 101, 108, 108, 111, 32, 87, 111, 114, 108, 100,
        ])
      ),
      publicKey:
        "0x0404e12210c57f81617918a5b783e51b6133790eb28a79f141df22519fb97977d2a681cc047f9f1a9b533df480eb2d816fb36606bd7c716e71a179efd53d2a55d1",
      sigName: "sig1",
    },
  });
  console.log("results: ", JSON.stringify(results.response, null, 2));
};

runLitAction();
```
