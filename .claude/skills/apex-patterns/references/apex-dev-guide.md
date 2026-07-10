---
title: Apex Clean Code Guide
description: Naming conventions, formatting rules, method and class design, error handling, and other Clean Code practices for Salesforce Apex development.
tags: [apex, clean-code, naming, formatting, error-handling, solid]
load-when: Coding conventions, naming decisions, method/class design, formatting, or code review
---

# Apex Clean Code Guide

## Naming Conventions

Good names communicate intent — a reader should never need to check the implementation to understand what a name means.

### No Acronyms — Applies to All Names

**Never use acronyms, initialisms, or abbreviated words in class names, method names, or variables.** Full words cost nothing in Apex and make the code self-documenting.

```java
// Good — full words, clear intent
public class OpportunityService { }
public class AccountRepository extends AbstractRepository { }
public Decimal calculateTotalAmount(List<Order__c> orders) { }

List<Opportunity> opportunities = findByStage('Proposal');
Decimal totalAmount = calculateTotalAmount(orders);
Integer retryCount = 0;

// Bad — acronyms and abbreviations hide meaning
public class OppSvc { }
public class AccRepo extends AbstractRepository { }
public Decimal calcTotalAmt(List<Order__c> orders) { }

List<Opportunity> opps = findByStage('Proposal');
Decimal totAmt = calcTotalAmt(orders);
Integer retryCnt = 0;
```

Common offenders to eliminate at every level:

| Avoid | Use instead |
|-------|-------------|
| `Svc` / `Srvc` | `Service` |
| `Mgr` | `Manager` *(or better: a specific role class)* |
| `Repo` / `Rep` | `Repository` |
| `Ctrl` / `Ctlr` | `Controller` |
| `Hlpr` / `Hlp` | `Helper` |
| `Util` / `Utils` | `Utility` *(or a specific class name)* |
| `calc` / `Calc` | `calculate` |
| `amt` | `amount` |
| `qty` | `quantity` |
| `cnt` / `ctr` | `count` |
| `idx` | `index` |
| `cfg` | `configuration` |
| `ctx` | `context` |
| `tmp` / `temp` | `temporary` or a descriptive name |
| `val` | `value` or `validator` |
| `res` | `result` or `response` |
| `req` | `request` |
| `msg` | `message` |
| `err` | `error` |
| `info` | `information` *(or a concrete noun)* |
| `param` | `parameter` *(or the actual parameter name)* |
| `obj` | the actual type name |
| `num` / `no` | `number` |

> **Rule of thumb:** if you would not say the abbreviation aloud in a conversation ("I need the opp repo"), don't use it in code.

### Classes

Use **PascalCase**. The name should describe what the class *is* or what it *does*. Include the pattern suffix when the class implements a known pattern.

```java
// Good — intent is clear, pattern suffix included
public inherited sharing class AccountEnricher extends Enricher { }
public inherited sharing class ContractRepository extends AbstractRepository { }
public inherited sharing class DocumentValidator extends Validator { }

// Bad — generic, no context
public class Helper { }
public class Manager { }
public class Util { }
```

### Methods

Use **camelCase**. The name should describe the *action* (verb first) and its subject.

Query methods must be prefixed with `findBy`. Boolean methods must be prefixed with `is`, `has`, or `can`.

```java
// Good
public List<Account> findByStatus(String status) { }
public Boolean isEligibleForDiscount(Account account) { }
public void applyDiscount(List<Order__c> orders) { }

// Bad — no verb, no subject, ambiguous
public List<Account> accounts() { }
public Boolean check(Account account) { }
public void process(List<Order__c> orders) { }
```

### Variables

Use **camelCase**. Use full, descriptive words — never single letters or cryptic abbreviations. Loop variables that represent a domain object should be named after that object.

