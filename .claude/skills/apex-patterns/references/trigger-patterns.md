---
title: Trigger Handler Patterns Reference
description: Rules and implementation guide for trigger patterns in Apex — TriggerHandler, Filter, Enricher, and Validator — keeping triggers thin and business logic organized in dedicated classes.
tags: [apex, trigger, trigger-handler, filter, enricher, validator, pattern]
load-when: Implementing or reviewing trigger handlers, filters, enrichers, validators, or any Apex trigger logic
---

# Trigger Patterns

## TriggerHandler

### Why Use TriggerHandler

- Separates business logic from the trigger declaration
- Guarantees execution order across trigger contexts
- Prevents recursion with a built-in loop counter
- Easy to test independently from the trigger
- Reusable and maintainable across the org

### Trigger Entry Point

Every trigger delegates to a handler via `run()`. One trigger, one handler:

```apex
trigger AccountTrigger on Account (before insert, before update, after insert, after update) {
    new AccountTriggerHandler().run();
}
```

### Extending TriggerHandler

Create one handler class per SObject. Use a secondary constructor that accepts trigger collections directly — this enables unit testing without a live trigger context.

```apex
public with sharing class AccountTriggerHandler extends TriggerHandler {

    List<Account> newAccounts;
    Map<Id, Account> oldAccounts;

    AccountValidator validator;
    AccountEnricher enricher;
    AccountFilter filter;

    public AccountTriggerHandler() {
        this((List<Account>) Trigger.new,
             (Map<Id, Account>) Trigger.oldMap);
    }

    public AccountTriggerHandler(List<Account> newAccounts, Map<Id, Account> oldAccounts) {
        this.newAccounts = newAccounts;
        this.oldAccounts = oldAccounts;
        this.filter = new AccountFilter();
        this.enricher = new AccountEnricher();
        this.validator = new AccountValidator();
    }

    override
    public void beforeInsert() {
        enrich();
    }

    override
    public void beforeUpdate() {
        validate();
    }

    override
    public void afterInsert() {
        createFirstContactTask();
    }

    private void enrich() {
        enricher.enrich(newAccounts);
    }

    private void validate() {
        List<Account> changedAccounts = filter.byChangedRatingToHot(newAccounts, oldAccounts);
        validator.validate(changedAccounts);
    }

    private void createFirstContactTask() {
        enricher.createFirstContactTask(newAccounts);
    }
}
```

### Overridable Context Methods

Override only the contexts you need — all are no-ops by default:

```apex
protected virtual void beforeInsert()   { }
protected virtual void beforeUpdate()   { }
protected virtual void beforeDelete()   { }
protected virtual void afterInsert()    { }
protected virtual void afterUpdate()    { }
protected virtual void afterDelete()    { }
protected virtual void afterUndelete()  { }
```

### Recursion Control: `setMaxLoopCount`

Prevents infinite loops when handler DML re-fires the same trigger. Call `setMaxLoopCount` in the constructor:

```apex
public AccountTriggerHandler() {
    setMaxLoopCount(2); // max 2 executions per transaction
}
```

If the limit is exceeded, the framework throws `TriggerHandlerException` with a message identifying the handler and the max count reached.

### Bypass

Temporarily disables a specific handler. Useful in batch jobs, data migrations, and unit tests where trigger logic must not run:

```apex
TriggerHandler.bypass('AccountTriggerHandler');      // disable
TriggerHandler.clearBypass('AccountTriggerHandler'); // re-enable
TriggerHandler.isBypassed('AccountTriggerHandler');  // check
TriggerHandler.clearAllBypasses();                   // re-enable all
```

`handlerName` is the exact class name that extends `TriggerHandler`.

### Testing with Bypass

```apex
@IsTest
private class AccountTriggerHandlerTest {

    @IsTest
    static void testInsertWithoutHandler() {
        TriggerHandler.bypass('AccountTriggerHandler');

        Account account = new Account(Name = 'No trigger logic');
        insert account;

        System.assertEquals('No trigger logic', account.Name);
        TriggerHandler.clearBypass('AccountTriggerHandler');
    }

    @IsTest
    static void testInsertWithHandler() {
        TriggerHandler.clearBypass('AccountTriggerHandler');

        Account account = new Account(Name = 'With trigger');
        insert account;

        account = [SELECT Description FROM Account WHERE Id = :a.Id];
        System.assertEquals('Created via TriggerHandler', account.Description);
    }
}
```

