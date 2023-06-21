# Sparkscon: Alexa Voice Commerce

## Interaction Model
1. Invocation name
2. Intent
3. Slot Types
4. Sample utterances
5. Dialog model (Requirements, Validations, Prompts)
   
## Testing
- Utterance Profiler
- Simulator
- Real Device (Echo)
  
## Beispiel Intent 'WhereToGetFoodHandler' Backend Code

*Den Intent Handler erstellen*
```
const WhereToGetFoodHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'WhereToGetFood';
    },
    handle(handlerInput) {
	const food = Alexa.getSlotValue(handlerInput.requestEnvelope, 'food');
        const speakOutput = `Du findest ${food} draußen in einem der Stände.`;

        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(speakOutput)
            .getResponse();
    }
};
```

*Den Intent Handler registrieren*
```
.addRequestHandlers(
	...,
	WhereToGetFoodHandler,
	...)
```

## SSML
```
<speak>    
	<amazon:emotion name="excited" intensity="medium">Willkommen zur <lang xml:lang="en-US">Sparkscon</lang></amazon:emotion>. Deutschlands größte Digital Experience Konferenz.
</speak> 
```

```
<speak>   
    Noch ein Geheimtipp von mir: 
    <amazon:effect name="whispered">Nach der <lang xml:lang="en-US">Sparkscon</lang> steigt eine after show party</amazon:effect>.
</speak>
```

## Variable Handling

### Session Variablen (Attributemanager)

*Alle Session Variablen holen, einen Wert zuweisen und wieder in den attributesManager schreiben*
```
const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();

sessionAttributes['food'] = food;
handlerInput.attributesManager.setSessionAttributes(sessionAttributes);
```

*Die Variable in einem anderen Intent holen und ausgeben*
```
const WhatIsMyFavoriteFoodHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'WhatIsMyFavoriteFood';
    },
    handle(handlerInput) {
	const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();

        const speakOutput = `Mein Lieblingsessen ist ${sessionAttributes['food']}`;

        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(speakOutput)
            .getResponse();
    }
};
```

*Wieder den handler des neuen Intents registrieren*
```
.addRequestHandlers(
	...,
	WhatIsMyFavoriteFoodHandler,
	...)
```

### Variablen persistieren
*Zu Beginn den persistanceAdapter initialisieren*
```
// Get an instance of the persistence adapter
var persistenceAdapter = getPersistenceAdapter();

function getPersistenceAdapter(tableName) {
    // This function is an indirect way to detect if this is part of an Alexa-Hosted skill
    function isAlexaHosted() {
        return process.env.S3_PERSISTENCE_BUCKET;
    }
    if (isAlexaHosted()) {
        const {S3PersistenceAdapter} = require('ask-sdk-s3-persistence-adapter');
        return new S3PersistenceAdapter({
            bucketName: process.env.S3_PERSISTENCE_BUCKET
        });
    }
}
```

*Inceptors für das Laden und Speichern der Variablen anlegen*
```
/* *
 * Below we use async and await ( more info: javascript.info/async-await )
 * It's a way to wrap promises and waait for the result of an external async operation
 * Like getting and saving the persistent attributes
 * */
const LoadAttributesRequestInterceptor = {
    async process(handlerInput) {
        const {attributesManager, requestEnvelope} = handlerInput;
        if (Alexa.isNewSession(requestEnvelope)){ //is this a new session? this check is not enough if using auto-delegate (more on next module)
            const persistentAttributes = await attributesManager.getPersistentAttributes() || {};
            console.log('Loading from persistent storage: ' + JSON.stringify(persistentAttributes));
            //copy persistent attribute to session attributes
            attributesManager.setSessionAttributes(persistentAttributes); // ALL persistent attributtes are now session attributes
        }
    }
};

// If you disable the skill and reenable it the userId might change and you loose the persistent attributes saved below as userId is the primary key
const SaveAttributesResponseInterceptor = {
    async process(handlerInput, response) {
        if (!response) return; // avoid intercepting calls that have no outgoing response due to errors
        const {attributesManager, requestEnvelope} = handlerInput;
        const sessionAttributes = attributesManager.getSessionAttributes();
        const shouldEndSession = (typeof response.shouldEndSession === "undefined" ? true : response.shouldEndSession); //is this a session end?
        if (shouldEndSession || Alexa.getRequestType(requestEnvelope) === 'SessionEndedRequest') { // skill was stopped or timed out
            // we increment a persistent session counter here
            sessionAttributes['sessionCounter'] = sessionAttributes['sessionCounter'] ? sessionAttributes['sessionCounter'] + 1 : 1;
            // we make ALL session attributes persistent
            console.log('Saving to persistent storage:' + JSON.stringify(sessionAttributes));
            attributesManager.setPersistentAttributes(sessionAttributes);
            await attributesManager.savePersistentAttributes();
        }
    }
};
```

*Beide Inceptors am Ende des Files registrieren*
```
.addRequestInterceptors(
	...
	LoadAttributesRequestInterceptor)
.addResponseInterceptors(
  	...
	SaveAttributesResponseInterceptor)
.withPersistenceAdapter(persistenceAdapter)
```

