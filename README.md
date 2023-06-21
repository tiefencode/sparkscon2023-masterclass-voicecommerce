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
  
## Sample Intent:

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

```
.addRequestHandlers(
	...,
	WhereToGetFoodHandler,
	...)
```

## SSML
```
<speak>    
	<amazon:emotion name="excited" intensity="medium">Willkommen</amazon:emotion> zur <lang xml:lang="en-US">Sparkscon</lang>. Deutschlands größte Digital Experience Konferenz.
</speak> 
```

```
<speak>   
    Noch ein Geheimtipp von mir: 
    <amazon:effect name="whispered">Nach der Sparkscon steigt eine after show party</amazon:effect>.
</speak>
```
## Variable Handling

### Session Variablen (Attributemanager)
```
const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();

sessionAttributes['food'] = food;
handlerInput.attributesManager.setSessionAttributes(sessionAttributes);
```

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

```
.addRequestHandlers(
	...,
	WhatIsMyFavoriteFoodHandler,
	...)
```

### Variablen persistieren
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

```
.addRequestInterceptors(
	...
	LoadAttributesRequestInterceptor)
.addResponseInterceptors(
  	...
	SaveAttributesResponseInterceptor)
.withPersistenceAdapter(persistenceAdapter)
```



## Notitzen

Alexa Conversations

Vorteile:

Conversations liegen parallel zum Interaction Model
Beispiel Konversationen geben Raum für Variationen. Müssen nicht selbst erzeugt werden.
Happy Path kann schneller erreicht werden
Kontext: Abhängigkeiten zwischen Intents können leichter aufgelößt werden