### Testing by Triggering DML Directly

The recommended approach — insert the record and assert the side effects:

```apex
@IsTest
private class EventQueueTriggerHandlerTest {

    @IsTest
    static void shouldAutoSchedule() {
        EventQueue__c event = EventQueueFixtureFactory.newEvent('AccountCreated');

        insert event; // fires trigger -> EventQueueTriggerHandler.run()

        // assert side effects from beforeInsert / afterInsert logic
    }
}
```

---

## Filter

### Role of Filter

A Filter class encapsulates collection-filtering logic for a specific SObject domain, keeping handlers free of inline filtering detail. The base `Filter` class provides generic field-comparison methods. Subclasses add domain-specific methods whose names express business intent.

### Extending the Base Filter

Create one Filter subclass per SObject:

```apex
public class AccountFilter extends Filter {

    public List<Account> byChangedRatingToHot(List<Account> newAccounts, Map<Id, Account> oldAccounts) {
        return (List<Account>) byChangedValue(newAccounts, oldAccounts, 'Rating', 'Hot');
    }
}
```

### Base Filter Methods

| Method | Description |
|---|---|
| `byChangedValue(new, old, field, value)` | Records where a field changed to a specific value |
| `byChangedFields(new, old, fields)` | Records where any listed field changed |
| `isChanged(changed, old, fields)` | `true` if any field differs between old and new |
| `byValue(records, field, value)` | Records matching a specific field value |
| `byNotNull(records, field)` | Records where field is not null |
| `byNull(records, field)` | Records where field is null |
| `byNotEmpty(records, field)` | Records where string field is not blank |
| `byEmpty(records, field)` | Records where string field is blank |
| `withError(records)` | Records that have errors |
| `hasError(records)` | `true` if any record has errors |

### Example: Using a Domain Filter in a Handler

```apex
ContractFilter filter = new ContractFilter();

// contracts whose status just changed to "Signed"
List<Contract> amendmentContracts = filter.byChangedToSignedStatus(newContracts, oldContracts);
```

---

## Enricher

### Role of Enricher

An Enricher encapsulates the complete data-enrichment flow for a set of records. It coordinates filters, repositories, and business rules — returning the enriched objects and persisting when appropriate. This keeps handlers free of implementation detail.

The class and method names together express the business intent:
- `InactivateOriginalContractEnricher.inactivatedBy(signedContracts)`
- `AccountContractEnricher.enrichWithTotalPendingContracts(contracts)`

### Example: InactivateOriginalContractEnricher

```java
public class InactivateOriginalContractEnricher {

    ContractRepository repository;
    ContractFilter filter;

    public InactivateOriginalContractEnricher() {
        this.repository = new ContractRepository();
        this.filter = new ContractFilter();
    }

    public List<Contract> inactivatedBy(List<Contract> signedContracts) {
        List<Contract> originalContracts = findOriginalContracts(signedContracts);

        if (originalContracts.isEmpty()) {
            return new List<Contract>();
        }

        List<Contract> inactivatedContracts = inactivateOriginalContracts(originalContracts, signedContracts);

        return repository.save(inactivatedContracts);
    }

    private List<Contract> findOriginalContracts(List<Contract> signedContracts) {
        List<String> originalContractIds = filter.extractOriginalContractIds(signedContracts);
        return repository.findByIds(originalContractIds);
    }

    private List<Contract> inactivateOriginalContracts(
        List<Contract> originalContracts,
        List<Contract> signedContracts
    ) {
        Map<Id, Contract> indexedOriginalContracts = filter.indexById(originalContracts);

        for (Contract signedContract : signedContracts) {
            Contract originalContract = indexedOriginalContracts.get(signedContract.OriginalContract__c);
            originalContract.Status = 'Inactive';
        }

        return originalContracts;
    }
}
```

