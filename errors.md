# Error Handing in Go for Java Authors

This document compares error handling methodologies in Go and Java.  It holds the view
that there are more commonalities than not between the two languages in terms
of capabilities, approaches, and level of expedience.

The analysis is concerned solely with the matters of having a programmer programmatically deal
with error conditions, not with operators debugging problems at runtime.  This runtime problem
diagnosis use case is perfectly valid but beyond the scope of this document.

## Cheap and Easy

On the cheapest end of the spectrum, one can use stringly-typed errors in both
languages:

```java
void compute(Argument arg) {
  if (!arg.isValid()) {
    throw new IllegalArgumentException("arg is invalid.");
  }
  // Rest omitted.
}

```

```go
func Compute(arg Argument) error {
  if !arg.valid() {
    return fmt.Errorf("arg is invalid.")
  }
  // Rest omitted.
}
```

Both errors contain some information for a human to debug the problem from logs and
middleware handlers. But all their callers can do (at runtime) is merely detect that
there was an error, potentially the problem class.  In neither case can the program
react to the error beyond simple happy and sad paths.

For some cases the boolean error case may be acceptable.

## Discriminating Error Reasons

In contrast to the _Cheap and Easy_ approach, one may require the call graph to take
decisive and intelligent action based on a specific error condition and provide general handling for everything else.

```java
void debitAccount(Integer accountId, Integer amount) throws AuthorizationException {
  // Only agent may debit the principal's account.
  if (!isDebitAuthorized(this.agentId, accountId)) {
    throw new AuthorizationException(this.agentId, accountId, "debit");
  }
  // Rest omitted.
}

class AuthorizationException extends Exception {
  private Integer agentId;
  private Integer accountId;
  private String operation;
  
  AuthorizationException(Integer agentId, Integer accountId, String operation) {
    this.agentId = agentId;
	this.accountId = accountId;
	this.operation = operation;
  }
  
  public String toString() {
    return "AuthorizationException(" + agentId + " is not authorized to " + opertation + " account" + accountId;
  }
}
```

```go
func debitAccount(agentId, accountId, amount int) error {
  if !isDebitAuthorized(agentId, accountId) {
    return &AuthorizationError{agentId: agentId, accountId: accountId, operation: "debit"}
  }
  // Rest omitted.
}

type AuthorizationError struct {
  agentId, accountId int
  operation string
}

func (err *AuthorizationError) Error() string {
  return fmt.Sprintf("%v is not authorized to %v %v", err.agentId, err.accountId, err.operation)
}
```

In either case, the caller of `debitAccount` can distinguish between _which_ error occurs.

```java
boolean reconcileAccount() {
  try {
    debitAccount(agentId, accountId, amount);
  } catch (AuthorizationException ex) {
    return false;
  } catch (Exception ex) {
    pageOncall(ex);
	return false;
  }
  return true;
}
```

```go
func (m *transactionManager) reconcileAccount() bool {
  err := debitAccount(m.agentId, m.accountId, m.amount)
  switch err.(type) {
    case *AuthorizationError:
	case nil:
	  return true
	default:
	  m.pageOncall(err)
  }
  return false
}
```

If an `AuthorizationException` or `AuthorizationError` arise, we can handle it accordingly, leaving
any other errors to page the poor oncall.  For instance, the two functions could respectively produce
`TransactionException` or `TransactionError`. These are probably worth alerting the DBA for.

In Go, you can take another approach.  In the example above, we included information about the context
in which the error occurred: `agentId`, `accountId`, and `operation`.  It may be that the caller does
not need this information, even for reporting.  Potentially using error sentinel is appropriate here.

```go
var ErrNotAuthorized = errors.New("not authorized")

func debitAccount(agentId, accountId, amount integer) error {
  if !isDebitAuthorized(agentId, accountId) {
    return ErrNotAuthorized
  }
  // Rest omitted.
}

func (m *transactionManager) reconcileAccount() bool {
  err := debitAccount(m.agentId, m.accountId, m.amount)
  switch err {
    case ErrNotAuthorized:
	case nil:
	  return true
	default:
	  m.pageOncall(err)
  }
  return false
}
```

## Structured Data

