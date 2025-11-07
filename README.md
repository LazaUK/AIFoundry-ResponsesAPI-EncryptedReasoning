# Azure AI Foundry: Encrypted Reasoning with Responses API
This repo demonstrates the use of **Encrypted Reasoning** feature of AI Foundry's _Responses API_ to maintain model intelligence across multi-turn, stateless conversations while using function calling.

The use of encrypted reasoning is applicable to two main scenarios:
- `Zero Data Retention (ZDR)`: When your organisation has strict data retention policies and sets store=False (stateless mode) to prevent the conversation history from persisting on AI Foundry deployments.
- `Maintaining Model Intelligence`: Reasoning models (like OpenAI o-series) perform an internal "_chain-of-thought_" that is not exposed as plaintext. Passing the encrypted reasoning from one turn back in the next turn's input allows the model to leverage its previous thinking, leading to higher performance and more cost-effective token utilisation.

> [!NOTE]
> This demo uses a simulated payment processing function to show how the reasoning model analyses complex tool outputs (including a simulated risk assessment) across multiple steps.

## 📑 Table of Contents:
- [Part 1: Configuring Solution Environment](#part-1-configuring-solution-environment)
- [Part 2: Defining the Business Tool](#part-2-defining-the-business-tool)
- [Part 3: Stateless Function Calling with Encrypted Reasoning]()

## Part 1: Configuring Solution Environment
To run the provided Jupyter notebook, you'll need to set up your Azure AI Foundry environment and install the required Python packages.

### 1.1 Azure OpenAI Service Setup
Ensure you have an Azure AI Foundry project with a model deployment that supports reasoning and function calling (e.g., _o4-mini_).

### 1.2 Authentication
This notebook uses _Microsoft Entra ID_ authentication via **DefaultAzureCredential** from the _azure.identity_ package. Define a token provider using the **get_bearer_token_provider()** function to secure your client initialisation:

``` Python
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "https://cognitiveservices.azure.com/.default"
)
```

### 1.3 Environment Variables
Configure the following environment variables for your Azure AI Foundry deployment:

| Environment Variable                | Description                                                                          |
| :---------------------------------- | :----------------------------------------------------------------------------------- |
| `AZURE_OPENAI_API_BASE`             | Azure AI Foundry endpoint URL (e.g., https://<YOUR_AOAI_RESOURCE>.openai.azure.com). |
| `AZURE_OPENAI_API_VERSION`          | The API version (e.g., 2025-04-01-preview).                                          |
| `AZURE_OPENAI_API_DEPLOY_REASONING` | The name of your model deployment (e.g., o4-mini).                                   |

### 1.4 Installation of Required Python Packages
Install the necessary packages:

``` Bash
pip install openai azure-identity
```

### 1.5 Azure OpenAI Client Setup
Initialise the AzureOpenAI client using your environment variables and the Entra ID token provider:

``` Python
from openai import AzureOpenAI

client = AzureOpenAI(  
    azure_endpoint = AOAI_API_BASE,
    azure_ad_token_provider = token_provider,
    api_version = AOAI_API_VERSION,
)
```

## Part 2: Defining the Business Tool
The demo solution uses a custom function, _process_credit_card_transaction_, which simulates a payment gateway and returns a detailed risk assessment (status, risk score, risk factors). This complexity requires the model to reason before presenting the final answer.

### 2.1 Function Definition
The function and its tool schema are defined in the notebook, providing a robust set of inputs and a detailed JSON output.

``` Python
tools = [
    {
        "type": "function",
        "name": "process_credit_card_transaction",
        "description": "Processes a credit card payment securely. Returns detailed risk assessment with specific factors analyzed.",
        "parameters": {
            "type": "object",
            "properties": {
                "card_number": {
                    "type": "string",
                    "description": "The 16-digit credit card number"
                },
                "expiry_date": {
                    "type": "string",
                    "description": "Card expiry date in MM/YY format"
                },
                "cvv": {
                    "type": "string",
                    "description": "The 3 or 4-digit security code"
                },
                "amount": {
                    "type": "number",
                    "description": "The transaction amount to charge"
                },
                "merchant_id": {
                    "type": "string",
                    "description": "The identifier for the merchant"
                }
            },
            "required": ["card_number", "expiry_date", "cvv", "amount", "merchant_id"]
        }
    }
]
```

## Part 3: Stateless Function Calling with Encrypted Reasoning
The core of this example is demonstrating how to retrieve and reuse the encrypted reasoning content across multiple API calls.

### 3.1 Step 1: Initial Call and Reasoning Extraction
The first API call is made with two critical parameters:
- `store=False`: Enforces statelessness (no conversation history is stored on the backend side).
- `include=["reasoning.encrypted_content"]`: Requests the model's internal reasoning chain to be returned as an encrypted object.

The model generates a tool call, and the resulting response includes the _reasoning_ item with the _encrypted_content_ populated.

``` Python
response_1 = client.responses.create(
    model=AOAI_DEPLOYMENT,
    input=[user_request],
    tools=tools,
    reasoning={"effort": "low"},
    store=False, # Stateless mode
    include=["reasoning.encrypted_content"] # Request encrypted data
)

# Extract the encrypted reasoning item from response_1.output
reasoning_item = response_1.output[0]
tool_call_item = response_1.output[1]
```

### 3.2 Step 2: Reusing Encrypted Reasoning in the Next Turn
For the subsequent turn, a new _input_ array is constructed, which includes all items from the previous turn plus the new user request. Crucially, the _reasoning_item_ with the **encrypted_content** is passed back into the input context.

This object is now part of the conversation history for the model to use for its internal thinking process in the current turn.

``` Python
follow_up_context = [
    user_request,
    reasoning_item, # REUSED ENCRYPTED REASONING
    tool_call_item,
    {
        "type": "function_call_output",
        "call_id": tool_call_item.call_id,
        "output": json.dumps(transaction_result) # Function output
    },
    {
        "role": "user",
        "content": "Explain your risk assessment for this transaction."
    }
]

response_2 = client.responses.create(
    model=AOAI_DEPLOYMENT,
    current_context=follow_up_context,
    tools=tools,
    store=False # Still stateless
)
```

This ensures that the model can correctly analyse and explain the multi-step risk assessment from the function output without needing a full, server-side stored history.