**Flow:**
- **Input**: `List<Contract> signedContracts` — amendment contracts just signed
- **Steps**: finds originals via repository, marks them inactive, persists via `repository.save()`
- **Output**: the inactivated original contracts

### Using an Enricher in a Handler

```java

public with sharing class ContractTriggerHandler extends TriggerHandler {
   
    // some code were omitted 

    public ContractTriggerHandler ( List<Contract> newContracts, Map<Id, Contract> oldContracts ) {

        this.newContracts = newContracts;
        this.oldContracts = oldContracts;         
        this.filter = new ContractFilter ();    
        this.inactivateOriginalContract = new InactivateOriginalContractEnricher();
                                   
    }

    override
    public void afterUpdate() {
        inactivateOldContracts ();
    }

    private void inactivateOldContracts () {
        // filter: contracts whose status changed to "Signed"
        List<Contract> amendmentContracts = filter.byChangedToAssignedStatus ( newContracts, oldContracts );
        
        // delegate enrichment
        inactivateOriginalContract.inactivatedBy ( amendmentContracts );

    }

```

**Pattern:**
1. Handler filters the relevant records using a Filter.
2. Handler delegates enrichment to an Enricher via an expressive method name.
3. Enricher handles all repository calls, calculations, and DML internally.

### Testing an Enricher

```java
Mocker mocker = Mocker.startStubbing();
ContractRepository mockedRepository = (ContractRepository) mocker.mock(ContractRepository.class);

// configure findByIds and save behavior...

InactivateOriginalContractEnricher enricher = new InactivateOriginalContractEnricher();
enricher.setRepository(mockedRepository);

List<Contract> inactivatedContracts = enricher.inactivatedBy(amendmentContracts);

// assert...
```

---

## Validator

### Role of Validator

A Validator encapsulates validation rules for an object or field, completely separated from triggers, enrichers, and repositories. Its only responsibility is "is this valid?" and "if not, flag the error."

Each Validator covers one specific type of validation:

| Validator | Responsibility |
|---|---|
| `PersonDocumentValidator` | Validates CPF format |
| `CompanyDocumentValidator` | Validates CNPJ format |
| `BrazilianDocumentValidator` | Delegates to CPF or CNPJ by document length |
| `AccountValidator` | Validates an Account record including its document number |

### Example 1 — Simple Value Validator (CPF)

```java
public class PersonDocumentValidator {
    private Integer MAX_SIZE = 11;

    public Boolean isCPF(String cpf) {
        // CPF validation logic
    }
}
```

Composing validators:

```apex
public class BrazilianDocumentValidator {
    private PersonDocumentValidator personValidator;
    private CompanyDocumentValidator companyValidator;

    public BrazilianDocumentValidator() {
        this.personValidator  = new PersonDocumentValidator();
        this.companyValidator = new CompanyDocumentValidator();
    }

    public Boolean isValid(String document) {
        if (String.isBlank(document)) return false;

        String unmasked = document.replaceAll('\\.|-|/', '');
        if (unmasked.length() <= 11) {
            return personValidator.isCPF(document);
        }
        return companyValidator.isCnpj(document);
    }
}
```

### Example 2 — Domain Object Validator (AccountValidator)

```java
public class AccountValidator {
    BrazilianDocumentValidator documentValidator;

    public AccountValidator() {
        this.documentValidator = new BrazilianDocumentValidator();
    }

    public void validate(List<Account> accounts) {
        for (Account account : accounts) {
            validateDocumentNumber(account);
        }
    }

    private void validateDocumentNumber(Account account) {
        if (this.documentValidator.isValid(account.DocumentNumber__c)) {
            return;
        }
        account.DocumentNumber__c.addError('Invalid document number');
    }
}
```

### Integrating Validator with TriggerHandler

Validators are called from within the standard `beforeInsert` / `beforeUpdate` context methods:

```apex
public with sharing class AccountTriggerHandler extends TriggerHandler {
    
    private AccountValidator validator;
        
    public AccountTriggerHandler(List<Account> newAccounts, Map<Id, Account> oldAccounts) {
        this.newAccounts = newAccounts;
        this.validator   = new AccountValidator();
    }

    override
    public void beforeInsert() {
        validate();
    }

    override
    public void beforeUpdate() {
        validate();
    }

    private void validate () {
        validator.validate(newAccounts);
    }
}
```