So far the examples we have provided have given the program meagre ability to provide custom behavior based on the error circumstances.
We are going to take this further.  In this example we will compute a ledger for accounts at month close.  The code in question assumes the purchase records have been stored in various
database shards (all numeric).  Let's assume that the computation requires the databases to be in a groomed state before performing the important queries, but grooming is very expensive, so
we do not want to arbitrarily do it.  In this case, we assume that `GroomException` and `GroomError` include information that the caller can use to determine what went wrong and programmatically
act accordingly.

```java
Ledger computeMonthClose(Integer accountId) {
  Ledger.Builder memo = Ledger.newBuilder();
  List<Integer> shardIds = getDatabaseShardsFor(accountId);
  List<GroomException> toGroom = new List<>();
  for (Integer shardId : shardIds) {
    try {
      memo.addTransactions(loadRecords(accountId, shardId));
    } catch (GroomException ex) {
	  toGroom.add(ex);
	} catch (Exception ex) {
	  pageOncall(ex);
	  return Ledger.empty();  // Better to return nothing than something incorrect.
	}
  }
  
  // Retry
  
  for (GroomException pending : toGroom) {
    performGroom(pending.getDatabasePath());
	try {
	  memo.addTransactions(loadRecords(accountId, pending.getShardId()));
    } catch (GroomException ex) {
	  // Can't happen, but it is a checked exception.
	} catch (Exception ex) {
	  pageOncall(ex);
	  return Ledger.empty();  // Better to return nothing than something incorrect.
	}
  }

  return memo.build();
}

Transaction loadRecords(Integer accountId, shardId) throws GroomException {
  String databasePath = getDatabasePath(accountId, shardId);
  if (!loadDataState(accountId, shardId).Equal(GROOMED)) {
    throw new GroomException(databasePath, shardId);
  }
  // Rest omitted.
}

class GroomException extends Exception {
  private String databasePath;
  private Integer shardId;
  
  GroomException(String databasePath, Integer shardId) {
    this.databasePath = databasePath;
	this.shardId = shardId;
  }
  
  String getDatabasePath() {
    return databasePath;
  }
  
  Integer getShardId() {
    return shardId;
  }
}
```

```go
func (r *AccountRecords) computeMonthClose(accountId int) *Ledger {
  var out Ledger
  shardIds := r.databaseShards(accountId)
  var toGroom []*GroomError
  for shardId := range shardIds {
    records, err := r.loadRecords(accountId, shardId)
	switch err.(type) {
  	case nil:
	  out.merge(records)
	case *GroomError:
	  toGroom = append(toGroom, err)
	default:
	  r.pageOncall(err)
	  return nil // Better to return nothing than something incorrect.
	}
  }
  
  // Retry
  
  for _, pending := range toGroom {
    r.performGroom(pending.DBPath)
	records, err := r.loadRecords(accountId, shardId)
	if err != nil {
	  r.pageOncall(err)
	  return nil // Better to return nothing than something incorrect.
	}
	out.merge(records)
  }
  return &out
}

func (r *AccountRecords) loadRecords(accountId, shardId int) (*Transaction, error) {
  dbPath := r.databasePath(accountId, shardId)
  if (r.dataState(databasePath) != Groomed) {
    return nil, &GroomError{DBPath: dbPath, ShardId: shardId}
  }
  // Rest omitted.
}

type GroomError struct{
  DBPath string
  ShardId int
}

func (err *GroomError) Error() string { return fmt.Sprintf("DBPath %q for ShardID %v requires grooming.", err.DBPath, err.ShardID) }
```

While the examples are contrived, they do demonstrate that there is a significant correspondence between structured error handling in Java and Go.
Go makes authoring structured error types a breeze.  Consider:

```go
type CycleError struct {
  Path []Node
}

func (err *CycleError) Error() string {
  // Rest omitted.
}
```

Not only can your debug representation of the error be nice; your APIs callers can extensibly handle defects at-will.

## Reappraisal with golang.org/x/exp/errors

In Go 1.13, many of the ideas in golang.org/x/exp/errors will debut.

### Call Stack Diagnostic Information

Out of the box up until Go 1.13, the Go standard library provide no way to perform stack trace annotation on errors.  This is a disadvantage compared to Java's Throwable with tracked stack trace, yet not
providing such information does mean errors are treated as ordinary values, which means errors in Go traditionally are low-overhead.  From an error-handling perspective, it is rare that a program makes
use of such stack traces for programmatic handling, aside from better informing developers and operators.

