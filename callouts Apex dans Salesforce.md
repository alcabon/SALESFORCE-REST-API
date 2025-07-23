Voici les bonnes pratiques essentielles pour les callouts Apex dans Salesforce :

## Configuration et authentification

**Configurez les Named Credentials pour la sécurité :**
```apex
// Utilisez Named Credentials plutôt que des endpoints codés en dur
Http http = new Http();
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:My_API/endpoint');
req.setMethod('GET');
req.setHeader('Content-Type', 'application/json');
```

**Gérez l'authentification OAuth :**
```apex
public class AuthService {
    @future(callout=true)
    public static void refreshToken() {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:OAuth_Service/token');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/x-www-form-urlencoded');
        req.setBody('grant_type=client_credentials&client_id={!$Credential.Username}&client_secret={!$Credential.Password}');
        
        Http http = new Http();
        HttpResponse res = http.send(req);
        
        if (res.getStatusCode() == 200) {
            Map<String, Object> tokenData = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
            // Stocker le token en Custom Setting ou Custom Metadata
        }
    }
}
```

## Gestion robuste des erreurs

**Implémentez une gestion d'erreurs complète :**
```apex
public class CalloutService {
    
    public static CalloutResponse makeCallout(String endpoint, String method, String body) {
        CalloutResponse response = new CalloutResponse();
        
        try {
            HttpRequest req = new HttpRequest();
            req.setEndpoint(endpoint);
            req.setMethod(method);
            req.setHeader('Content-Type', 'application/json');
            req.setTimeout(30000); // 30 secondes
            
            if (String.isNotBlank(body)) {
                req.setBody(body);
            }
            
            Http http = new Http();
            HttpResponse res = http.send(req);
            
            response.statusCode = res.getStatusCode();
            response.body = res.getBody();
            response.success = (res.getStatusCode() >= 200 && res.getStatusCode() < 300);
            
            // Gestion des codes d'erreur spécifiques
            if (res.getStatusCode() == 401) {
                response.errorMessage = 'Non autorisé - Token expiré';
                // Déclencher un refresh du token
            } else if (res.getStatusCode() == 429) {
                response.errorMessage = 'Limite de taux atteinte - Retry après délai';
            } else if (res.getStatusCode() >= 500) {
                response.errorMessage = 'Erreur serveur - Service temporairement indisponible';
            }
            
        } catch (CalloutException e) {
            response.success = false;
            response.errorMessage = 'Erreur de callout: ' + e.getMessage();
            System.debug('CalloutException: ' + e.getMessage());
            
        } catch (Exception e) {
            response.success = false;
            response.errorMessage = 'Erreur inattendue: ' + e.getMessage();
            System.debug('Exception: ' + e.getMessage());
        }
        
        return response;
    }
}

public class CalloutResponse {
    public Boolean success;
    public Integer statusCode;
    public String body;
    public String errorMessage;
    
    public CalloutResponse() {
        this.success = false;
    }
}
```

## Patterns asynchrones

**Utilisez @future pour les callouts asynchrones :**
```apex
public class AsyncCalloutService {
    
    @future(callout=true)
    public static void sendDataAsync(String recordId, String jsonData) {
        try {
            CalloutResponse response = CalloutService.makeCallout(
                'callout:External_API/data',
                'POST',
                jsonData
            );
            
            // Mettre à jour l'enregistrement avec le résultat
            updateRecordStatus(recordId, response.success, response.errorMessage);
            
        } catch (Exception e) {
            updateRecordStatus(recordId, false, e.getMessage());
        }
    }
    
    private static void updateRecordStatus(String recordId, Boolean success, String message) {
        // Mise à jour de l'enregistrement sans callout
        sObject record = Database.query('SELECT Id FROM ' + recordId.getSObjectType() + ' WHERE Id = :recordId');
        record.put('Status__c', success ? 'Synchronized' : 'Error');
        record.put('Error_Message__c', success ? null : message);
        
        try {
            update record;
        } catch (DmlException e) {
            System.debug('Erreur mise à jour: ' + e.getMessage());
        }
    }
}
```

