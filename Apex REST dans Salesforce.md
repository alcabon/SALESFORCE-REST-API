Voici les bonnes pratiques essentielles pour développer des services Apex REST dans Salesforce :

## Structure et annotations

**Utilisez les bonnes annotations :**
```apex
@RestResource(urlMapping='/api/accounts/*')
global with sharing class AccountRestService {
    
    @HttpGet
    global static ResponseWrapper getAccount() {
        // Implementation
    }
    
    @HttpPost
    global static ResponseWrapper createAccount(String name, String industry) {
        // Implementation
    }
}
```

**Créez des classes wrapper pour les réponses :**
```apex
global class ResponseWrapper {
    global Boolean success;
    global String message;
    global Object data;
    global List<String> errors;
    
    global ResponseWrapper(Boolean success, String message, Object data) {
        this.success = success;
        this.message = message;
        this.data = data;
        this.errors = new List<String>();
    }
}
```

## Gestion des erreurs robuste

**Implémentez une gestion d'erreurs complète :**
```apex
@HttpPost
global static ResponseWrapper createAccount() {
    try {
        RestRequest req = RestContext.request;
        RestResponse res = RestContext.response;
        
        // Validation des paramètres
        if (String.isBlank(req.params.get('name'))) {
            res.statusCode = 400;
            return new ResponseWrapper(false, 'Le nom est requis', null);
        }
        
        Account acc = new Account(Name = req.params.get('name'));
        insert acc;
        
        res.statusCode = 201;
        return new ResponseWrapper(true, 'Compte créé avec succès', acc);
        
    } catch (DmlException e) {
        RestContext.response.statusCode = 400;
        return new ResponseWrapper(false, 'Erreur lors de la création', e.getDmlMessage(0));
        
    } catch (Exception e) {
        RestContext.response.statusCode = 500;
        return new ResponseWrapper(false, 'Erreur interne du serveur', e.getMessage());
    }
}
```

## Sécurité et autorisations

**Utilisez `with sharing` pour respecter les règles de partage :**
```apex
@RestResource(urlMapping='/api/accounts/*')
global with sharing class AccountRestService {
    // Le code respectera les règles de partage de l'utilisateur
}
```

**Validez les autorisations explicitement :**
```apex
@HttpGet
global static ResponseWrapper getAccounts() {
    if (!Schema.sObjectType.Account.isAccessible()) {
        RestContext.response.statusCode = 403;
        return new ResponseWrapper(false, 'Accès non autorisé', null);
    }
    // Suite du code
}
```

## Optimisation des performances

**Évitez les requêtes dans les boucles :**
```apex
// ❌ Mauvais
List<Contact> contacts = new List<Contact>();
for (Id accountId : accountIds) {
    contacts.addAll([SELECT Id, Name FROM Contact WHERE AccountId = :accountId]);
}

// ✅ Bon
List<Contact> contacts = [SELECT Id, Name, AccountId FROM Contact WHERE AccountId IN :accountIds];
```

**Utilisez la pagination pour les gros volumes :**
```apex
@HttpGet
global static ResponseWrapper getAccounts() {
    Integer pageSize = Integer.valueOf(RestContext.request.params.get('pageSize') ?? '20');
    Integer offset = Integer.valueOf(RestContext.request.params.get('offset') ?? '0');
    
    List<Account> accounts = [SELECT Id, Name FROM Account LIMIT :pageSize OFFSET :offset];
    
    return new ResponseWrapper(true, 'Succès', accounts);
}
```

## Validation et sérialisation

**Validez les données d'entrée :**
```apex
@HttpPost
global static ResponseWrapper createAccount() {
    try {
        String requestBody = RestContext.request.requestBody.toString();
        Map<String, Object> params = (Map<String, Object>) JSON.deserializeUntyped(requestBody);
        
        // Validation
        if (!params.containsKey('name') || String.isBlank((String)params.get('name'))) {
            RestContext.response.statusCode = 400;
            return new ResponseWrapper(false, 'Le nom est requis', null);
        }
        
        // Traitement...
        
    } catch (JSONException e) {
        RestContext.response.statusCode = 400;
        return new ResponseWrapper(false, 'Format JSON invalide', null);
    }
}
```

## Logging et monitoring

**Implémentez un système de logging :**
```apex
@HttpPost
global static ResponseWrapper processData() {
    String requestId = String.valueOf(DateTime.now().getTime());
    
    try {
        System.debug('Request ID: ' + requestId + ' - Début du traitement');
        
        // Logique métier
        
        System.debug('Request ID: ' + requestId + ' - Traitement terminé avec succès');
        return new ResponseWrapper(true, 'Succès', result);
        
    } catch (Exception e) {
        System.debug('Request ID: ' + requestId + ' - Erreur: ' + e.getMessage());
        // Gestion d'erreur
    }
}
```

## Tests unitaires

**Créez des tests complets avec des données de test :**
```apex
@IsTest
private class AccountRestServiceTest {
    
    @IsTest
    static void testCreateAccountSuccess() {
        RestRequest req = new RestRequest();
        RestResponse res = new RestResponse();
        
        req.requestURI = '/services/apexrest/api/accounts/';
        req.httpMethod = 'POST';
        req.requestBody = Blob.valueOf('{"name":"Test Account"}');
        
        RestContext.request = req;
        RestContext.response = res;
        
        Test.startTest();
        ResponseWrapper result = AccountRestService.createAccount();
        Test.stopTest();
        
        System.assertEquals(true, result.success);
        System.assertEquals(201, res.statusCode);
    }
    
    @IsTest
    static void testCreateAccountError() {
        // Test avec données invalides
        RestRequest req = new RestRequest();
        req.requestBody = Blob.valueOf('{"name":""}');
        RestContext.request = req;
        RestContext.response = new RestResponse();
        
        ResponseWrapper result = AccountRestService.createAccount();
        
        System.assertEquals(false, result.success);
        System.assertEquals(400, RestContext.response.statusCode);
    }
}
```

## Autres bonnes pratiques importantes

- **Utilisez des limites de gouverneur appropriées** et surveillez la consommation
- **Documentez vos APIs** avec des commentaires clairs
- **Versionnez vos endpoints** pour maintenir la compatibilité
- **Implémentez une authentification robuste** (OAuth, JWT)
- **Utilisez des noms d'URL cohérents** et RESTful
- **Gérez les différents formats de contenu** (JSON, XML selon les besoins)
- **Implémentez des mécanismes de cache** quand approprié

Ces pratiques vous aideront à créer des services Apex REST maintenables, sécurisés et performants dans Salesforce.
