# Alexa GPT

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Boost your Alexa by making it respond as ChatGPT.

This repository contains a tutorial on how to create a simple Alexa skill that uses the OpenAI API to generate responses from the ChatGPT model.

<div align="center">
  <img src="images/test.png" />
</div>

## Prerequisites

- An [Amazon Developer account](https://developer.amazon.com/)
- An [OpenAI API key](https://platform.openai.com/api-keys)

## Step-by-step tutorial

### 1. <span name=item-1></span>
Log in to your Amazon Developer account and navigate to the [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask).

### 2.
Click on "Create Skill" and name the skill "Chat". Choose the primary locale according to your language.

![name your skill](images/name_your_skill.png)

### 3.
Choose "Other" and "Custom" for the model.

![type of experience](images/type_of_experience.png)

![choose a model](images/choose_a_model.png)

### 4.
Choose "Alexa-hosted (Python)" for the backend resources.

![hosting services](images/hosting_services.png)

### 5.
You now have two options:
- Click on "Import Skill", paste the link of this repository (https://github.com/k4l1sh/alexa-gpt.git) and click on "Import".
![template](images/import_git_skill.png)

Or if you want to create the skill manually
- Select "Start from Scratch" and click on "Create Skill"

![template](images/select_template.png)

### 6.
In the "Build" section, navigate to the "JSON Editor" tab.

### 7.
If you have directly imported the skill from this repository, just change the "invocationName" to "chat" or another preferred word for activation and proceed to [step 12](#12).

However, if you chose to manually create the skill, replace the existing JSON content with the [provided JSON content](json_editor.json):

```json
{
    "interactionModel": {
        "languageModel": {
            "invocationName": "chat",
            "intents": [
                {
                    "name": "GptQueryIntent",
                    "slots": [
                        {
                            "name": "query",
                            "type": "AMAZON.Person"
                        }
                    ],
                    "samples": [
                        "{query}"
                    ]
                },
                {
                    "name": "AMAZON.CancelIntent",
                    "samples": []
                },
                {
                    "name": "AMAZON.HelpIntent",
                    "samples": []
                },
                {
                    "name": "AMAZON.StopIntent",
                    "samples": []
                },
                {
                    "name": "AMAZON.NavigateHomeIntent",
                    "samples": []
                }
            ],
            "types": []
        }
    }
}
```

![json_editor](images/intents_json_editor.png)

### 8.
Save the model and click on "Build Model".

### 9.
Go to "Code" section and add "openai" to requirements.txt. Your requirements.txt should look like this:

```txt
ask-sdk-core==1.11.0
boto3==1.9.216
requests>=2.20.0
```

### 10.
Create an OpenAI API key on the [API keys page](https://platform.openai.com/api-keys) by clicking "+ Create new secret key".

### 11.
Replace your lambda_functions.py file with the [provided lambda_function.py](lambda/lambda_function.py).

```python
from ask_sdk_core.dispatch_components import AbstractExceptionHandler
from ask_sdk_core.dispatch_components import AbstractRequestHandler
from ask_sdk_core.skill_builder import SkillBuilder
from ask_sdk_core.handler_input import HandlerInput
from ask_sdk_model import Response
import ask_sdk_core.utils as ask_utils
import requests
import logging
import json

# Set your OpenAI API key
api_key = "YOUR_API_KEY"

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

class LaunchRequestHandler(AbstractRequestHandler):
    """Handler for Skill Launch."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool

        return ask_utils.is_request_type("LaunchRequest")(handler_input)

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response
        speak_output = "Chat G.P.T. mode activated"

        session_attr = handler_input.attributes_manager.session_attributes
        session_attr["chat_history"] = []

        return (
            handler_input.response_builder
                .speak(speak_output)
                .ask(speak_output)
                .response
        )

class GptQueryIntentHandler(AbstractRequestHandler):
    """Handler for Gpt Query Intent."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool
        return ask_utils.is_intent_name("GptQueryIntent")(handler_input)

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response
        query = handler_input.request_envelope.request.intent.slots["query"].value

        session_attr = handler_input.attributes_manager.session_attributes
        if "chat_history" not in session_attr:
            session_attr["chat_history"] = []
        response = generate_gpt_response(session_attr["chat_history"], query)
        session_attr["chat_history"].append((query, response))

        return (
                handler_input.response_builder
                    .speak(response)
                    .ask("Any other questions?")
                    .response
            )

class CatchAllExceptionHandler(AbstractExceptionHandler):
    """Generic error handling to capture any syntax or routing errors."""
    def can_handle(self, handler_input, exception):
        # type: (HandlerInput, Exception) -> bool
        return True

    def handle(self, handler_input, exception):
        # type: (HandlerInput, Exception) -> Response
        logger.error(exception, exc_info=True)

        speak_output = "Sorry, I had trouble doing what you asked. Please try again."

        return (
            handler_input.response_builder
                .speak(speak_output)
                .ask(speak_output)
                .response
        )

class CancelOrStopIntentHandler(AbstractRequestHandler):
    """Single handler for Cancel and Stop Intent."""
    def can_handle(self, handler_input):
        # type: (HandlerInput) -> bool
        return (ask_utils.is_intent_name("AMAZON.CancelIntent")(handler_input) or
                ask_utils.is_intent_name("AMAZON.StopIntent")(handler_input))

    def handle(self, handler_input):
        # type: (HandlerInput) -> Response
        speak_output = "Leaving Chat G.P.T. mode"

        return (
            handler_input.response_builder
                .speak(speak_output)
                .response
        )

def generate_gpt_response(chat_history, new_question):
    """Generates a GPT response to a new question"""
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    url = "https://api.openai.com/v1/chat/completions"
    messages = [{"role": "system", "content": "You are a helpful assistant. Answer in 50 words or less."}]
    for question, answer in chat_history[-10:]:
        messages.append({"role": "user", "content": question})
        messages.append({"role": "assistant", "content": answer})
    messages.append({"role": "user", "content": new_question})
    
    data = {
        "model": "gpt-4o-mini",
        "messages": messages,
        "max_tokens": 300,
        "temperature": 0.5
    }
    try:
        response = requests.post(url, headers=headers, data=json.dumps(data))
        response_data = response.json()
        if response.ok:
            return response_data['choices'][0]['message']['content']
        else:
            return f"Error {response.status_code}: {response_data['error']['message']}"
    except Exception as e:
        return f"Error generating response: {str(e)}"

sb = SkillBuilder()

sb.add_request_handler(LaunchRequestHandler())
sb.add_request_handler(GptQueryIntentHandler())
sb.add_request_handler(CancelOrStopIntentHandler())
sb.add_exception_handler(CatchAllExceptionHandler())

lambda_handler = sb.lambda_handler()
```

### 12.
Put your OpenAI API key that you got from your [OpenAI account](https://platform.openai.com/api-keys)

![openai_api_key](images/api_key.png)

### 13.
Save and deploy. Go to "Test" section and enable "Skill testing" in "Development".

![development_enabled](images/development_enabled.png)

### 14.
You are now ready to use your Alexa in ChatGPT mode. You should see results like this:

![test](images/test.png)

Please note that running this skill will incur costs for using both AWS Lambda and the OpenAI API. Make sure you understand the pricing structure and monitor your usage to avoid unexpected charges.