**Utilisez Queueable pour plus de flexibilité :**
```apex
public class QueueableCallout implements Queueable, Database.AllowsCallouts {
    
    private List<String> recordIds;
    private Integer batchSize = 5;
    
    public QueueableCallout(List<String> recordIds) {
        this.recordIds = recordIds;
    }
    
    public void execute(QueueableContext context) {
        List<String> currentBatch = new List<String>();
        
        // Traiter par lots pour respecter les limites
        for (Integer i = 0; i < Math.min(batchSize, recordIds.size()); i++) {
            currentBatch.add(recordIds[i]);
        }
        
        // Effectuer les callouts pour ce lot
        for (String recordId : currentBatch) {
            processRecord(recordId);
        }
        
        // Continuer avec le lot suivant si nécessaire
        if (recordIds.size() > batchSize) {
            List<String> remaining = new List<String>();
            for (Integer i = batchSize; i < recordIds.size(); i++) {
                remaining.add(recordIds[i]);
            }
            
            if (!Test.isRunningTest()) {
                System.enqueueJob(new QueueableCallout(remaining));
            }
        }
    }
    
    private void processRecord(String recordId) {
        // Logique de callout
    }
}
```

## Retry et circuit breaker

**Implémentez un mécanisme de retry :**
```apex
public class RetryCalloutService {
    
    private static final Integer MAX_RETRIES = 3;
    private static final Integer BASE_DELAY = 1000; // millisecondes
    
    public static CalloutResponse makeCalloutWithRetry(String endpoint, String method, String body) {
        CalloutResponse lastResponse;
        
        for (Integer attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            lastResponse = CalloutService.makeCallout(endpoint, method, body);
            
            if (lastResponse.success) {
                return lastResponse;
            }
            
            // Retry seulement pour certains codes d'erreur
            if (shouldRetry(lastResponse.statusCode) && attempt < MAX_RETRIES) {
                // Exponential backoff (simulé via Custom Settings pour les delays)
                Integer delay = BASE_DELAY * (Integer) Math.pow(2, attempt - 1);
                System.debug('Retry attempt ' + attempt + ' après ' + delay + 'ms');
                
                // En production, utiliser Platform Events pour différer
                if (!Test.isRunningTest()) {
                    scheduleRetry(endpoint, method, body, attempt + 1, delay);
                    break;
                }
            }
        }
        
        return lastResponse;
    }
    
    private static Boolean shouldRetry(Integer statusCode) {
        return statusCode == 429 || // Rate limit
               statusCode == 502 || // Bad Gateway
               statusCode == 503 || // Service Unavailable
               statusCode == 504;   // Gateway Timeout
    }
    
    @future(callout=true)
    private static void scheduleRetry(String endpoint, String method, String body, Integer attempt, Integer delay) {
        // Implémenter le retry avec délai
    }
}
```

## Monitoring et logging

