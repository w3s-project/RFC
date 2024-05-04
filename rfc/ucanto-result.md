## Abstract

Ucanto's `Result` type gives us the ability to use the TypeScript type system to specify the errors a function may return. 

We have started using `Result`s in a variety of different interfaces in our system, but have not been careful to catch and wrap runtime errors with specified error types in all cases. This has lead to functions that require callers to implement two different types of error handling - detection and handling of "error" `Result`s like `{error: {name: 'AnError', message: 'A message describing the error'}}` as well as standard `try...catch` error handling. 

We describe the tradeoffs and suggest that best practice is to coerce all errors into `Result` types in functions that return `Result`.

## `Ucanto.Result`

Ucanto `Result`s are described by the [UCAN invocation specification](https://github.com/ucan-wg/invocation/#6-result) and [implemented as a TypeScript `type` in `@web3-storage/ucanto`](https://github.com/web3-storage/ucanto/blob/main/packages/interface/src/lib.ts#L506). 

Briefly, a `Ucanto.Result` is a TypeScript `Record` with EITHER an `ok` or `error` key. If `ok` is defined, `error` must be `undefined`, and vice-versa. Defining an function that returns `Result` allows us to specify possible error types for that function, which enables conveniences like IDE auto-complete and helps clients understand what types of errors they should expect to handle:

```typescript
import * as Ucanto from '@ucanto/interface'

interface PlanGetSuccess {
  updatedAt: ISO8601Date
  product: DID
}

interface PlanNotFound extends Ucanto.Failure {
  name: 'PlanNotFound'
}

type PlanGetFailure = PlanNotFound

interface PlansStorage {
  /**
   * Get plan information for a customer
   *
   */
  get: (
    account: AccountDID
  ) => Promise<Ucanto.Result<PlanGetSuccess, PlanGetFailure>>
}

// using a Result from a normal TypeScript function
async function getProductForAccount(plansStorage: PlansStorage, account: AccountDID): Promise<DID | null> {
  const result = await plansStorage.get(account)
  if (result.ok) {
    return result.ok.product
  } else if (result.error.name === 'PlanNotFound' ) {
    return null
  } else {
    throw result.error
  }
}

// alternatively, Result.try can be used for a slightly different and arguably cleaner syntax
async function getProductForAccount(plansStorage: PlansStorage, account: AccountDID): Promise<DID | null> {
  try {
    return Result.try(await plansStorage.get(account)).product
  } catch (err) {
    if (err.name === 'PlanNotFound') {
      return null
    } else {
      throw err
    }
  }
}
```

## Mixed Error Handling Considered Annoying

`Ucanto.Result` represents an alternative error handling paradigm for {Java|Type}Script that has some advantages in a typed environment. However, without care to wrap and handle unexpected errors in a function that returns a `Result` it can make comprehensive error handling annoying for callers. This becomes especially apparent when using multiple layers of `Result`-returning functions:

```typescript
interface CustomerGetSuccess {
  email: string
  plan: DID
}

interface CustomerNotFound extends Ucanto.Failure {
  name: 'CustomerNotFound'
}

interface UnexpectedError extends Ucanto.Failure {
  name: 'UnexpectedError'
}

type CustomerGetFailure = CustomerNotFound | UnexpectedError

interface CustomersStorage {
  /**
   * Get customer information
   *
   */
  get: (
    account: AccountDID
  ) => Promise<Ucanto.Result<CustomerGetSuccess, CustomerGetFailure>>
}

interface PIIGetSuccess {
  email: string
}

type PIIGetFailure = CustomerNotFound | UnexpectedError

interface PIIStorage {
  /**
   * Get Personally Identifying Information about a customer
   *
   */
  get: (
    account: AccountDID
  ) => Promise<Ucanto.Result<PIIGetSuccess, PIIGetFailure>>
}

function createCustomersStorage(plansStorage: PlansStorage, piiStorage: PIIStorage): CustomersStorage {
  return {
    async get(account){
      let plan
      let email
      try {
        // this line may throw an error, so we wrap it in a try...catch
        const result = await plansStorage.get(account)
        if (result.ok) {
          plan = result.ok.plan
        // alternatively, it may return an error Result
        } else {
          return { error: new CustomerNotFound(`could not find plan for ${account}`, { cause: result.error }) }
        }
      } catch (err) {
        return { error: new UnexpectedError('Unexpected error getting plan', { cause: err }) }
      }

      try {
        const result = await piiStorage.get(account)
        if (result.ok) {
          email = result.ok.email
        } else {
          return { error: new CustomerNotFound(`could not find email for ${account}`, { cause: result.error }) }
        }
      } catch (err) {
        return { error: new UnexpectedError('Unexpected error getting email', { cause: err }) }
      }

      return { ok: { customer: { plan, email } } }
    }
  }
}
```

Note that we could choose to simply let unexpected errors bubble up, but this would push this error handling logic up to the caller, where there may not be sufficient context to understand the error. This is arguably not _soooo_ bad since many JavaScript functions also do this, but it means leaving some of the benefits of `Result` on the table.

## `Result` Consistency Calls for Developer Diligence

We can get a slightly better result if we assume `Result`-returning functions cannot throw errors:

```typescript
function createCustomersStorage(plansStorage: PlansStorage, piiStorage: PIIStorage): CustomersStorage {
  return {
    async get(account){
      const planResult = await plansStorage.get(account)
      if (planResult.ok) {
        customer.plan = planResult.ok.plan
        const piiResult = await piiStorage.get(account)
        if (result.ok) {
          return ({
            ok: {
              customer: {
                plan: planResult.ok.plan, 
                email: piiResult.ok.email
              }
            } 
          })
        } else {
          return { error: new CustomerNotFound(`could not find email for ${account}`, { cause: piiResult.error }) }
        }
      } else {
        return { error: new CustomerNotFound(`could not find plan for ${account}`, { cause: planResult.error }) }
      }
    }
  }
}
```

 In this world any unexpected errors should be considered bugs and fixed by wrapping possible error-producing blocks of code (or the entire function body) in a `try...catch` block:

 ```typescript
 function createCustomersStorage(plansStorage: PlansStorage, piiStorage: PIIStorage): CustomersStorage {
  return {
    async get(account){
      const planResult = await plansStorage.get(account)
      if (planResult.ok) {
        customer.plan = planResult.ok.plan
        const emailResult = await piiStorage.get(account)
        if (result.ok) {
          try {
            const name = await functionThatMayThrowAnError()
          } catch (err){
            return { error: new CustomerNotFound(`could not find name for ${account}`, { cause: err }) }
          }
          return ({
            ok: {
              customer: {
                plan: planResult.ok.plan, 
                email: emailResult.ok.email
              }
            } 
          })
        } else {
          return { error: new CustomerNotFound(`could not find email for ${account}`, { cause: emailResult.error }) }
        }
      } else {
        return { error: new CustomerNotFound(`could not find plan for ${account}`, { cause: planResult.error }) }
      }
    }
  }
}
 ```

 ## Proposal: `Result` Returning Functions Should Never Throw

 Given these two options, the latter feels a bit better - if we are committed to using `Result` in a variety of different domains, we propose that functions that return `Result` should never throw an error. This means that all code inside such functions that may throw an error should be wrapped in `try...catch` and convert caught errors to error `Result`s. 

 ## Open Question: When to use `Result`?

 `Result` feels particularly appropriate for interfaces and APIs where comprehensive error documentation is very useful. It may not be as useful in lower-level utility functions. We welcome discussion on when and when not to use `Result` in our codebase.