```java
// Good
List<Account> activeAccounts = findByStatus('Active');
for (Account account : activeAccounts) {
    account.Description = 'Reviewed';
}

// Bad — abbreviations and letters convey nothing
List<Account> lst = findByStatus('Active');
for (Account a : lst) {
    a.Description = 'Reviewed';
}
```

**Never use common Salesforce abbreviations.** These are the most frequent offenders:

| Avoid | Use instead |
|-------|-------------|
| `repo` | `repository` |
| `opp` | `opportunity` |
| `acc` | `account` |
| `con` / `cont` | `contact` |
| `prod` | `product` |
| `rec` | `record` |
| `msg` | `message` |
| `val` | `value` or `validator` |
| `res` | `result` or `response` |
| `req` | `request` |

```java
// Good
AccountRepository repository = new AccountRepository();
for (Opportunity opportunity : opportunities) {
    opportunity.StageName = 'Closed Won';
}

// Bad
AccountRepository repo = new AccountRepository();
for (Opportunity opp : opportunities) {
    opp.StageName = 'Closed Won';
}
```

Avoid encoding type information in the name.

```java
// Good
List<Account> accounts;
Map<Id, Contact> contactsById;

// Bad — type suffix is redundant noise
List<Account> accountList;
Map<Id, Contact> contactMap;
```

### Collections

Name collections with the **plural of the element type**. For maps, encode the key in the name using the `ByKey` suffix.

```java
// Good
List<Contact> contacts;
Map<Id, Account> accountsById;
Map<String, List<Order__c>> ordersByStatus;

// Bad — generic, says nothing about contents
List<Contact> items;
Map<Id, Account> data;
Map<String, List<Order__c>> groupedOrders;
```

---

## Method Design

### Single Responsibility

Every method must do exactly one thing. If you need the word "and" or "or" to describe what a method does, it is doing too much.

```java
// Good — two focused methods
public void validateDocument(Account account) {
    if (String.isBlank(account.DocumentNumber__c)) {
        account.addError('Document number is required');
    }
}

public void enrichWithSegment(Account account) {
    account.Segment__c = resolveSegment(account.Revenue__c);
}

// Bad — validation and enrichment mixed in one method
public void validateAndEnrich(Account account) {
    if (String.isBlank(account.DocumentNumber__c)) {
        account.addError('Document number is required');
    }
    account.Segment__c = resolveSegment(account.Revenue__c);
}
```

### Method Size

A method should fit on a single screen (~20 lines). When a method grows beyond that, extract the sub-steps into private helper methods with descriptive names.

```java
// Good — top-level method reads like a table of contents
public void process(List<Opportunity> opportunities) {
    List<Opportunity> eligible = filterEligible(opportunities);
    List<Quote__c> quotes = buildQuotes(eligible);
    notifyOwners(eligible);
    repository.save(quotes);
}

// Bad — all logic inlined, impossible to scan
public void process(List<Opportunity> opportunities) {
    List<Opportunity> eligible = new List<Opportunity>();
    for (Opportunity opportunity : opportunities) {
        if (opportunity.StageName == 'Proposal' && opportunity.Amount > 10000) {
            eligible.add(opportunity);
        }
    }
    List<Quote__c> quotes = new List<Quote__c>();
    for (Opportunity opportunity : eligible) {
        Quote__c quote = new Quote__c();
        quote.Opportunity__c = opportunity.Id;
        quote.ExpirationDate__c = Date.today().addDays(30);
        quotes.add(quote);
    }
    // ... 30 more lines
}
```

### Guard Clauses (Early Return)

Return early for invalid or empty inputs instead of wrapping the entire method body in an `if` block. This eliminates nesting and makes the happy path easy to read.

```java
// Good — guard at the top, no extra indentation
public void applyDiscount(List<Order__c> orders, Decimal rate) {
    if (orders == null || orders.isEmpty()) return;
    if (rate == null || rate <= 0) return;

    for (Order__c order : orders) {
        order.DiscountRate__c = rate;
    }
}

// Bad — nested, happy path buried inside conditions
public void applyDiscount(List<Order__c> orders, Decimal rate) {
    if (orders != null && !orders.isEmpty()) {
        if (rate != null && rate > 0) {
            for (Order__c order : orders) {
                order.DiscountRate__c = rate;
            }
        }
    }
}
```

