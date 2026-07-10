---
name: apex-base-framework
description: Use when writing Apex code in the integration framework project — creating repositories, trigger handlers, logging, collection utilities, scheduling jobs, or using shared base-framework utilities like TypeFactory, SObjects, Filter, Lists, Maps, Environment, Scheduler, ContentVersionBuilder, or StackTraceCollector.
---

# Apex Base-Framework Reference

Framework location: `force-app/main/default/classes/base-framework/`

## Constraints

### MUST DO

- Always copy full content for used class

---

## Directory Structure

```
base-framework/
├── collections/
│   ├── Lists.cls
│   └── Maps.cls
├── commons/
│   ├── Environment.cls
│   ├── SObjects.cls
│   ├── StackTraceCollector.cls
│   ├── TypeFactory.cls
│   └── brazilian-document/
│       ├── BrazilianDocumentValidator.cls
│       ├── CompanyDocumentValidator.cls
│       └── PersonDocumentValidator.cls
├── domain/
│   ├── AbstractRepository.cls
│   ├── Filter.cls
│   ├── SObjectMapper.cls
│   └── content-document/
│       ├── ContentDocumentRepository.cls
│       └── builder/
│           └── ContentVersionBuilder.cls
├── log/
│   ├── DefaultLoggerAdapter.cls
│   ├── ILogger.cls
│   ├── Log.cls
│   └── Logger.cls
├── scheduler/
│   └── Scheduler.cls
├── tests/
│   ├── CommonsFixtureFactory.cls
│   └── generators/
│       ├── CharType.cls
│       ├── IdGenerator.cls
│       ├── StringGenerator.cls
│       └── postal-code/
│           └── BrazilianPostalCodeGenerator.cls
└── trigger-handler/
    └── TriggerHandler.cls
```

---

## Logger

> Source: `references/base-framework-source/log/Logger.cls` · `Log.cls` · `ILogger.cls` · `DefaultLoggerAdapter.cls`

Singleton. Use `Logger.getLogger()`. Never instantiate directly.

```apex
Logger log = Logger.getLogger();

log.log('message');
log.log('message', new Map<String, Object>{ 'key' => value });
log.debug('debug message');
log.warn('warn message');
log.error('error message');
log.error('error', ex);
log.error('error', ex, new Map<String, Object>{ 'context' => value });

// Fluent Log object
log.log(new Log()
    .level(LoggingLevel.INFO)
    .message('msg')
    .addParameter('key', value)
    .tag('MyClass'));
```

Custom adapter: implement `ILogger` and pass to `Logger.getLogger(myAdapter)`.

---

## Lists & Maps

> Source: `references/base-framework-source/collections/Lists.cls` · `Maps.cls`

### Lists

```apex
Lists.isEmpty(records)                              // null-safe empty check
Lists.isNotEmpty(records)
Lists.toIds(records)                                // → List<String> of Id
Lists.toSetIds(records)                             // → Set<Id>
Lists.pullField(records, 'FieldName__c')            // → List<String> of field values
Lists.pullAllSubQueryRecords(records, 'Children__r') // flatten sub-query
Lists.getFirstOrNull(records)                       // SObject or null
Lists.removeDuplicateLines(objects)
Lists.removeEmptyLines(objects)
```

### Maps

```apex
Map<String, SObject> byId = Maps.indexBy('Id', records);
Map<String, SObject> byExt = Maps.indexBy('ExternalId__c', records);
Map<String, List<SObject>> grouped = Maps.grouppBy('Status__c', records);
Maps.isEmpty(mapObject)  // null-safe
```

---

## SObjects

> Source: `references/base-framework-source/commons/SObjects.cls`

Schema and instance utilities. Supports dot-notation for relationship traversal.

```apex
SObjects.getObjectType('Account')             // Schema.SObjectType
SObjects.getObjectFields('Account')           // Map<String, Schema.SObjectField>
SObjects.getFieldValue(record, 'Owner.Name')  // traverses relationships
SObjects.getObjectName(record)                // 'Account'
SObjects.newInstance('Account')               // SObject
SObjects.newEmptyList('Account')              // List<Account>
SObjects.newEmptyList(existingList)           // same type as list
SObjects.newEmptyMap(existingList)            // Map<String, SObject>
SObjects.newEmptyGroupedMap(existingList)     // Map<String, List<SObject>>
```

---

## AbstractRepository

> Source: `references/base-framework-source/domain/AbstractRepository.cls`

Extend for any custom DML class. Uses `Database.upsert` with `allOrNone = false` by default.

```apex
public class MyObjectRepository extends AbstractRepository {
    public List<MyObject__c> findByStatus(String status) {
        return [SELECT Id FROM MyObject__c WHERE Status__c = :status];
    }
}

// Usage
MyObjectRepository repo = new MyObjectRepository();
repo.save(record);                          // upsert by Id
repo.save(records);
repo.save(record, MyObject__c.ExternalId__c); // upsert by external id
repo.save(records, MyObject__c.ExternalId__c);
repo.saveRelated(master, 'ParentId__c', children);  // sets lookup then upserts
repo.remove(record);
repo.remove(records);
repo.withAllOrNoneEnabled().save(records);  // strict mode
```