### Testing a Validator

Validators can be tested directly, without any trigger context:

```apex
@IsTest
private class PersonDocumentValidatorTest {

    @IsTest
    static void shouldAcceptValidCpf() {
        PersonDocumentValidator validator = new PersonDocumentValidator();

        System.assertTrue(validator.isCpf('331.099.440-65'), 'Should be a valid CPF');
        System.assertTrue(validator.isCpf('221.935.320-60'), 'Should be a valid CPF');
    }

    @IsTest
    static void shouldRejectSameDigits() {
        PersonDocumentValidator validator = new PersonDocumentValidator();
        String template = '###.###.###-##';

        for (Integer i = 0; i < 10; i++) {
            Boolean result = validator.isCpf(template.replaceAll('#', String.valueOf(i)));
            System.assertFalse(result, 'All-same-digit CPF should be invalid');
        }
    }
}
```

---

## Complete Handler Examples

### ContractTriggerHandler

```apex
public with sharing class ContractTriggerHandler extends TriggerHandler {

    List<Contract> newContracts;
    Map<Id, Contract> oldContracts;

    ContractRepository contractRepository;
    ContractFilter filter;
    InactivateOriginalContractEnricher inactivateOriginalContract;
    AccountContractEnricher accountEnricher;

    public ContractTriggerHandler() {
        this((List<Contract>) Trigger.new,
             (Map<Id, Contract>) Trigger.oldMap);
    }

    public ContractTriggerHandler(List<Contract> newContracts, Map<Id, Contract> oldContracts) {
        this.newContracts = newContracts;
        this.oldContracts = oldContracts;
        this.contractRepository         = new ContractRepository();
        this.filter                     = new ContractFilter();
        this.inactivateOriginalContract = new InactivateOriginalContractEnricher();
        this.accountEnricher            = new AccountContractEnricher();
    }

    override
    public void afterInsert() {
        enrichWithPendingContracts();
    }

    override
    public void afterUpdate() {
        inactivateOldContracts();
        enrichWithPendingContracts();
    }

    private void inactivateOldContracts() {
        List<Contract> amendmentContracts = filter.byChangedToAssignedStatus(newContracts, oldContracts);
        inactivateOriginalContract.inactivatedBy(amendmentContracts);
    }

    private void enrichWithPendingContracts() {
        accountEnricher.enrichWithTotalPendingContracts(newContracts);
    }
}
```

### AccountTriggerHandler

```apex
public with sharing class AccountTriggerHandler extends TriggerHandler {

    List<Account> newAccounts;
    Map<Id, Account> oldAccounts;

    AccountValidator validator;
    AccountEventEnricher enricher;

    public AccountTriggerHandler() {
        this((List<Account>) Trigger.new,
             (Map<Id, Account>) Trigger.oldMap);
    }

    public AccountTriggerHandler(List<Account> newAccounts, Map<Id, Account> oldAccounts) {
        this.newAccounts = newAccounts;
        this.oldAccounts = oldAccounts;
        this.validator = new AccountValidator();
        this.enricher  = new AccountEventEnricher();
    }

    override
    public void beforeInsert() {
        validate();
    }

    override
    public void beforeUpdate() {
        validate();
    }

    override
    public void afterInsert() {
        scheduleFirstAdvisorMeeting();
    }

    private void validate() {
        this.validator.validate(this.newAccounts);
    }

    private void scheduleFirstAdvisorMeeting() {
        this.enricher.scheduleFirstAdvisorMeeting(this.newAccounts);
    }
}
```

---

## Base Class Implementations

### TriggerHandler

Kevin O'Hara's trigger framework — the base class all handlers extend.

