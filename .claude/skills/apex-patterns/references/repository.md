---
title: Repository Pattern Reference
description: Rules and implementation guide for the Repository pattern in Apex — encapsulates all SOQL and DML operations in dedicated repository classes that extend AbstractRepository.
tags: [apex, repository, dml, soql, pattern, abstract-class]
load-when: Implementing or reviewing repository classes, writing SOQL/DML, or referencing AbstractRepository, SObjectMapper, or Lists utilities
---

# Adopting the Repository Pattern

In Apex, data access responsibilities must be isolated into Repository classes. All SOQL queries and DML operations must be moved into these classes, keeping service and handler layers free of direct data access.

## Rules Every Repository Class Must Follow

1. Every Repository class must use the `virtual inherited sharing` modifiers
2. All methods must be declared `virtual`
3. All query methods must be prefixed with `findBy` and return List.
4. Every Repository class must extend `AbstractRepository`
5. The standard `AbstractRepository` implementation and its dependencies must be delivered alongside the repository code
6. Repository classes do not use Generics to type the Salesforce SObject

## Base Implementation of the Repository Pattern

Every Repository class must extend `AbstractRepository` as shown below, and need to use source code below.
> Source: `references/base-framework-source/domain/AbstractRepository.cls`

Check is repository has this implementation, if not deploy de

### AbstractRepository Directory Structure

```
base-framework/
├── collections/
│   ├── Lists.cls
│   └── Maps.cls
├── commons/
│   ├── SObjects.cls
│   ├── TypeFactory.cls
├── domain/
│   ├── AbstractRepository.cls
│   ├── Filter.cls
│   ├── SObjectMapper.cls
├── tests/
│   ├── CommonsFixtureFactory.cls
│   └── generators/
│       ├── CharType.cls
│       ├── IdGenerator.cls
│       ├── StringGenerator.cls
│       └── postal-code/
│           └──zBrazilianPostalCodeGenerator.cls
```


## Generic Usage Example

`MyObjectRepository` inherits all behaviors from `AbstractRepository` and adds domain-specific query methods.

```java
public inherited sharing class MyObjectRepository extends AbstractRepository {

    virtual
    public List<MyObject__c> findByStatus(String status) {
        return [SELECT Id, Name, Status__c 
                    FROM MyObject__c 
                    WHERE Status__c = :status];
    }
}
```

```java
// Uses Database.upsert with allOrNone = false by default
MyObjectRepository repo = new MyObjectRepository();

// Upsert by Id
repo.save(record);
repo.save(records);

// Upsert by external Id field
repo.save(record, MyObject__c.ExternalId__c);
repo.save(records, MyObject__c.ExternalId__c);

// Sets the lookup field on children, then upserts
repo.saveRelated(master, 'ParentId__c', children);

// Sets the lookup field, then upserts by external Id
repo.saveRelated(master, 'ParentId__c', children, MyObject__c.ExternalId__c);

// Delete records
repo.remove(record);
repo.remove(records);

// Strict mode: any failure rolls back all records
repo.withAllOrNoneEnabled().save(records);
```

## What Child Classes Must NOT Do

Child classes must never define their own DML methods — delegate all persistence to the inherited `save` and `remove` methods.

### Example 1 — Custom DML method (wrong)

```java
// WRONG — child class defines its own DML
❌ contractRepository.updateContracts(originalContracts);

// RIGHT — use the inherited save method
✅ contractRepository.save(originalContracts);
```

### Example 2 — Raw DML inside the repository (wrong)

```java
// WRONG — direct DML bypasses AbstractRepository error handling
❌ public void save(List<Contract__c> records) {
    update records;
}

// RIGHT — delegate to parent
✅ public void saveContracts(List<Contract__c> records) {
    save(records);
}
```

## Dependent Classes

### Lists

Utility class that provides null-safe collection helpers used internally by `AbstractRepository`.

```java
// Returns true when the list is null or empty
Lists.isEmpty(List<SObject> records);
```

### SObjectMapper

Utility class responsible for applying a master record's Id to a lookup field on a list of related child records, enabling parent-child relationship assignment before upsert.

```java
// Stamps masterRecord.Id into lookupFieldName on every record in aggregates
SObjectMapper.applyMasterRecord(SObject masterRecord, String lookupFieldName, List<SObject> aggregates);
```