Errors are added to records via `addError()` — check with `Filter.hasError(records)`.

---

## Filter

> Source: `references/base-framework-source/domain/Filter.cls`

In-memory record filtering. Extend or use directly.

```apex
Filter filter = new Filter();

filter.byValue(records, 'Status__c', 'ACTIVE')
filter.byValues(records, 'Status__c', new List<Object>{'A','B'})
filter.byNull(records, 'Field__c')
filter.byNotNull(records, 'Field__c')
filter.byEmpty(records, 'Name')
filter.byNotEmpty(records, 'Name')
filter.byChangedFields(newRecords, oldRecords, new String[]{'Status__c'})
filter.byChangedValue(newRecords, oldRecords, 'Status__c', 'ACTIVE')
filter.withError(records)       // only records with addError() set
filter.hasError(records)        // boolean
```

---

## TriggerHandler

> Source: `references/base-framework-source/trigger-handler/TriggerHandler.cls`

Extend per trigger. Call `new MyHandler().run()` from the trigger file.

```apex
public class MyObjectTriggerHandler extends TriggerHandler {
    List<MyObject__c> newRecords;
    Map<Id, MyObject__c> oldMap;

    public MyObjectTriggerHandler() {
        this.newRecords = (List<MyObject__c>) Trigger.new;
        this.oldMap     = (Map<Id, MyObject__c>) Trigger.oldMap;
    }

    override protected void afterInsert() { /* ... */ }
    override protected void afterUpdate() { /* ... */ }
    override protected void beforeInsert() { /* ... */ }
}
```

Bypass control (use in tests or batch contexts):

```apex
TriggerHandler.bypass('MyObjectTriggerHandler');
TriggerHandler.clearBypass('MyObjectTriggerHandler');
TriggerHandler.clearAllBypasses();
```

Built-in `this.filter` (`Filter` instance) available in handler subclasses.

---

## TypeFactory

> Source: `references/base-framework-source/commons/TypeFactory.cls`

Dynamic class instantiation by name string.

```apex
IMyInterface impl = (IMyInterface) TypeFactory.newInstance('MyConcreteClass', IMyInterface.class);
// Returns null if className is blank, type not found, or not assignable.
```

---

## Environment

> Source: `references/base-framework-source/commons/Environment.cls`

Detects current org type from domain URL.

```apex
Environment env = Environment.get();
env.getType();    // EnvironmentType: DEVELOP | SCRATCH | SANDBOX | PRODUCTION
env.getOrgName(); // sandbox name or org name
env.getHost();    // full domain URL host
```

---

## Scheduler

> Source: `references/base-framework-source/scheduler/Scheduler.cls`

Wraps `System.schedule` with cron helpers.

```apex
Scheduler.scheduleIntoNextMinutes(new MyJob(), 5);  // 5 min from now
Scheduler.scheduleAtHour(new MyJob(), 2);            // daily at 2:00
Scheduler.schedule(new MyJob(), '0 0 * * * ?');      // raw cron
Scheduler.abort(schedulableContext);
Scheduler.abort(jobId);
```

---

## ContentVersionBuilder & ContentDocumentRepository

> Source: `references/base-framework-source/domain/content-document/builder/ContentVersionBuilder.cls` · `domain/content-document/ContentDocumentRepository.cls`

Attach files to records.

```apex
ContentVersion cv = ContentVersionBuilder.create()
    .withTitle('Log - ' + recordId)
    .withVersionData(jsonPayload)
    .withFirstPublishLocation(recordId)
    .build();
insert cv;

// Query
ContentDocumentRepository repo = new ContentDocumentRepository();
List<ContentVersion> versions = repo.findFirstPublishLocationIds(new List<String>{ recordId });
ContentVersion cv = repo.findVersionId(cvId);
```

---

## SObjectMapper

> Source: `references/base-framework-source/domain/SObjectMapper.cls`

Sets a lookup field from a master record on a list of children (skips if already set).

```apex
SObjectMapper.applyMasterRecord(masterRecord, 'ParentId__c', childRecords);
```

---

## StackTraceCollector

> Source: `references/base-framework-source/commons/StackTraceCollector.cls`

Captures runtime call stack (useful for logging utilities).

```apex
StackTraceCollector.StackTraceInfo frame = StackTraceCollector.getCurrentStack();
frame.apexClass  // 'MyClass'
frame.method     // 'myMethod'
frame.line       // 42
frame.column     // 12

List<StackTraceCollector.StackTraceInfo> stack = StackTraceCollector.getStackTrace();
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `new Logger()` | Use `Logger.getLogger()` — it's a singleton |
| Casting `Maps.indexBy` to `Map<String, MyObject__c>` | Keep as `Map<String, SObject>`, cast individual values |
| Using `Maps.grouppBy` — note the double-p | Correct spelling is `grouppBy` (typo in source) |
| Calling `repo.save()` and assuming exceptions on error | Errors go to `record.addError()` — check with `filter.hasError()` |
| Creating a new `Filter()` in a `TriggerHandler` subclass | Use `this.filter` — already instantiated by base class |
| `TypeFactory.newInstance` returning null silently | Always check for null; logs nothing on mismatch |