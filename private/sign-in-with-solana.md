# Sign in with Solana: A Guide to Web3 Authentication

## Introduction
I recently had the opportunity to implement Sign in with Solana (SIWS) authentication. In this post, I will walk you through how to implement it on both the front-end and back-end. I hope this guide provides useful insights for anyone looking to integrate SIWS into their Solana applications.


## Why Use Sign in with Solana?
- **Decentralized authentication**: No need for centralized databases storing user credentials.
- **User control**: Users own their identity and can sign in securely without third-party involvement.
- **Seamless onboarding**: No need for password resets or email verifications.
- **Enhanced security**: Private keys never leave the user's wallet.

## How Sign in with Solana Works
1. The user connects their Solana wallet (e.g., Phantom, Solflare) to your website.
2. The website generates a unique message for authentication.
3. The user signs the message using their wallet.
4. The website verifies the signature and allows access.

## Example Implementation
Let's implement **Sign in with Solana** in a simple web app using JavaScript and Rust.

### Step 1: Front End
For the front-end, we leverage solana-web3.js along with Next.js authentication mechanisms.

Solana Web3.js Example

We integrate authentication using next/auth with a custom credential provider.

```js
    CredentialsProvider({
      id: "solana",
      name: "Login with Solana",
      credentials: {
        host: { label: "Host", type: "text" },
        address: { label: "Wallet Address", type: "text" },
        account: { label: "Account Info", type: "text" },
        signedMessage: { label: "Signed Message", type: "text" },
        signature: { label: "Signature", type: "text" },
      },
      async authorize(credentials) {
        const host = credentials?.host;
        const address = credentials?.address;
        const account = credentials?.account;
        const signedMessage = credentials?.signedMessage;
        const signature = credentials?.signature;

        if (!address || !signedMessage || !signature) {
          throw new Error("Missing credentials");
        }

        const parsedAccount = JSON.parse(account!);
        const parsedSignedMessage = JSON.parse(signedMessage!);
        const parsedSignature = JSON.parse(signature!);

        const { apiUrl } = getApiConfig();
        const response = await validateAndVerify(
          apiUrl,
          host!,
          parsedAccount,
          parsedSignedMessage,
          parsedSignature,
        );
        return {
          id: response.data,
          name: response.data,
        };
      },
    }),

const validateAndVerify = async (
  apiUrl: string,
  domain: string,
  accountData: UiWalletAccount | undefined,
  signedMessageData: Uint8Array<ArrayBufferLike> | undefined,
  signatureData: Uint8Array<ArrayBufferLike> | undefined,
) => {
  const url = `${apiUrl}/rest/login?domain=${domain}`;

  const convertPublicKeyToArray = (publicKey: any) => {
    if (!publicKey) return [];
    return Object.keys(publicKey)
      .sort((a, b) => Number(a) - Number(b))
      .map((key) => publicKey[key]);
  };

  const getByteArray = (input: any) => {
    if (!input) return [];
    if (input instanceof Uint8Array) return Array.from(input);
    if (input.data) return Array.from(input.data);
    return [];
  };

  const account = {
    publicKey: convertPublicKeyToArray(accountData?.publicKey),
  };
  const signedMessage = getByteArray(signedMessageData);
  const signature = getByteArray(signatureData);
  const data = {
    account,
    signedMessage,
    signature,
  };

  const res = await fetch(url, {
    headers: {
      "Content-Type": "application/json",
    },
    method: "POST",
    body: JSON.stringify(data),
  });

  if (res.status !== 200) {
    const statusText = res.statusText;
    const responseBody = await res.text();
    throw new Error(
      `NCN Portal has encountered an error with a status code of ${res.status} ${statusText}: ${responseBody}`,
    );
  }

  const json = await res.json();
  return json;
};
```

Component Integration

We create a React component to handle user authentication:

