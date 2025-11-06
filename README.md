# Azure AI Foundry: Encrypted Reasoning with Responses API

This repo demonstrates the use of **Encrypted Reasoning** feature of AI Foundry's _Responses API_ to maintain model intelligence across multi-turn, stateless conversations while using function calling.

The use of encrypted reasoning is applicable to two main scenarios:
- `Zero Data Retention (ZDR)`: When your organisation has strict data retention policies and sets store=False (stateless mode) to prevent the conversation history from persisting on AI Foundry deployments.
- `Maintaining Model Intelligence`: Reasoning models (like OpenAI o-series) perform an internal "_chain-of-thought_" that is not exposed as plaintext. Passing the encrypted reasoning from one turn back in the next turn's input allows the model to leverage its previous thinking, leading to higher performance and more cost-effective token utilisation.

> [!NOTE]
> This demo uses a simulated payment processing function to show how the reasoning model analyses complex tool outputs (including a simulated risk assessment) across multiple steps.

## 📑 Table of Contents:
- [Part 1: Configuring Solution Environment]()
- [Part 2: Defining the Business Tool]()
- [Part 3: Stateless Function Calling with Encrypted Reasoning]()

## Part 1: Configuring Solution Environment
To run the provided notebook, you'll need to set up your Azure OpenAI environment and install the required Python packages.1.1 Azure OpenAI Service SetupEnsure you have an Azure OpenAI Service resource with a model deployment that supports reasoning and function calling (e.g., a modern GPT-4 deployment).1.2 AuthenticationThis notebook uses Microsoft Entra ID authentication via DefaultAzureCredential from the azure.identity package.Define a token provider using the get_bearer_token_provider() function to secure your client initialization:Pythontoken_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "https://cognitiveservices.azure.com/.default"
)
1.3 Environment VariablesConfigure the following environment variables for your Azure OpenAI deployment:Environment VariableDescriptionAZURE_OPENAI_API_BASEYour Azure OpenAI endpoint URL (e.g., https://<YOUR_AOAI_RESOURCE>.openai.azure.com).AZURE_OPENAI_API_VERSIONThe API version (e.g., 2025-04-01-preview).AZURE_OPENAI_API_DEPLOY_REASONINGThe name of your model deployment (e.g., gpt-4-1106-preview).1.4 Installation of Required Python PackagesInstall the necessary packages:Bashpip install openai azure-identity python-dotenv
1.5 Azure OpenAI Client SetupInitialise the AzureOpenAI client using your environment variables and the Entra ID token provider:Pythonfrom openai import AzureOpenAI

client = AzureOpenAI(  
    azure_endpoint = AOAI_API_BASE,
    azure_ad_token_provider = token_provider,
    api_version = AOAI_API_VERSION,
)
Part 2: Defining the Business ToolThe demonstration uses a custom function, process_credit_card_transaction, which simulates a payment gateway and returns a detailed risk assessment (status, risk score, risk factors). This complexity requires the model to reason before presenting the final answer.2.1 Function DefinitionThe function and its tool schema are defined in the notebook, providing a robust set of inputs and a detailed JSON output.Tool NameDescriptionprocess_credit_card_transactionProcesses a credit card payment securely and returns a detailed risk assessment with specific factors analyzed.Part 3: Stateless Function Calling with Encrypted ReasoningThe core of this example is demonstrating how to retrieve and reuse the encrypted reasoning content across multiple API calls, ensuring the model maintains context in a stateless environment (store=False).3.1 Step 1: Initial Call and Reasoning ExtractionThe first API call is made with two critical parameters:store=False: Enforces statelessness (no conversation history is stored on the server).include=["reasoning.encrypted_content"]: Requests the model's internal reasoning chain to be returned as an encrypted object.The model generates a tool call, and the resulting response includes the reasoning item with the encrypted_content populated.API Call Snippet:Pythonresponse_1 = client.responses.create(
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
3.2 Step 2: Reusing Encrypted Reasoning in the Next TurnFor the subsequent turn, a new input array is constructed, which includes all items from the previous turn plus the new user request. Crucially, the reasoning_item with the encrypted_content is passed back into the input context.This object is now part of the conversation history for the model to use for its internal thinking process in the current turn.Follow-up Context Structure:JSON[
    { /* Previous User Request */ },
    { /* Reasoning Item from Turn 1 with encrypted_content */ },
    { /* Tool Call Item from Turn 1 */ },
    { /* Function Call Output Item from Turn 1 (Transaction Result) */ },
    { /* New User Request: "Explain your risk assessment..." */ }
]
Second API Call Snippet:Pythonfollow_up_context = [
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
# Model returns detailed explanation based on its previous internal analysis.
This ensures that the model can correctly analyze and explain the multi-step risk assessment from the function output without needing a full, server-side stored history.