```apex
/**
 * @author kevinohara80
 * @see https://github.com/kevinohara80/sfdc-trigger-framework
 */
public virtual class TriggerHandler {

    private static Map<String, LoopCount> loopCountMap;
    private static Set<String> bypassedHandlers;

    @TestVisible
    private TriggerContext context;

    @TestVisible
    private Boolean isTriggerExecuting;

    static {
        loopCountMap     = new Map<String, LoopCount>();
        bypassedHandlers = new Set<String>();
    }

    public TriggerHandler() {
        this.setTriggerContext();
    }

    public void run() {

        if (!validateRun()) {
            return;
        }

        addToLoopCount();

        switch on this.context {
            when BEFORE_INSERT  { this.beforeInsert(); }
            when BEFORE_UPDATE  { this.beforeUpdate(); }
            when BEFORE_DELETE  { this.beforeDelete(); }
            when AFTER_INSERT   { this.afterInsert(); }
            when AFTER_UPDATE   { this.afterUpdate(); }
            when AFTER_DELETE   { this.afterDelete(); }
            when AFTER_UNDELETE { this.afterUndelete(); }
        }
    }

    public void setMaxLoopCount(Integer max) {
        String handlerName = getHandlerName();
        if (!TriggerHandler.loopCountMap.containsKey(handlerName)) {
            TriggerHandler.loopCountMap.put(handlerName, new LoopCount(max));
        } else {
            TriggerHandler.loopCountMap.get(handlerName).setMax(max);
        }
    }

    public void clearMaxLoopCount() {
        this.setMaxLoopCount(-1);
    }

    public static void bypass(String handlerName) {
        TriggerHandler.bypassedHandlers.add(handlerName);
    }

    public static void clearBypass(String handlerName) {
        TriggerHandler.bypassedHandlers.remove(handlerName);
    }

    public static Boolean isBypassed(String handlerName) {
        return TriggerHandler.bypassedHandlers.contains(handlerName);
    }

    public static void clearAllBypasses() {
        TriggerHandler.bypassedHandlers.clear();
    }

    @TestVisible
    private void setTriggerContext() {
        this.setTriggerContext(null, false);
    }

    @TestVisible
    private void setTriggerContext(String ctx, Boolean testMode) {

        if (!Trigger.isExecuting && !testMode) {
            this.isTriggerExecuting = false;
            return;
        } else {
            this.isTriggerExecuting = true;
        }

        if ((Trigger.isExecuting && Trigger.isBefore && Trigger.isInsert) ||
                (ctx != null && ctx == 'before insert')) {
            this.context = TriggerContext.BEFORE_INSERT;
        } else if ((Trigger.isExecuting && Trigger.isBefore && Trigger.isUpdate) ||
                (ctx != null && ctx == 'before update')) {
            this.context = TriggerContext.BEFORE_UPDATE;
        } else if ((Trigger.isExecuting && Trigger.isBefore && Trigger.isDelete) ||
                (ctx != null && ctx == 'before delete')) {
            this.context = TriggerContext.BEFORE_DELETE;
        } else if ((Trigger.isExecuting && Trigger.isAfter && Trigger.isInsert) ||
                (ctx != null && ctx == 'after insert')) {
            this.context = TriggerContext.AFTER_INSERT;
        } else if ((Trigger.isExecuting && Trigger.isAfter && Trigger.isUpdate) ||
                (ctx != null && ctx == 'after update')) {
            this.context = TriggerContext.AFTER_UPDATE;
        } else if ((Trigger.isExecuting && Trigger.isAfter && Trigger.isDelete) ||
                (ctx != null && ctx == 'after delete')) {
            this.context = TriggerContext.AFTER_DELETE;
        } else if ((Trigger.isExecuting && Trigger.isAfter && Trigger.isUndelete) ||
                (ctx != null && ctx == 'after undelete')) {
            this.context = TriggerContext.AFTER_UNDELETE;
        }
    }

    @TestVisible
    private void addToLoopCount() {
        String handlerName = getHandlerName();
        if (TriggerHandler.loopCountMap.containsKey(handlerName)) {
            Boolean exceeded = TriggerHandler.loopCountMap.get(handlerName).increment();
            if (exceeded) {
                Integer max = TriggerHandler.loopCountMap.get(handlerName).max;
                throw new TriggerHandlerException('Maximum loop count of ' + String.valueOf(max) + ' reached in ' + handlerName);
            }
        }
    }

    @TestVisible
    private Boolean validateRun() {
        if (!this.isTriggerExecuting || this.context == null) {
            throw new TriggerHandlerException('Trigger handler called outside of Trigger execution');
        }
        return !TriggerHandler.bypassedHandlers.contains(getHandlerName());
    }

    @TestVisible
    private String getHandlerName() {
        return String.valueOf(this).substring(0, String.valueOf(this).indexOf(':'));
    }

    @TestVisible protected virtual void beforeInsert()   { }
    @TestVisible protected virtual void beforeUpdate()   { }
    @TestVisible protected virtual void beforeDelete()   { }
    @TestVisible protected virtual void afterInsert()    { }
    @TestVisible protected virtual void afterUpdate()    { }
    @TestVisible protected virtual void afterDelete()    { }
    @TestVisible protected virtual void afterUndelete()  { }

    @TestVisible
    private class LoopCount {
        private Integer max;
        private Integer count;

        public LoopCount() {
            this.max   = 5;
            this.count = 0;
        }

        public LoopCount(Integer max) {
            this.max   = max;
            this.count = 0;
        }

        public Boolean increment() {
            this.count++;
            return this.exceeded();
        }

        public Boolean exceeded() {
            return this.max >= 0 && this.count > this.max;
        }

        public Integer getMax()   { return this.max; }
        public Integer getCount() { return this.count; }

        public void setMax(Integer max) {
            this.max = max;
        }
    }

    @TestVisible
    private enum TriggerContext {
        BEFORE_INSERT, BEFORE_UPDATE, BEFORE_DELETE,
        AFTER_INSERT, AFTER_UPDATE, AFTER_DELETE,
        AFTER_UNDELETE
    }

    public class TriggerHandlerException extends Exception {}
}
```