**Créez un système de logging complet :**
```apex
public class CalloutLogger {
    
    public static void logCallout(String endpoint, String method, String requestBody, 
                                HttpResponse response, Long duration) {
        
        Callout_Log__c log = new Callout_Log__c();
        log.Endpoint__c = endpoint;
        log.Method__c = method;
        log.Request_Body__c = truncateString(requestBody, 32000);
        log.Response_Body__c = truncateString(response?.getBody(), 32000);
        log.Status_Code__c = response?.getStatusCode();
        log.Duration_Ms__c = duration;
        log.Timestamp__c = DateTime.now();
        log.Success__c = (response?.getStatusCode() >= 200 && response?.getStatusCode() < 300);
        
        try {
            insert log;
        } catch (DmlException e) {
            System.debug('Erreur logging callout: ' + e.getMessage());
        }
    }
    
    private static String truncateString(String str, Integer maxLength) {
        if (String.isBlank(str) || str.length() <= maxLength) {
            return str;
        }
        return str.substring(0, maxLength) + '...';
    }
}

// Utilisation avec monitoring
public static CalloutResponse makeCalloutWithLogging(String endpoint, String method, String body) {
    Long startTime = DateTime.now().getTime();
    HttpResponse response;
    
    try {
        HttpRequest req = new HttpRequest();
        req.setEndpoint(endpoint);
        req.setMethod(method);
        req.setBody(body);
        
        Http http = new Http();
        response = http.send(req);
        
    } catch (Exception e) {
        System.debug('Erreur callout: ' + e.getMessage());
    } finally {
        Long duration = DateTime.now().getTime() - startTime;
        CalloutLogger.logCallout(endpoint, method, body, response, duration);
    }
    
    return new CalloutResponse(response);
}
```

## Tests unitaires avec mocks

**Créez des mocks pour vos tests :**
```apex
@IsTest
global class MockHttpResponseGenerator implements HttpCalloutMock {
    
    global HTTPResponse respond(HTTPRequest req) {
        HttpResponse res = new HttpResponse();
        res.setHeader('Content-Type', 'application/json');
        
        // Simuler différentes réponses selon l'endpoint
        if (req.getEndpoint().contains('/success')) {
            res.setStatusCode(200);
            res.setBody('{"status": "success", "data": {"id": "123"}}');
        } else if (req.getEndpoint().contains('/error')) {
            res.setStatusCode(500);
            res.setBody('{"error": "Internal Server Error"}');
        } else if (req.getEndpoint().contains('/timeout')) {
            res.setStatusCode(408);
            res.setBody('{"error": "Request Timeout"}');
        }
        
        return res;
    }
}

@IsTest
private class CalloutServiceTest {
    
    @IsTest
    static void testSuccessfulCallout() {
        Test.setMock(HttpCalloutMock.class, new MockHttpResponseGenerator());
        
        Test.startTest();
        CalloutResponse response = CalloutService.makeCallout('callout:Test_API/success', 'GET', null);
        Test.stopTest();
        
        System.assertEquals(true, response.success);
        System.assertEquals(200, response.statusCode);
    }
    
    @IsTest
    static void testErrorCallout() {
        Test.setMock(HttpCalloutMock.class, new MockHttpResponseGenerator());
        
        Test.startTest();
        CalloutResponse response = CalloutService.makeCallout('callout:Test_API/error', 'GET', null);
        Test.stopTest();
        
        System.assertEquals(false, response.success);
        System.assertEquals(500, response.statusCode);
    }
}
```

## Bonnes pratiques supplémentaires

**Gestion des limites de gouverneur :**
- Maximum 100 callouts par transaction
- Timeout maximum de 120 secondes
- Taille maximum de réponse : 6 MB

**Optimisation des performances :**
```apex
// Grouper les callouts quand possible
public static void bulkCallout(List<String> endpoints) {
    List<CalloutResponse> responses = new List<CalloutResponse>();
    
    for (String endpoint : endpoints) {
        if (Limits.getCallouts() < Limits.getLimitCallouts()) {
            responses.add(makeCallout(endpoint, 'GET', null));
        } else {
            // Queue le reste pour traitement asynchrone
            QueueableCallout job = new QueueableCallout(
                endpoints.subList(endpoints.indexOf(endpoint), endpoints.size())
            );
            System.enqueueJob(job);
            break;
        }
    }
}
```

**Sécurité :**
- Toujours utiliser HTTPS
- Valider les certificats SSL
- Chiffrer les données sensibles
- Utiliser Named Credentials pour les informations d'authentification
- Implémenter une validation des réponses

Ces pratiques vous permettront de créer des callouts robustes, maintenables et performants dans Salesforce.