### Avoid Boolean Parameters

Boolean parameters are a signal that the method is doing two different things. Split it into two methods.

```java
// Good
public void sendWelcomeEmail(Contact contact) { }
public void sendReminderEmail(Contact contact) { }

// Bad — caller must know what `true` means
public void sendEmail(Contact contact, Boolean isWelcome) { }
```

---

## Class Design

### Class Size

A class that grows beyond ~200 lines is likely violating the Single Responsibility Principle. When a class becomes large, look for cohesive groups of methods that could be extracted into their own class.

```java
// Good — each class has one clear purpose
public inherited sharing class AccountService { }     // orchestration
public inherited sharing class AccountRepository extends AbstractRepository { }  // data access
public inherited sharing class AccountValidator extends Validator { }  // validation

// Bad — one class owns everything
public class AccountManager {
    // query methods
    // validation methods
    // business logic
    // email sending
    // reporting
}
```

### Cohesion

All methods in a class should operate on the same data and serve the same purpose. If a method does not use any of the class's fields, it may belong somewhere else.

---

## Formatting

### Indentation

Use **4 spaces** per indentation level. Never use tabs.

```java
// Good — 4-space indent
public void process(List<Account> accounts) {
    for (Account account : accounts) {
        if (account.IsActive__c) {
            account.Description = 'Active';
        }
    }
}

// Bad — 2-space indent
public void process(List<Account> accounts) {
  for (Account account : accounts) {
    if (account.IsActive__c) {
      account.Description = 'Active';
    }
  }
}
```

### Avoid Vertical Sprawl

Do not add blank lines between every statement. Blank lines should separate *logical groups*, not individual lines. Excessive vertical space forces the reader to scroll unnecessarily.

```java
// Good — blank line separates the guard from the main logic
public List<Quote__c> buildQuotes(List<Opportunity> opportunities) {
    if (opportunities == null) return new List<Quote__c>();

    List<Quote__c> quotes = new List<Quote__c>();
    for (Opportunity opportunity : opportunities) {
        quotes.add(buildQuote(opportunity));
    }
    return quotes;
}

// Bad — blank line after every statement
public List<Quote__c> buildQuotes(List<Opportunity> opportunities) {

    if (opportunities == null) return new List<Quote__c>();

    List<Quote__c> quotes = new List<Quote__c>();

    for (Opportunity opportunity : opportunities) {

        quotes.add(buildQuote(opportunity));

    }

    return quotes;

}
```

### Avoid Horizontal Alignment

Do not pad variable declarations or assignments with extra spaces to align names into columns. Alignment creates visual noise, breaks with every rename, and trains the eye to scan columns instead of reading code.

```java
// Good — single space between type and name
AccountRepository accountRepository;
ProductRepository productRepository;
PriceReferenceItemRepository priceRefItemRepository;
ApprovedVolumePremierRepository approvedVolRepository;
PriceReferenceConsumptionRepository consumptionRepository;

// Bad — padded to align names
AccountRepository                   accountRepository;
ProductRepository                   productRepository;
PriceReferenceItemRepository        priceRefItemRepository;
ApprovedVolumePremierRepository     approvedVolRepository;
PriceReferenceConsumptionRepository consumptionRepository;
```

### Method Declarations

Place the access modifier and the `virtual` keyword on separate lines when both are present, to make the method boundary easy to scan.

```java
// Good
virtual
public List<Account> findByStatus(String status) {
    return [SELECT Id, Name FROM Account WHERE Status__c = :status];
}

// Acceptable for non-virtual methods
public List<Account> findByStatus(String status) {
    return [SELECT Id, Name FROM Account WHERE Status__c = :status];
}
```