### Filter

Generic base class for all domain-specific Filter subclasses:

```apex
public virtual class Filter {

    @SuppressWarnings('PMD.ExcessiveParameterList')
    public List<SObject> byChangedValue(List<SObject> newRecords,
                                        Map<Id, SObject> oldRecords,
                                        String changedField,
                                        Object value) {
        List<SObject> changedRecords = byChangedFields(newRecords, oldRecords, new String[] { changedField });
        return byValue(changedRecords, changedField, value);
    }

    public List<SObject> byChangedFields(List<SObject> newRecords,
                                         Map<Id, SObject> oldRecords,
                                         List<String> changedFields) {
        List<SObject> changedRecords = new List<SObject>();

        if (newRecords == null || newRecords.isEmpty()) return changedRecords;
        if (oldRecords == null || oldRecords.isEmpty()) return newRecords;

        for (SObject record : newRecords) {
            SObject oldRecord = oldRecords.get((Id) record.get('Id'));
            if (isChanged(record, oldRecord, changedFields)) {
                changedRecords.add(record);
            }
        }

        return changedRecords;
    }

    public Boolean isChanged(SObject changed, SObject old, List<String> changedFields) {
        if (old == null) return true;
        for (String field : changedFields) {
            if (changed.get(field) != old.get(field)) return true;
        }
        return false;
    }

    public List<SObject> byValue(List<SObject> records, String fieldName, Object value) {
        List<SObject> result = new List<SObject>();
        for (SObject record : records) {
            Object recordValue = SObjects.getFieldValue(record, fieldName);
            if (recordValue == null) continue;
            if (recordValue == value) result.add(record);
        }
        return result;
    }

    public List<SObject> byNotNull(List<SObject> records, String fieldName) {
        List<SObject> filtered = new List<SObject>();
        for (SObject record : records) {
            if (SObjects.getFieldValue(record, fieldName) != null) filtered.add(record);
        }
        return filtered;
    }

    public List<SObject> byNull(List<SObject> records, String fieldName) {
        List<SObject> filtered = new List<SObject>();
        for (SObject record : records) {
            if (SObjects.getFieldValue(record, fieldName) == null) filtered.add(record);
        }
        return filtered;
    }

    public List<SObject> byNotEmpty(List<SObject> records, String fieldName) {
        List<SObject> filtered = new List<SObject>();
        for (SObject record : records) {
            String value = (String) SObjects.getFieldValue(record, fieldName);
            if (!String.isEmpty(value)) filtered.add(record);
        }
        return filtered;
    }

    public List<SObject> byEmpty(List<SObject> records, String fieldName) {
        List<SObject> filtered = new List<SObject>();
        for (SObject record : records) {
            String value = (String) SObjects.getFieldValue(record, fieldName);
            if (String.isEmpty(value)) filtered.add(record);
        }
        return filtered;
    }

    public List<SObject> withError(List<SObject> records) {
        List<SObject> filtered = new List<SObject>();
        for (SObject record : records) {
            if (record.hasErrors()) filtered.add(record);
        }
        return filtered;
    }

    public Boolean hasError(List<SObject> records) {
        for (SObject record : records) {
            if (record.hasErrors()) return true;
        }
        return false;
    }
}
```

