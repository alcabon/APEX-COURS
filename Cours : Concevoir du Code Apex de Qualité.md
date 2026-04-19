# Cours : Concevoir du Code Apex de Qualité
## Standards d'Ingénierie Logicielle Avancés pour Salesforce

---

> **À propos de ce cours**  
> Ce cours s'adresse aux développeurs Apex souhaitant élever leur niveau de maîtrise vers les standards d'ingénierie logicielle professionnels. Chaque module est illustré par des exemples concrets, des anti-patterns à éviter, et les patterns recommandés.

---

## Table des Matières

1. [Les Governor Limits : contrainte ou discipline ?](#module-1)
2. [Bulkification : penser en collections](#module-2)
3. [Architecture des Triggers : le Trigger Framework](#module-3)
4. [SOQL de qualité : Selector Layer](#module-4)
5. [DML de qualité : Unit of Work Pattern](#module-5)
6. [Principes SOLID appliqués à Apex](#module-6)
7. [Service Layer : la logique métier isolée](#module-7)
8. [Gestion des erreurs et exceptions](#module-8)
9. [Sécurité : CRUD, FLS et Sharing](#module-9)
10. [Apex Asynchrone : Future, Queueable, Batch, Schedulable](#module-10)
11. [Tests unitaires de qualité](#module-11)
12. [Conventions de nommage et lisibilité](#module-12)
13. [Architecture complète : fflib Apex Common](#module-13)

---

## Module 1 — Les Governor Limits : contrainte ou discipline ? {#module-1}

Les Governor Limits ne sont pas un obstacle : elles **imposent une discipline architecturale** que de nombreux langages ignorent, au détriment de la qualité du code.

### Limites synchrones critiques (à mémoriser)

| Limite | Valeur |
|--------|--------|
| Requêtes SOQL | 100 |
| Lignes retournées SOQL | 50 000 |
| Instructions DML | 150 |
| Enregistrements DML | 10 000 |
| Heap size | 6 MB |
| CPU time | 10 000 ms |
| Callouts HTTP | 100 |

### Règle d'or

> **Jamais de SOQL ou DML dans une boucle.**  
> Ce principe seul élimine 80% des violations de Governor Limits.

```apex
// ❌ ANTI-PATTERN — SOQL dans une boucle
for (Account acc : accounts) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
    // Violation : 1 SOQL par itération
}

// ✅ PATTERN CORRECT — Collecte puis requête unique
Set<Id> accountIds = new Map<Id, Account>(accounts).keySet();
Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();

for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
    if (!contactsByAccount.containsKey(c.AccountId)) {
        contactsByAccount.put(c.AccountId, new List<Contact>());
    }
    contactsByAccount.get(c.AccountId).add(c);
}
```

---

## Module 2 — Bulkification : penser en collections {#module-2}

La bulkification est le paradigme fondamental d'Apex. Tout code doit être conçu pour traiter **1 ou 1 000 000 enregistrements** avec la même efficacité.

### Principe : concevoir pour les ensembles

```apex
// ❌ ANTI-PATTERN — Méthode singleton (ne gère qu'un enregistrement)
public static void updateAccountRating(Account acc) {
    acc.Rating = 'Hot';
    update acc;
}

// ✅ PATTERN CORRECT — Méthode bulkifiée
public static void updateAccountRatings(List<Account> accounts) {
    for (Account acc : accounts) {
        acc.Rating = 'Hot';
    }
    update accounts;
}
```

### Pattern : Map pour la performance

```apex
public class OpportunityService {

    public static void linkContactRoles(List<Opportunity> opportunities) {
        // Étape 1 : Collecter les IDs en une passe
        Set<Id> accountIds = new Set<Id>();
        for (Opportunity opp : opportunities) {
            if (opp.AccountId != null) {
                accountIds.add(opp.AccountId);
            }
        }

        // Étape 2 : Une seule requête SOQL
        Map<Id, Account> accountMap = new Map<Id, Account>(
            [SELECT Id, OwnerId, Industry FROM Account WHERE Id IN :accountIds]
        );

        // Étape 3 : Traitement en mémoire
        List<OpportunityContactRole> rolesToInsert = new List<OpportunityContactRole>();
        for (Opportunity opp : opportunities) {
            if (accountMap.containsKey(opp.AccountId)) {
                Account acc = accountMap.get(opp.AccountId);
                // logique métier...
            }
        }

        // Étape 4 : Un seul DML
        if (!rolesToInsert.isEmpty()) {
            insert rolesToInsert;
        }
    }
}
```

---

## Module 3 — Architecture des Triggers : le Trigger Framework {#module-3}

### Le problème des triggers naïfs

```apex
// ❌ ANTI-PATTERN — Logique dans le trigger
trigger AccountTrigger on Account (before insert, before update, after insert) {
    if (Trigger.isBefore && Trigger.isInsert) {
        for (Account acc : Trigger.new) {
            acc.Description = 'Créé automatiquement';
            // 50 lignes de logique ici...
        }
    }
    // impossible à tester, à maintenir, à désactiver...
}
```

### Pattern : Trigger Handler avec délégation

#### Structure recommandée

```
force-app/
├── triggers/
│   └── AccountTrigger.trigger          // 3 lignes maximum
├── classes/
│   ├── TriggerHandler.cls              // classe abstraite de base
│   ├── AccountTriggerHandler.cls       // handler spécifique
│   └── AccountService.cls             // logique métier
```

#### TriggerHandler — Classe abstraite de base

```apex
public abstract class TriggerHandler {

    private static Map<String, LoopCount> loopCountMap = new Map<String, LoopCount>();
    private static Set<String> bypassedHandlers = new Set<String>();

    public TriggerHandler() {}

    public void run() {
        if (!validateRun()) {
            return;
        }
        addToLoopCount();

        switch on Trigger.operationType {
            when BEFORE_INSERT  { this.beforeInsert(); }
            when BEFORE_UPDATE  { this.beforeUpdate(); }
            when BEFORE_DELETE  { this.beforeDelete(); }
            when AFTER_INSERT   { this.afterInsert(); }
            when AFTER_UPDATE   { this.afterUpdate(); }
            when AFTER_DELETE   { this.afterDelete(); }
            when AFTER_UNDELETE { this.afterUndelete(); }
        }
    }

    // Méthodes virtuelles — surcharger selon le besoin
    protected virtual void beforeInsert()  {}
    protected virtual void beforeUpdate()  {}
    protected virtual void beforeDelete()  {}
    protected virtual void afterInsert()   {}
    protected virtual void afterUpdate()   {}
    protected virtual void afterDelete()   {}
    protected virtual void afterUndelete() {}

    // Contrôle de récursivité
    public static void bypass(String handlerName) {
        bypassedHandlers.add(handlerName);
    }

    public static void clearBypass(String handlerName) {
        bypassedHandlers.remove(handlerName);
    }

    private Boolean validateRun() {
        if (!Trigger.isExecuting) {
            throw new TriggerHandlerException('Trigger handler appelé hors contexte trigger.');
        }
        if (bypassedHandlers.contains(getHandlerName())) {
            return false;
        }
        return true;
    }

    private String getHandlerName() {
        return String.valueOf(this).substring(0, String.valueOf(this).indexOf(':'));
    }

    private void addToLoopCount() {
        String handlerName = getHandlerName();
        if (!loopCountMap.containsKey(handlerName)) {
            loopCountMap.put(handlerName, new LoopCount(5));
        }
        if (loopCountMap.get(handlerName).increment()) {
            throw new TriggerHandlerException('Récursivité détectée dans ' + handlerName);
        }
    }

    private class LoopCount {
        private Integer max;
        private Integer count = 0;
        public LoopCount(Integer max) { this.max = max; }
        public Boolean increment() { count++; return count > max; }
    }

    public class TriggerHandlerException extends Exception {}
}
```

#### AccountTrigger — Trigger épuré (3 lignes)

```apex
trigger AccountTrigger on Account (
    before insert, before update, after insert, after update, after delete
) {
    new AccountTriggerHandler().run();
}
```

#### AccountTriggerHandler — Handler spécifique

```apex
public class AccountTriggerHandler extends TriggerHandler {

    private List<Account> newRecords;
    private List<Account> oldRecords;
    private Map<Id, Account> newMap;
    private Map<Id, Account> oldMap;

    public AccountTriggerHandler() {
        this.newRecords = (List<Account>) Trigger.new;
        this.oldRecords = (List<Account>) Trigger.old;
        this.newMap    = (Map<Id, Account>) Trigger.newMap;
        this.oldMap    = (Map<Id, Account>) Trigger.oldMap;
    }

    protected override void beforeInsert() {
        AccountService.assignDefaultValues(newRecords);
    }

    protected override void afterInsert() {
        AccountService.createWelcomeTask(newRecords);
    }

    protected override void beforeUpdate() {
        AccountService.validateRatingChange(newRecords, oldMap);
    }
}
```

---

## Module 4 — SOQL de qualité : Selector Layer {#module-4}

### Le Pattern Selector

Le Selector centralise toutes les requêtes SOQL pour un SObject. Avantages : réutilisation, testabilité, cohérence des champs.

```apex
public class AccountSelector {

    // Champs de base toujours récupérés
    private static final Set<String> BASE_FIELDS = new Set<String>{
        'Id', 'Name', 'AccountNumber', 'Industry', 'Rating',
        'BillingCity', 'BillingCountry', 'OwnerId', 'CreatedDate'
    };

    // ---------------------------------------------------------------
    // Méthodes de requête — nomenclature : selectBy[Critère]
    // ---------------------------------------------------------------

    public List<Account> selectById(Set<Id> accountIds) {
        return [
            SELECT Id, Name, AccountNumber, Industry, Rating,
                   BillingCity, BillingCountry, OwnerId, CreatedDate
            FROM Account
            WHERE Id IN :accountIds
            WITH SECURITY_ENFORCED
            ORDER BY Name ASC
        ];
    }

    public List<Account> selectByIndustry(Set<String> industries) {
        return [
            SELECT Id, Name, Industry, Rating, OwnerId
            FROM Account
            WHERE Industry IN :industries
            AND IsDeleted = false
            WITH SECURITY_ENFORCED
            ORDER BY Name ASC
            LIMIT 2000
        ];
    }

    public Map<Id, Account> selectByIdAsMap(Set<Id> accountIds) {
        return new Map<Id, Account>(
            [SELECT Id, Name, Industry, Rating
             FROM Account
             WHERE Id IN :accountIds
             WITH SECURITY_ENFORCED]
        );
    }

    // Requête avec sous-requête (relationship query)
    public List<Account> selectWithContacts(Set<Id> accountIds) {
        return [
            SELECT Id, Name,
                (SELECT Id, FirstName, LastName, Email, Phone
                 FROM Contacts
                 WHERE IsDeleted = false
                 ORDER BY LastName ASC)
            FROM Account
            WHERE Id IN :accountIds
            WITH SECURITY_ENFORCED
        ];
    }
}
```

### Bonnes pratiques SOQL

```apex
// ✅ Utiliser des variables de liaison (bind variables) — jamais de concaténation
String industry = 'Technology';
List<Account> accs = [SELECT Id FROM Account WHERE Industry = :industry]; // ✅ Safe

// ❌ JAMAIS — vulnérabilité SOQL Injection
String query = 'SELECT Id FROM Account WHERE Industry = \'' + industry + '\''; // ❌ DANGEREUX
List<Account> accs2 = Database.query(query);

// ✅ WITH SECURITY_ENFORCED — respect des FLS
List<Contact> contacts = [SELECT Id, Email FROM Contact WITH SECURITY_ENFORCED];

// ✅ Limiter les champs — ne jamais utiliser SELECT *
// Apex n'a pas de SELECT * mais éviter de sélectionner des champs inutiles
```

---

## Module 5 — DML de qualité : Unit of Work Pattern {#module-5}

### Le problème des DML dispersés

```apex
// ❌ ANTI-PATTERN — DML multiples et non groupés
public void processData(List<Account> accounts) {
    insert accounts;                    // DML 1
    
    List<Contact> contacts = new List<Contact>();
    for (Account acc : accounts) {
        contacts.add(new Contact(AccountId = acc.Id, LastName = acc.Name));
    }
    insert contacts;                    // DML 2
    
    List<Task> tasks = new List<Task>();
    for (Contact c : contacts) {
        tasks.add(new Task(WhoId = c.Id, Subject = 'Appel de bienvenue'));
    }
    insert tasks;                       // DML 3
}
```

### Unit of Work Pattern simplifié

```apex
public class UnitOfWork {
    
    private Map<Schema.SObjectType, List<SObject>> toInsert = 
        new Map<Schema.SObjectType, List<SObject>>();
    private Map<Schema.SObjectType, List<SObject>> toUpdate = 
        new Map<Schema.SObjectType, List<SObject>>();
    private List<SObject> toDelete = new List<SObject>();

    public void registerNew(SObject record) {
        Schema.SObjectType sObjType = record.getSObjectType();
        if (!toInsert.containsKey(sObjType)) {
            toInsert.put(sObjType, new List<SObject>());
        }
        toInsert.get(sObjType).add(record);
    }

    public void registerDirty(SObject record) {
        Schema.SObjectType sObjType = record.getSObjectType();
        if (!toUpdate.containsKey(sObjType)) {
            toUpdate.put(sObjType, new List<SObject>());
        }
        toUpdate.get(sObjType).add(record);
    }

    public void registerDeleted(SObject record) {
        toDelete.add(record);
    }

    // Un seul point de commit — toutes les DML en une fois
    public void commitWork() {
        Savepoint sp = Database.setSavepoint();
        try {
            for (List<SObject> records : toInsert.values()) {
                insert records;
            }
            for (List<SObject> records : toUpdate.values()) {
                update records;
            }
            if (!toDelete.isEmpty()) {
                delete toDelete;
            }
        } catch (Exception e) {
            Database.rollback(sp);
            throw e;
        }
    }
}
```

### Utilisation du Unit of Work

```apex
public class AccountOnboardingService {

    public static void onboardNewAccounts(List<Account> accounts) {
        UnitOfWork uow = new UnitOfWork();

        for (Account acc : accounts) {
            // Contact principal
            Contact primaryContact = new Contact(
                AccountId = acc.Id,
                LastName  = 'Contact Principal',
                Email     = acc.Email__c
            );
            uow.registerNew(primaryContact);

            // Mise à jour du compte
            acc.Description = 'Intégration effectuée le ' + Date.today().format();
            uow.registerDirty(acc);
        }

        // Un seul commit — toutes les DML groupées
        uow.commitWork();
    }
}
```

---

## Module 6 — Principes SOLID appliqués à Apex {#module-6}

### S — Single Responsibility Principle

```apex
// ❌ ANTI-PATTERN — classe qui fait tout
public class AccountManager {
    public void saveAccount(Account acc) { /* DML */ }
    public void sendWelcomeEmail(Account acc) { /* Email */ }
    public void logActivity(Account acc) { /* Log */ }
    public void validateAccount(Account acc) { /* Validation */ }
    public String formatAccountName(String name) { /* Formatage */ }
}

// ✅ PATTERN CORRECT — responsabilités séparées
public class AccountRepository    { public void save(Account acc) { update acc; } }
public class AccountEmailService  { public void sendWelcome(Account acc) { /* Email */ } }
public class AccountValidator     { public void validate(Account acc) { /* Validation */ } }
public class AccountNameFormatter { public String format(String name) { return name.trim().capitalize(); } }
```

### O — Open/Closed Principle

```apex
// ✅ Extension sans modification — via interface
public interface DiscountStrategy {
    Decimal calculateDiscount(Opportunity opp);
}

public class VolumeDiscountStrategy implements DiscountStrategy {
    public Decimal calculateDiscount(Opportunity opp) {
        return opp.Amount > 100000 ? 0.15 : 0.05;
    }
}

public class SeasonalDiscountStrategy implements DiscountStrategy {
    public Decimal calculateDiscount(Opportunity opp) {
        Integer month = Date.today().month();
        return (month == 12 || month == 1) ? 0.20 : 0.10;
    }
}

// Ajouter une nouvelle stratégie = nouvelle classe, zero modification
public class OpportunityPricingService {
    private DiscountStrategy strategy;
    
    public OpportunityPricingService(DiscountStrategy strategy) {
        this.strategy = strategy;
    }
    
    public Decimal getDiscountedAmount(Opportunity opp) {
        Decimal discount = strategy.calculateDiscount(opp);
        return opp.Amount * (1 - discount);
    }
}
```

### D — Dependency Inversion Principle

```apex
// Interface — abstraction
public interface IAccountSelector {
    List<Account> selectById(Set<Id> ids);
}

// Implémentation réelle
public class AccountSelector implements IAccountSelector {
    public List<Account> selectById(Set<Id> ids) {
        return [SELECT Id, Name FROM Account WHERE Id IN :ids];
    }
}

// Implémentation de test (mock)
public class MockAccountSelector implements IAccountSelector {
    public List<Account> selectById(Set<Id> ids) {
        return new List<Account>{
            new Account(Id = ids.iterator().next(), Name = 'Test Account')
        };
    }
}

// Service qui dépend de l'abstraction, pas de l'implémentation
public class AccountService {
    private IAccountSelector selector;
    
    // Injection de dépendance via constructeur
    public AccountService(IAccountSelector selector) {
        this.selector = selector;
    }
    
    public AccountService() {
        this(new AccountSelector()); // défaut en production
    }
    
    public void process(Set<Id> accountIds) {
        List<Account> accounts = selector.selectById(accountIds);
        // logique...
    }
}
```

---

## Module 7 — Service Layer : la logique métier isolée {#module-7}

### Structure en couches

```
Trigger → TriggerHandler → Service → Selector / Repository
                                   → UnitOfWork
```

### Règles du Service Layer

1. **Stateless** : pas d'état entre les appels
2. **Bulkifié** : toujours accepter des listes
3. **Sans DML direct** : déléguer au Unit of Work
4. **Sans SOQL direct** : déléguer au Selector

```apex
public class OpportunityService {

    // -------------------------------------------------------
    // Méthodes publiques — point d'entrée métier
    // -------------------------------------------------------
    
    public static void qualifyOpportunities(List<Opportunity> opportunities) {
        validateOpportunitiesForQualification(opportunities);
        
        UnitOfWork uow = new UnitOfWork();
        List<Task> followUpTasks = buildFollowUpTasks(opportunities);
        
        for (Opportunity opp : opportunities) {
            opp.StageName = 'Qualification';
            uow.registerDirty(opp);
        }
        
        for (Task t : followUpTasks) {
            uow.registerNew(t);
        }
        
        uow.commitWork();
    }

    // -------------------------------------------------------
    // Méthodes privées — logique interne
    // -------------------------------------------------------
    
    private static void validateOpportunitiesForQualification(List<Opportunity> opps) {
        for (Opportunity opp : opps) {
            if (opp.Amount == null || opp.Amount <= 0) {
                throw new OpportunityServiceException(
                    'Le montant est obligatoire pour qualifier l\'opportunité : ' + opp.Name
                );
            }
        }
    }

    private static List<Task> buildFollowUpTasks(List<Opportunity> opportunities) {
        List<Task> tasks = new List<Task>();
        for (Opportunity opp : opportunities) {
            tasks.add(new Task(
                WhatId       = opp.Id,
                Subject      = 'Suivi qualification — ' + opp.Name,
                ActivityDate = Date.today().addDays(3),
                Priority     = 'High',
                Status       = 'Not Started',
                OwnerId      = opp.OwnerId
            ));
        }
        return tasks;
    }

    public class OpportunityServiceException extends Exception {}
}
```

---

## Module 8 — Gestion des erreurs et exceptions {#module-8}

### Hiérarchie d'exceptions personnalisées

```apex
// Exception de base de l'application
public abstract class AppException extends Exception {}

// Exceptions spécialisées
public class ValidationException     extends AppException {}
public class IntegrationException    extends AppException {}
public class ConfigurationException  extends AppException {}
public class AuthorizationException  extends AppException {}
```

### Database.SaveResult — Gestion partielle des erreurs

```apex
public class AccountSaveService {

    public class SaveResult {
        public List<Account> successRecords = new List<Account>();
        public List<String>  errorMessages  = new List<String>();
        public Boolean       hasErrors { get { return !errorMessages.isEmpty(); } }
    }

    public static SaveResult saveAccounts(List<Account> accounts) {
        SaveResult result = new SaveResult();
        
        // allOrNone = false → traitement partiel
        List<Database.SaveResult> dbResults = Database.insert(accounts, false);
        
        for (Integer i = 0; i < dbResults.size(); i++) {
            Database.SaveResult sr = dbResults[i];
            if (sr.isSuccess()) {
                result.successRecords.add(accounts[i]);
            } else {
                String errorMsg = buildErrorMessage(accounts[i], sr.getErrors());
                result.errorMessages.add(errorMsg);
                logError(errorMsg);
            }
        }
        
        return result;
    }

    private static String buildErrorMessage(Account acc, List<Database.Error> errors) {
        List<String> messages = new List<String>();
        for (Database.Error err : errors) {
            messages.add(
                'Champ(s) : ' + String.join(err.getFields(), ', ') +
                ' — Code : ' + err.getStatusCode() +
                ' — Message : ' + err.getMessage()
            );
        }
        return 'Erreur sur Account [' + acc.Name + '] : ' + String.join(messages, ' | ');
    }

    private static void logError(String message) {
        // Intégrer avec votre solution de logging (Custom Object, Platform Event, etc.)
        System.debug(LoggingLevel.ERROR, message);
    }
}
```

### Try/Catch avec Savepoint

```apex
public static void executeWithRollback(List<SObject> records) {
    Savepoint sp = Database.setSavepoint();
    try {
        insert records;
        // Autres opérations...
        
    } catch (DmlException e) {
        Database.rollback(sp);
        throw new AppException('Erreur DML — rollback effectué : ' + e.getMessage());
        
    } catch (QueryException e) {
        Database.rollback(sp);
        throw new AppException('Erreur SOQL — rollback effectué : ' + e.getMessage());
        
    } catch (Exception e) {
        Database.rollback(sp);
        throw new AppException('Erreur inattendue — rollback effectué : ' + e.getMessage());
    }
}
```

---

## Module 9 — Sécurité : CRUD, FLS et Sharing {#module-9}

### Sharing Model

```apex
// with sharing — respecte les règles de partage Salesforce (recommandé par défaut)
public with sharing class AccountService {
    public List<Account> getAccounts() {
        return [SELECT Id, Name FROM Account]; // limité aux enregistrements accessibles
    }
}

// without sharing — bypass des règles (justification obligatoire en commentaire)
public without sharing class SystemIntegrationService {
    // ⚠️ without sharing justifié : intégration système nécessitant accès global
    public List<Account> getAllAccountsForSync() {
        return [SELECT Id, Name, ExternalId__c FROM Account];
    }
}

// inherited sharing — hérite du contexte appelant (pour les classes utilitaires)
public inherited sharing class AccountUtils {
    public static String formatName(String name) {
        return name?.trim()?.capitalize();
    }
}
```

### CRUD & FLS

```apex
public class SecureAccountService {

    public void createAccount(Account acc) {
        // Vérification CRUD — l'utilisateur peut-il créer des Accounts ?
        if (!Schema.SObjectType.Account.isCreateable()) {
            throw new AuthorizationException(
                'L\'utilisateur n\'a pas la permission de créer des Accounts.'
            );
        }
        
        // Vérification FLS — l'utilisateur peut-il écrire dans ces champs ?
        Map<String, Schema.SObjectField> fieldsMap = Schema.SObjectType.Account.fields.getMap();
        List<String> unaccessibleFields = new List<String>();
        
        for (String fieldName : new List<String>{'Name', 'Industry', 'Rating'}) {
            if (!fieldsMap.get(fieldName).getDescribe().isCreateable()) {
                unaccessibleFields.add(fieldName);
            }
        }
        
        if (!unaccessibleFields.isEmpty()) {
            throw new AuthorizationException(
                'Champs inaccessibles : ' + String.join(unaccessibleFields, ', ')
            );
        }
        
        insert acc;
    }
}
```

### WITH SECURITY_ENFORCED vs stripInaccessible

```apex
// Option 1 : WITH SECURITY_ENFORCED (throw exception si champ inaccessible)
List<Account> accounts = [SELECT Id, Name, Industry FROM Account WITH SECURITY_ENFORCED];

// Option 2 : stripInaccessible (retire silencieusement les champs inaccessibles)
List<Account> rawAccounts = [SELECT Id, Name, Industry, Rating FROM Account];
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, rawAccounts);
List<Account> safeAccounts = (List<Account>) decision.getRecords();
```

---

## Module 10 — Apex Asynchrone {#module-10}

### Tableau comparatif

| Type | Use Case | Limite |
|------|----------|--------|
| `@future` | Callout après DML, traitement simple | 50/transaction |
| `Queueable` | Chaînage, state, objets complexes | 50 jobs en queue |
| `Batchable` | Traitement massif (millions d'enregistrements) | 5 jobs simultanés |
| `Schedulable` | Planification temporelle | 100 jobs planifiés |

### @future — Callouts HTTP

```apex
public class ExternalApiService {

    // Méthode synchrone qui déclenche le callout en async
    public static void syncAccountToErp(Set<Id> accountIds) {
        if (!accountIds.isEmpty()) {
            callExternalApi(accountIds);
        }
    }

    @future(callout=true)
    private static void callExternalApi(Set<Id> accountIds) {
        List<Account> accounts = [SELECT Id, Name, Industry FROM Account WHERE Id IN :accountIds];
        
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:ERP_Named_Credential/api/accounts');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(JSON.serialize(accounts));
        req.setTimeout(30000);
        
        HttpResponse res = new Http().send(req);
        
        if (res.getStatusCode() != 200) {
            throw new IntegrationException(
                'Erreur ERP [' + res.getStatusCode() + '] : ' + res.getBody()
            );
        }
    }
}
```

### Queueable — Traitement chaînable

```apex
public class AccountSyncQueueable implements Queueable, Database.AllowsCallouts {
    
    private List<Id> accountIds;
    private Integer batchSize;
    private Integer currentIndex;

    public AccountSyncQueueable(List<Id> accountIds) {
        this(accountIds, 0, 10);
    }

    private AccountSyncQueueable(List<Id> accountIds, Integer currentIndex, Integer batchSize) {
        this.accountIds   = accountIds;
        this.currentIndex = currentIndex;
        this.batchSize    = batchSize;
    }

    public void execute(QueueableContext context) {
        // Traiter le lot courant
        Integer endIndex = Math.min(currentIndex + batchSize, accountIds.size());
        List<Id> batch = new List<Id>();
        for (Integer i = currentIndex; i < endIndex; i++) {
            batch.add(accountIds[i]);
        }

        processAccounts(batch);

        // Chaîner si des enregistrements restent
        if (endIndex < accountIds.size()) {
            System.enqueueJob(
                new AccountSyncQueueable(accountIds, endIndex, batchSize)
            );
        }
    }

    private void processAccounts(List<Id> ids) {
        // Traitement...
    }
}
```

### Batch Apex — Traitement massif

```apex
public class AccountDataCleanupBatch implements Database.Batchable<SObject>, Database.Stateful {

    // Database.Stateful permet de conserver l'état entre les chunks
    private Integer processedCount = 0;
    private List<String> errors    = new List<String>();

    public Database.QueryLocator start(Database.BatchableContext context) {
        return Database.getQueryLocator([
            SELECT Id, Name, Description, LastModifiedDate
            FROM Account
            WHERE LastModifiedDate < LAST_N_YEARS:2
            AND Description = null
        ]);
    }

    public void execute(Database.BatchableContext context, List<Account> scope) {
        List<Account> toUpdate = new List<Account>();
        
        for (Account acc : scope) {
            acc.Description = 'Archivé — Nettoyage ' + Date.today().format();
            toUpdate.add(acc);
        }

        try {
            update toUpdate;
            processedCount += toUpdate.size();
        } catch (DmlException e) {
            errors.add('Lot échoué : ' + e.getMessage());
        }
    }

    public void finish(Database.BatchableContext context) {
        // Notification de fin
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
        email.setToAddresses(new List<String>{'admin@company.com'});
        email.setSubject('Batch Account Cleanup terminé');
        email.setPlainTextBody(
            'Traités : ' + processedCount + '\n' +
            'Erreurs : ' + errors.size() + '\n' +
            (errors.isEmpty() ? '' : String.join(errors, '\n'))
        );
        Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{email});
    }
}

// Exécution
Database.executeBatch(new AccountDataCleanupBatch(), 200); // 200 = taille de chunk recommandée
```

### Schedulable

```apex
public class DailyCleanupScheduler implements Schedulable {

    public void execute(SchedulableContext context) {
        // Lancer le batch depuis le scheduler
        Database.executeBatch(new AccountDataCleanupBatch(), 200);
    }

    // Méthode utilitaire pour planifier
    public static void schedule() {
        String cronExpression = '0 0 2 * * ?'; // Tous les jours à 2h du matin
        System.schedule('Daily Account Cleanup', cronExpression, new DailyCleanupScheduler());
    }
}
```

---

## Module 11 — Tests unitaires de qualité {#module-11}

### Les 3 piliers d'un bon test

1. **Isolation** : tester un comportement précis
2. **Couverture** : cas nominal + cas d'erreur
3. **Lisibilité** : le test documente le comportement

### Structure AAA (Arrange, Act, Assert)

```apex
@IsTest
private class OpportunityServiceTest {

    @TestSetup
    static void setupTestData() {
        // Données partagées entre tous les tests de la classe
        Account acc = new Account(Name = 'Test Corp', Industry = 'Technology');
        insert acc;

        List<Opportunity> opps = new List<Opportunity>();
        for (Integer i = 0; i < 5; i++) {
            opps.add(new Opportunity(
                Name       = 'Test Opp ' + i,
                AccountId  = acc.Id,
                StageName  = 'Prospecting',
                CloseDate  = Date.today().addDays(30),
                Amount     = 10000 * (i + 1)
            ));
        }
        insert opps;
    }

    @IsTest
    static void qualifyOpportunities_WithValidAmount_ShouldUpdateStage() {
        // ── Arrange ──────────────────────────────────────────────
        List<Opportunity> opps = [SELECT Id, Name, Amount, StageName, OwnerId
                                  FROM Opportunity
                                  LIMIT 5];

        // ── Act ──────────────────────────────────────────────────
        Test.startTest();
        OpportunityService.qualifyOpportunities(opps);
        Test.stopTest();

        // ── Assert ───────────────────────────────────────────────
        List<Opportunity> updatedOpps = [SELECT Id, StageName FROM Opportunity WHERE Id IN :opps];
        for (Opportunity opp : updatedOpps) {
            System.assertEquals('Qualification', opp.StageName,
                'La stage devrait être Qualification après qualification');
        }
    }

    @IsTest
    static void qualifyOpportunities_WithNullAmount_ShouldThrowException() {
        // ── Arrange ──────────────────────────────────────────────
        Opportunity oppWithoutAmount = new Opportunity(
            Name      = 'Opp sans montant',
            StageName = 'Prospecting',
            CloseDate = Date.today().addDays(30),
            Amount    = null
        );
        // NE PAS insérer — tester la validation en mémoire

        // ── Act & Assert ─────────────────────────────────────────
        Test.startTest();
        try {
            OpportunityService.qualifyOpportunities(
                new List<Opportunity>{oppWithoutAmount}
            );
            System.assert(false, 'Une exception était attendue');
        } catch (OpportunityService.OpportunityServiceException e) {
            System.assert(
                e.getMessage().contains('montant est obligatoire'),
                'Message d\'exception incorrect : ' + e.getMessage()
            );
        }
        Test.stopTest();
    }

    @IsTest
    static void qualifyOpportunities_BulkTest_ShouldHandle200Records() {
        // ── Arrange — test de charge ──────────────────────────────
        Account acc = [SELECT Id FROM Account LIMIT 1];
        List<Opportunity> largeList = new List<Opportunity>();
        for (Integer i = 0; i < 200; i++) {
            largeList.add(new Opportunity(
                Name      = 'Bulk Opp ' + i,
                AccountId = acc.Id,
                StageName = 'Prospecting',
                CloseDate = Date.today().addDays(30),
                Amount    = 5000
            ));
        }
        insert largeList;

        // ── Act ──────────────────────────────────────────────────
        Test.startTest();
        OpportunityService.qualifyOpportunities(largeList);
        Test.stopTest();

        // ── Assert ───────────────────────────────────────────────
        Integer qualifiedCount = [
            SELECT COUNT() FROM Opportunity
            WHERE StageName = 'Qualification'
            AND Id IN :largeList
        ];
        System.assertEquals(200, qualifiedCount, 'Les 200 opportunités doivent être qualifiées');
    }
}
```

### Test avec Mock (Pattern Stub)

```apex
@IsTest
private class AccountServiceTest {

    // Mock du selector
    private class MockSelector implements IAccountSelector {
        public List<Account> selectById(Set<Id> ids) {
            return new List<Account>{
                new Account(Id = ids.iterator().next(), Name = 'Mock Account', Industry = 'Tech')
            };
        }
    }

    @IsTest
    static void process_ShouldUseInjectedSelector() {
        // Injection du mock — aucune donnée en base requise
        AccountService service = new AccountService(new MockSelector());
        
        Test.startTest();
        // ... appel du service avec le mock
        Test.stopTest();
        
        // Assertions sans queries SOQL dans le test
    }
}
```

---

## Module 12 — Conventions de nommage et lisibilité {#module-12}

### Conventions générales

| Élément | Convention | Exemple |
|---------|-----------|---------|
| Classe | PascalCase | `AccountTriggerHandler` |
| Méthode | camelCase, verbe | `calculateDiscount()` |
| Variable | camelCase | `primaryContact` |
| Constante | UPPER_SNAKE_CASE | `MAX_RECORDS_PER_BATCH` |
| Classe de test | `[Classe]Test` | `AccountServiceTest` |
| Exception custom | `[Domaine]Exception` | `OpportunityServiceException` |
| Interface | `I[Nom]` | `IAccountSelector` |

### Nommage expressif

```apex
// ❌ MAUVAIS — opaque et cryptique
public void proc(List<Account> a) {
    for (Account x : a) {
        if (x.r == 'Hot') {
            x.d = true;
        }
    }
}

// ✅ BON — se lit comme de la prose
public void markHotAccountsAsDeleted(List<Account> accounts) {
    for (Account account : accounts) {
        if (isHotAccount(account)) {
            account.IsDeleted__c = true;
        }
    }
}

private Boolean isHotAccount(Account account) {
    return account.Rating == 'Hot';
}
```

### Organisation d'une classe

```apex
public with sharing class AccountService {

    // ── 1. Constantes ────────────────────────────────────────────
    private static final Integer MAX_BATCH_SIZE = 200;
    private static final String  DEFAULT_RATING  = 'Warm';

    // ── 2. Variables d'instance ──────────────────────────────────
    private IAccountSelector selector;

    // ── 3. Constructeurs ─────────────────────────────────────────
    public AccountService() {
        this(new AccountSelector());
    }

    public AccountService(IAccountSelector selector) {
        this.selector = selector;
    }

    // ── 4. Méthodes publiques ────────────────────────────────────
    public void processAccounts(Set<Id> accountIds) { /* ... */ }

    // ── 5. Méthodes privées ──────────────────────────────────────
    private void validate(List<Account> accounts) { /* ... */ }

    // ── 6. Classes internes ──────────────────────────────────────
    public class AccountServiceException extends Exception {}
}
```

---

## Module 13 — Architecture complète : fflib Apex Common {#module-13}

### Vue d'ensemble de l'architecture en couches

```
┌─────────────────────────────────────────────────────────┐
│                    COUCHE PRÉSENTATION                   │
│         (Triggers, LWC Controllers, REST APIs)           │
└─────────────────────────┬───────────────────────────────┘
                           │
┌─────────────────────────▼───────────────────────────────┐
│                    COUCHE SERVICE                        │
│            (Logique métier, orchestration)               │
└───────────┬─────────────────────────┬───────────────────┘
            │                         │
┌───────────▼────────┐   ┌────────────▼───────────────────┐
│  COUCHE DOMAINE    │   │       COUCHE SÉLECTEUR          │
│  (Règles métier    │   │       (Requêtes SOQL)           │
│   sur les SObjects)│   └────────────────────────────────┘
└────────────────────┘
            │
┌───────────▼────────────────────────────────────────────┐
│                   UNIT OF WORK                          │
│              (Gestion des DML)                          │
└────────────────────────────────────────────────────────┘
```

### Application Order Summary

Pour un projet de qualité professionnelle, appliquer dans l'ordre :

1. **Un seul trigger par SObject** → délégation immédiate au handler
2. **TriggerHandler** → délégation au service, zéro logique
3. **Service** → logique métier bulkifiée, pas de SOQL ni DML directs
4. **Selector** → toutes les requêtes SOQL centralisées, `WITH SECURITY_ENFORCED`
5. **Unit of Work** → tous les DML en un seul commit, avec rollback
6. **Interfaces** → pour l'injection de dépendances et la testabilité
7. **Tests** → AAA pattern, bulk test (200 records), cas d'erreur

---

## Checklist de code review

Avant chaque commit, vérifier :

- [ ] **Zéro SOQL dans une boucle**
- [ ] **Zéro DML dans une boucle**
- [ ] **Tous les SOQL utilisent `WITH SECURITY_ENFORCED` ou `stripInaccessible`**
- [ ] **Toutes les classes déclarent `with sharing` ou justifient `without sharing`**
- [ ] **La logique métier est dans le Service, pas dans le Trigger**
- [ ] **Chaque méthode publique a un test unitaire correspondant**
- [ ] **Les cas d'erreur sont testés (exception, null, liste vide)**
- [ ] **Un test bulk avec 200+ enregistrements existe**
- [ ] **Pas de `System.debug` laissé en production**
- [ ] **Les noms de variables/méthodes sont expressifs et en anglais**
- [ ] **Le code compile sans warning**
- [ ] **Couverture de tests ≥ 85% (objectif ≥ 95%)**

---

*Cours rédigé selon les standards d'ingénierie logicielle avancés pour Salesforce Apex.*  
*Dernière révision : Avril 2026*
