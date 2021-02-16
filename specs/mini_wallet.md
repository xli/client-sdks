# Diem MiniWallet

MiniWallet is a highly simplified wallet application backend server that integrates with Diem Payment Network.


## Goals

1. Demonstrate SDK features for integrating an application with Diem Payment Network.
2. Play as counterparty wallet application for wallet applications CI builds, and develop a standard test suite for improving the ramp-up time of building Diem wallet applications.
3. Develop a standard test suite for testing cross-language SDKs features.


## Use Cases

![Use Cases](mini_wallet/use-cases.png)


## API Specification

The MiniWallet API is organized around [REST], accepts JSON-encoded request bodies, returns JSON-encoded responses, and uses standard HTTP response codes and verbs.

We value simplicity in favor of performance in MiniWallet API design.


### Endpoints Overview

1. Server responses 200 for succeeded request.
2. For `POST` creation request, server should accept JSON encoded request body, and response created resource JSON as the body of the response; no `Location` header is required.
3. Server should set `Content-Type` header to `application/json` for JSON responses.
4. Server should respond 400 for client error,  and 500 for server internal error.
5. There is no requirement about the response body for errors; recommend human readable messages and stack traces for easy debugging.


| Method | Path                 | Description                                                                |
|--------|----------------------|----------------------------------------------------------------------------|
| GET    | /accounts            | Returns all accounts ordered by creation time.                             |
| POST   | /accounts            | Create a new account with initial deposit.                                 |
| GET    | /receive\_payments   | Returns all receive\_payments ordered by creation time.                    |
| POST   | /receive\_payments   | Create a payment that is paid by a payer to the account                    |
| GET    | /transactions        | Returns all transactions ordered by creation time.                         |
| POST   | /transactions        | Deposit credit or create a payment that is paid by the account to a payee. |
| POST   | /sync                | Trigger background tasks for syncing data with external systems.           |
| GET    | /samples/kyc         | Returns [KycSamples](kyc-samples) for clients to use in test.              |
| POST   | /offchain/v2/command | Diem Off-chain API. Please refer to [DIP-1] for the API details.           |

> `POST /accounts` should prevent account balances exceeding the sum of all VASP on-chain account amounts.

> `POST /commands` should first make an outbound call to counterparty service off-chain API. If the outbound call failed, return the error message. If the outbound call succeeds, update the command and return success.

> `POST /send_payments` should prevent account balance from becoming negative.

> `POST /sync` is designed for clients to trigger data synchronization with external systems, which makes tests more efficient.
> For example, a client can call this API while it is waiting for a payment transaction synchronized from Diem Network.
> A MiniWallet implementation can choose to do nothing if manual triggering data sync is not an option.
> Server should always respond to code 204 without a response body.
> Server should respond with 500 errors if someone went wrong.


### Resources

`Type` column flags:
1. `!`: read-only field, server sets value.
2. `?`: nullable field


#### Account

Account is a record of payment activities and transactions. It serves for multiple purposes:

1. Initializing test data.
2. Isolating test data.

All other resources are scoped by account id.

| Attribute                  | Type    | Description                                |
|----------------------------|---------|--------------------------------------------|
| id                         | string! | Unique account id                          |
| initial\_deposit\_currency | string  | Currency code of the initial deposit.      |
| initial\_deposit\_amount   | uint    | The amount of initial deposit.             |
| kyc\_data                  | string  | Valid [KycDataObject] JSON-encoded string. |

> Server generates `additional_kyc_data` by the following format: Use account `id` as `additional_kyc_data` if it is requested by the counterparty service.


#### ReceivePayment

ReceivePayment records an intent to receive funds from a payer to the account.

| Attribute           | Type    | Description                                       |
|---------------------|---------|---------------------------------------------------|
| id                  | string! | Unique identifier                                 |
| account\_identifier | string! | [Account Identifier] for receiving the payment.   |
| subaddress_hex      | string! | Hex-encoded subaddress for receiving the payment. |
| account\_id         | string  | Account id                                        |
| currency            | string? | Currency code                                     |
| amount              | uint?   | Amount of the payment                             |