### Lists

Null-safe collection utility used throughout the framework:

```apex
public class Lists {

    public static List<String> toIds(List<SObject> records) {
        return pullField(records, 'Id');
    }

    public static Set<Id> toSetIds(List<SObject> records) {
        return toSetIds(toIds(records));
    }

    public static Set<Id> toSetIds(List<String> ids) {
        Set<Id> filtered = new Set<Id>();
        for (String id : ids) {
            filtered.add(id);
        }
        return filtered;
    }

    public static List<String> pullField(List<SObject> records, String fieldName) {
        if (isEmpty(records)) return new List<String>();

        Set<String> filtered = new Set<String>();
        for (SObject record : records) {
            Object fieldValue = SObjects.getFieldValue(record, fieldName);
            if (fieldValue == null) continue;
            filtered.add('' + fieldValue);
        }
        return new List<String>(filtered);
    }

    public static List<SObject> pullAllSubQueryRecords(List<SObject> records, String subQueryName) {
        if (isEmpty(records)) return new List<SObject>();

        List<SObject> allRecords;
        for (SObject record : records) {
            if (record.getSObjects(subQueryName) == null) continue;
            List<SObject> subRecords = record.getSObjects(subQueryName);
            if (subRecords.isEmpty()) continue;
            if (allRecords == null) allRecords = SObjects.newEmptyList(subRecords);
            allRecords.addAll(subRecords);
        }
        return allRecords;
    }

    public static Boolean isEmpty(List<SObject> records) {
        return records == null || records.isEmpty();
    }

    public static Boolean isEmpty(List<Object> records) {
        return records == null || records.isEmpty();
    }

    public static Boolean isNotEmpty(List<Object> records) {
        return !isEmpty(records);
    }

    public static List<Object> removeDuplicateLines(List<Object> records) {
        return new List<Object>(new Set<Object>(records));
    }

    public static List<Object> removeEmptyLines(List<Object> records) {
        Set<Object> recordsSet = new Set<Object>(records);
        recordsSet.remove('');
        recordsSet.remove(null);
        return new List<Object>(recordsSet);
    }

    public static SObject getFirstOrNull(List<SObject> records) {
        return isNotEmpty(records) ? records.get(0) : null;
    }
}
```

### Maps

Collection-indexing and grouping utility for SObjects:

```apex
public class Maps {

    public static Map<String, SObject> indexBy(String fieldName, List<SObject> records) {
        if (Lists.isEmpty(records)) return new Map<String, SObject>();

        Map<String, SObject> values = SObjects.newEmptyMap(records);

        for (SObject record : records) {
            String value = (String) SObjects.getFieldValue(record, fieldName);
            if (value == null || values.containsKey(value)) continue;
            values.put(value, record);
        }

        return values;
    }

    public static Map<String, List<SObject>> groupBy(String fieldName, List<SObject> records) {
        if (Lists.isEmpty(records)) return new Map<String, List<SObject>>();

        Map<String, List<SObject>> groupedRecords = SObjects.newEmptyGroupedMap(records);
        String objectName = SObjects.getObjectName(records);

        for (SObject record : records) {
            String value = (String) SObjects.getFieldValue(record, fieldName);
            if (String.isEmpty(value)) continue;
            if (groupedRecords.get(value) == null) groupedRecords.put(value, SObjects.newEmptyList(objectName));
            groupedRecords.get(value).add(record);
        }

        return groupedRecords;
    }
}
```