```js
"use client";

import React, { useContext, useState } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import { useSignIn } from "@solana/react";
import { Badge, Button, Dialog } from "@radix-ui/themes";
import { ChainContext } from "@/components/Provider/ChainContext";
import { SelectedWalletAccountContext } from "@/components/context/SelectedWalletAccountContext";
import { NO_ERROR } from "@/util/errors";
import { UiWalletAccount } from "@wallet-standard/react";
import { signIn } from "next-auth/react";

type Props = Readonly<{
  account: UiWalletAccount;
}>;

const SignIn = ({ account }: Props) => {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [selectedWalletAccount] = useContext(SelectedWalletAccountContext);
  const signInWithSolana = useSignIn(account);

  const [lastSignature, setLastSignature] = useState<Uint8Array | undefined>();
  const [error, setError] = useState<symbol | any>(NO_ERROR);
  const [isSendingTransaction, setIsSendingTransaction] = useState(false);

  const {
    displayName: currentChainName,
    chain,
    setChain,
  } = useContext(ChainContext);
  const currentChainBadge = (
    <Badge color="gray" style={{ verticalAlign: "middle" }}>
      {currentChainName}
    </Badge>
  );

  const request = async (requestType: string) => {
    const url = "/api/login";

    const data = {
      requestType,
      address: selectedWalletAccount?.address,
      domain: window.location.host,
    };

    return await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(data),
    });
  };

  const handleLogin = async (e: any) => {
    e.preventDefault();
    setError(NO_ERROR);
    setIsSendingTransaction(true);
    try {
      const resMessage = await request("getSiwsMessage");
      const messageJson = await resMessage.json();
      const { account, signedMessage, signature } = await signInWithSolana(
        messageJson.data,
      );

      const callbackUrl = searchParams.get("callbackUrl") || "/";

      const result = await signIn("solana", {
        redirect: false,
        callbackUrl,
        host: window.location.host,
        address: account.address,
        account: JSON.stringify(account),
        signedMessage: JSON.stringify(signedMessage),
        signature: JSON.stringify(signature),
      });

      if (result?.ok) {
        router.push(callbackUrl);
      } else {
        setError({ message: "Authentication failed" });
      }
    } catch (e) {
      setLastSignature(undefined);
      setError({ message: "You are not whitelisted" });
    } finally {
      setIsSendingTransaction(false);
    }
  };

  return (
    <div className="flex flex-col gap-4">
      {selectedWalletAccount ? (
        <Dialog.Root
          open={!!lastSignature}
          onOpenChange={(open) => {
            if (!open) {
              setLastSignature(undefined);
            }
          }}
        >
          <Dialog.Trigger>
            <Button
              color={error ? undefined : "red"}
              loading={isSendingTransaction}
              type="button"
              className="cursor-pointer"
              onClick={handleLogin}
            >
              Sign In With Solana
            </Button>
          </Dialog.Trigger>
        </Dialog.Root>
      ) : (
        <div></div>
      )}
    </div>
  );
};

export default SignIn;
```

Explanation:

The SignIn component handles the authentication process.

It connects to the Solana wallet, signs a message, and sends the credentials to the backend.

The server verifies the credentials, and upon success, the user is redirected to the appropriate page.

### Step 2: Backend

The backend in Rust is responsible for validating the signed message and issuing a JWT token upon successful authentication.

```rs
pub async fn login(
    State(_state): State<Arc<AppState>>,
    Query(params): Query<ValidateAndVerifyQueryParams>,
    Json(output): Json<SiwsOutput>,
) -> Result<impl IntoResponse, NcnPortalError> {
    let message = SiwsMessage::try_from(&output.signed_message)
        .with_context(|| "Failed to read siws message")
        .map_err(NcnPortalError::AuthError)?;

    // Validate the message
    message
        .validate(ValidateOptions {
            domain: Some(params.domain),
            nonce: None,
            time: None,
        })
        .with_context(|| "Failed to validate")
        .map_err(NcnPortalError::AuthError)?;

    match output.verify() {
        Ok(_res) => {
            let token = encode_jwt(message.address)?;
            let res: NcnPortalResponse<String> = NcnPortalResponse::new_with_success(Some(token));

            Ok(Json(res))
        }
        Err(e) => {
            let err_message = format!("Failed to verify: {e}");
            let err = anyhow::Error::msg(err_message);
            Err(NcnPortalError::AuthError(err))
        }
    }
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct GetSiwsMessageParams {
    address: String,
    domain: String,
}

pub async fn get_siws_message(
    State(state): State<Arc<AppState>>,
    Path(wallet_pubkey): Path<String>,
    Json(params): Json<GetSiwsMessageParams>,
) -> Result<impl IntoResponse, NcnPortalError> {
    let _pubkey = Pubkey::from_str(&wallet_pubkey)
        .map_err(|e| NcnPortalError::Unexpected(anyhow::Error::new(e)))?;

    let row = sqlx::query!(
        r#"
        SELECT *
        FROM admin_user
        WHERE pubkey = $1
        "#,
        wallet_pubkey,
    )
    .fetch_optional(&state.pool)
    .await
    .context("Failed to performed a query to retrieve stored whitelist.")?;

    match row {
        Some(_) => {
            let issued_at = OffsetDateTime::now_utc();
            let expiration_time = issued_at + time::Duration::minutes(5);

            let msg = SiwsMessage {
                domain: params.domain,
                address: params.address,
                uri: None,
                version: Some("0.0.1".to_string()),
                statement: None,
                chain_id: Some("solana:mainnet".to_string()),
                nonce: None,
                issued_at: Some(issued_at.into()),
                expiration_time: Some(expiration_time.into()),
                not_before: None,
                request_id: None,
                resources: vec![],
            };

            let res = NcnPortalResponse::new_with_success(Some(String::from(&msg)));
            Ok(Json(res))
        }
        None => {
            let err = anyhow::Error::msg("Not Found Pubkey");
            Err(NcnPortalError::Unexpected(err))
        }
    }
}
```

## Conclusion
**Sign in with Solana** offers a decentralized, secure, and user-friendly way to authenticate users on Web3 applications. By leveraging cryptographic signatures, users can verify their identities without relying on traditional centralized authentication mechanisms.

Start integrating SIWS into your Solana dApps today and embrace the future of Web3 authentication!

---

### Additional Resources
- [https://solana-labs.github.io/solana-web3.js/](https://github.com/phantom/sign-in-with-solana))