> `account_identifier` should be created by Diem account address and the subaddress from `subaddress_hex`.


#### Transaction

Transaction represents funds moving through a Mini Wallet account.

1. To deposit credits to the account, create a Transaction without `payee`.
2. To send payment to others, create a Transaction with `payee`.
3. Attribute `subaddress_hex` should be set by the server when the account subaddress used for the Diem transaction is created.
3. Attribute `signed_transaction` should be set for send payment transaction after the Diem transaction is submitted to Diem Network.
4. Attribute `diem_transaction_version` should be set after the transaction is executed successfully on Diem Network.
5. When received a Diem payment transaction successfully, a new Transaction should be created with the following attributes set:
   1. `diem_transaction_version`
   2. `subaddress_hex`
   3. `currency`
   4. `amount`
6. It's optional to set `signed_transaction` if the Transaction is a received payment transaction.
7. For the case `account_id` could not be found for an external payment transaction, server should create transaction without `account_id`.
8. When calculating account balance, `amount` field should be converted to negative if `payee` is set.
9. Canceled transactions should not be included in account balance.
10. Server should set transaction `status` to `pending` before `diem_transaction_version` is set or `canceled`.
11. Server should set transaction `status` to `completed` when Diem transaction is executed successfully.
12. Server should set transaction `status` to `canceled` when off-chain KYC data exchange failed or Diem transaction execution failed.
13. There is no support for withdrawing funds.


| Attribute                  | Type     | Description                                                                                  |
|----------------------------|----------|----------------------------------------------------------------------------------------------|
| id                         | string!  | Unique identifier                                                                            |
| account\_id                | string?  | Account id; it is not set if no account found by the transaction metadata                    |
| status                     | string!  | Status of the transaction, valid values: pending, completed, canceled                        |
| cancel_reason              | string?! | Human readable message to show why transaction is canceled; set if `status` == `canceled`    |
| currency                   | string   | Diem currency code.                                                                          |
| amount                     | uint     | Amount of the transaction.                                                                   |
| payee                      | string?  | [Account identifier], the receiver of the payment if it is set.                              |
| subaddress\_hex            | string?! | Hex-encoded subaddress for sending or receiving payment.                                     |
| signed\_transaction        | string?! | Hex-encoded signed transaction bytes. Set by server after submitting the signed transaction. |
| diem\_transaction\_version | uint?!   | Associated Diem transaction version.                                                         |


#### Kyc Samples

KYC data samples for clients to create account to do off-chain KYC data exchanging test.


| Attribute    | Type    | Description                                                                                                 |
|--------------|---------|-------------------------------------------------------------------------------------------------------------|
| minimum      | string! | JSON-encoded [KycDataObject]. [PaymentActorObject] containing it or with more data should be accepted.      |
| reject       | string! | JSON-encoded [KycDataObject]. [PaymentActorObject] containing it should be rejected.                        |
| soft\_match  | string! | JSON-encoded [KycDataObject]. [PaymentActorObject] containing it should trigger `soft_match`, then approve. |
| soft\_reject | string! | JSON-encoded [KycDataObject]. [PaymentActorObject] containing it should trigger `soft_match`, then reject.  |




[REST]: https://en.wikipedia.org/wiki/Representational_state_transfer
[DIP-1]: https://dip.diem.com/dip-1
[Account Identifier]: https://dip.diem.com/dip-5/#account-identifiers
[Diem ID]: https://dip.diem.com/dip-10
[KYC Data]: https://en.wikipedia.org/wiki/Know_your_customer
[KycDataObject]: https://dip.diem.com/dip-1/#kycdataobject
[PaymentActorObject]: https://dip.diem.com/dip-1/#paymentactorobject