### Line Length

Keep lines under **100 characters**. Break long method calls or SOQL queries across lines aligned to the opening delimiter.

```java
// Good — query broken for readability
List<Contact> contacts = [
    SELECT Id, Name, Email, AccountId
    FROM Contact
    WHERE AccountId IN :accountIds
        AND IsActive__c = true
    ORDER BY Name
];

// Bad — unreadable horizontal scroll
List<Contact> contacts = [SELECT Id, Name, Email, AccountId FROM Contact WHERE AccountId IN :accountIds AND IsActive__c = true ORDER BY Name];
```

---

## Error Handling

### Throw Specific Exceptions

Never swallow exceptions or catch `Exception` without re-throwing or logging. Use or extend specific exception types so callers know what went wrong.

```java
// Good — custom exception with a clear message
public class DocumentValidationException extends Exception { }

public void validate(Account account) {
    if (String.isBlank(account.DocumentNumber__c)) {
        throw new DocumentValidationException('Document number is required for account: ' + account.Name);
    }
}

// Bad — generic catch hides the root cause
try {
    validate(account);
} catch (Exception e) {
    // silently ignored
}
```

### Prefer `addError` Over Exceptions in Trigger Context

Inside trigger handlers, use `addError()` on the record rather than throwing exceptions. This integrates with Salesforce's standard validation error UI.

```java
// Good — user sees a field-level or record-level error
public void validate(List<Account> accounts) {
    for (Account account : accounts) {
        if (String.isBlank(account.DocumentNumber__c)) {
            account.DocumentNumber__c.addError('Document number is required');
        }
    }
}

// Bad — throws a generic DML exception visible as a raw stack trace
public void validate(List<Account> accounts) {
    for (Account account : accounts) {
        if (String.isBlank(account.DocumentNumber__c)) {
            throw new AuraHandledException('Document number is required');
        }
    }
}
```

### Validate at the Boundary

Validate inputs at the entry point of a method rather than deep inside a call chain. A null check buried three layers down is much harder to diagnose than one at the method signature.

```java
// Good — fail fast at the public method boundary
public void applySegment(Account account) {
    if (account == null) throw new IllegalArgumentException('Account must not be null');
    account.Segment__c = resolveSegment(account.Revenue__c);
}

// Bad — null is discovered deep inside a private helper
private String resolveSegment(Decimal revenue) {
    // NullPointerException thrown here, origin is hard to trace
}
```

---

## Magic Values

Never embed literal strings, numbers, or IDs in logic. Extract them into named constants or Custom Metadata/Custom Labels.

```java
// Good — intent is explicit
private static final String STATUS_ACTIVE = 'Active';
private static final Integer MAX_RETRY_ATTEMPTS = 3;

public List<Account> findActive() {
    return [SELECT Id FROM Account WHERE Status__c = :STATUS_ACTIVE];
}

// Bad — meaning of 'Active' and 3 must be inferred from context
public List<Account> findActive() {
    return [SELECT Id FROM Account WHERE Status__c = 'Active'];
}

if (retryCount > 3) { ... }
```

---

## Comments

Write code that does not need comments. A comment that explains *what* the code does is a symptom of unclear naming. Only add a comment when it explains *why* — a hidden constraint, a non-obvious workaround, or a platform quirk.

```java
// Good — explains a non-obvious platform behavior
// Database.upsert requires a concrete SObject list type; a List<SObject> causes a runtime error
List<Account> typedList = (List<Account>) records;
Database.upsert(typedList, false);

// Bad — restates what the code already says
// Loop through accounts and set the status to Active
for (Account account : accounts) {
    account.Status__c = 'Active';
}
```

Never leave commented-out code in a committed file. Use version control history to recover removed code.

```java
// Bad — dead code with no context
// account.OldField__c = value;
// account.AnotherField__c = 'X';
account.Status__c = 'Active';
```