Go 1.13 will change things with the introduction of golang.org/x/exp/errors.

TODO: show initial call stack for root error
TODO: show call stack for wrapped error with root error

### Error-Is Matching Semantics

TODO: rework sentinel matching here

### Error-As Matching Semantics

TODO: rework type/interface matching here

## Documentation

While Go does not have checked exceptions, it is typically regarded as a courtesy to API callers to document which types of errors your API could return and under what conditions.

With a sentinel error value:

```go
package sensitive

// PerformSensitiveAction does ... It returns UnauthorizedErr f the caller is
// not permitted to perform the action.
func PerformSensitiveAction() error { /* omitted */}

var UnauthorizedErr = errors.New("sensitive: the user is not authorized.")
```

With an error type:

```go
package auditlog

// AuthorizationError describes a failure case in which the user is not
// permitted to perform an operation.
type AuthorizationError struct{
  Operation string
}

func (err *AuthorizationError) Error() string {
  return fmt.Sprintf("auditlog: %v is not permitted", err.Operation)
}

// Purge clears the audit log. It returns an *AuthorizationError if the call
// is not permitted.
func Purge() error {
  if unauthorized {
    return &AuthorizationError{"Purge"}
  }
  // Rest omitted.
}

// Pause stops audit logging. It returns an *AuthorizationError if the call
// is not permitted.
func Pause() error {
  if unauthorized {
    return &AuthorizationError{"Pause"}
  }
  // Rest omitted.
}

// Resume starts audit logging. It returns an *AuthorizationError if the call
// is not permitted.
func Resume() error {
  if unauthorized {
    return &AuthorizationError{"Resume"}
  }
  // Rest omitted.
}
```

## Identifier Visibility
Like Java, Go has a concept of identifier visibility: exported and unexported.
Take care to ensure that your API's callers can access your error (sentinel)
values and types as necessary.

```go
// Package purchaserecord provides a high-level datastore around user purchase
// events. Integrators should prefer using this package than directly accessing
// the database without this API.
package purchaserecord

type DB struct{ /* rest omitted */ }

type LedgerInconsistencyError struct{
  Event *PurchaseEvent
}

// LoadRecords loads and summarizes the account's purchase events. It returns a
// *LedgerInconsistencyError if it detects a bookkeeping problem.
func (db *DB) LoadRecords(accountId int) (*Summary, error) { /* rest omitted */ }
```

Suppose instead of `LedgerInconsistencyError` the package had made it
`ledgerInconsistencyError`, where the error type is unexported (thusly
"private" from integrators' standpoing)? Callers could not handle this
programmatically.

## Checked Exceptions
Go has no conception of checked exceptions. It is unclear whether anything more
needs to be said about that.

## Runtime Exceptions, Panics, and Broken Invariants
### Removing or Reducing Possibility for Errors
It is best to design your API such that certain classes of problems are solved
statically.  This prevents the need for entrance precondition checking.  For
example:

```java
class Auditor {
  // Disable user instantiation.
  private Auditor() {}

  public void record(String userName, String operation) { /* rest omitted */ }
  
  // Creates an Auditor that emits to the provided stream.
  public static Auditor onStream(OutputStream stream) { /* rest omitted */ }
  
  // Creates an Auditor that does nothing.
  public static Auditor noop() { /* rest omitted */ }
  
  // Creates an Auditor that emits records to the usual location.
  public static Auditor newDefault() { /* rest omitted */ }
}
```

```go
package auditor

type Auditor struct{ /* rest omitted */ }

func (auditor *Auditor) Record(userName, operation string) { /* rest omitted */ }

// As an analogue for "newDefault":
//
// Choice 1: Give Auditor a useful zero value implementation.  User creates a new auditor as such:
//
// var auditor Auditor
//
// Choice 2: Create a construction function (usually if zero value does not suffice).

// New creates a new Auditor using the default configuration.
func New() *Auditor { /* rest omitted */ }

// NewWriter creates a new Auditor using the provided writer.
func NewWriter(w io.Writer) *Auditor { /* rest omitted */ }

// Noop is an auditor that does nothing.
var Noop = NewWriter(ioutil.DiscardWriter)
```

With these specific APIs, numerous input precondition validation can be removed.

### When to Panic

