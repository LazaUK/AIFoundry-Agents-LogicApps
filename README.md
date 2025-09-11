# Azure AI Foundry Agent Service: Using Logic Apps as Tools

This repository demonstrates how to integrate **Azure Logic Apps** as custom tools for an Azure AI Foundry agent.

> [!NOTE]
> For more in-depth information about the Azure AI Agent Service, refer to the official [Azure AI Foundry documentation](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview).

***

## 📑 Table of Contents:
- [Part 1: Configuring the Environment](#part-1-configuring-the-environment)
- [Part 2: Configuring Logic App as a Tool using Function Calling](#part-2-configuring-logic-app-as-a-tool-using-function-calling)
- [Part 3: Configuring Logic App as a Tool using OpenAPI Schema](#part-3-configuring-logic-app-as-a-tool-using-openapi-schema)

## Part 1: Configuring the Environment
To run the provided Jupyter notebooks, you'll need to set up your Azure AI Foundry environment and install the required Python packages.

### 1.1 Prerequisites
You need an **Azure AI Foundry** project with a model deployment that supports function calling (e.g., GPT-4.1-mini). You also need a pre-existing Azure Logic App with an **HTTP Trigger** configured to accept incoming requests.

### 1.2 Authentication
The demo uses **Microsoft Entra ID** authentication via `DefaultAzureCredential` from the `azure.identity` package. To enable this, ensure you are authenticated through the Azure CLI (`az login`), by setting up environment variables for a service principal, or by using a managed identity in an Azure environment.

### 1.3 Environment Variables
Configure the following environment variables with your Azure credentials and project details:

| Environment Variable             | Description                                               |
| :------------------------------- | :-------------------------------------------------------- |
| `AZURE_FOUNDRY_PROJECT_ENDPOINT` | The endpoint URL for your Azure AI Foundry project.       |
| `AZURE_FOUNDRY_GPT_MODEL`        | The name of your model deployment.                        |
| `AZURE_SUBSCRIPTION_ID`          | Your Azure subscription ID.                               |
| `RESOURCE_GROUP_NAME`            | The name of the resource group containing your Logic App. |

### 1.4 Required Libraries
Install the necessary Python packages using pip:

``` bash
pip install -r requirements.txt
```

***

## Part 2: Configuring Logic App as a Tool using Function Calling
This section details how to wrap a Logic App's functionality within a Python function and expose it to the AI agent as a tool.

### 2.1 The `AzureLogicAppTool` Class
A dedicated class, `AzureLogicAppTool`, is used to manage the Logic App. It performs two key tasks:
* `register_logic_app`: This method retrieves the unique callback URL for the Logic App's HTTP-Trigger. It uses the `LogicManagementClient` to securely obtain this URL, which is required to invoke the Logic App.
* `invoke_logic_app`: This method sends a `POST` request to the registered callback URL with a JSON payload, effectively triggering the Logic App workflow.

``` Python
class AzureLogicAppTool:
    """
    A service that manages Logic Apps by retrieving their callback URLs and invoking them.
    """

    def __init__(self, subscription_id: str, resource_group: str, credential=None):
        if credential is None:
            credential = DefaultAzureCredential()
        self.subscription_id = subscription_id
        self.resource_group = resource_group
        self.logic_client = LogicManagementClient(credential, subscription_id)
        self.callback_urls: Dict[str, str] = {}

    def register_logic_app(self, logic_app_name: str, trigger_name: str) -> None:
        """
        Retrieves and stores a callback URL for a specific Logic App + trigger.
        """
        <...>

    def invoke_logic_app(self, logic_app_name: str, payload: Dict[str, Any]) -> Dict[str, Any]:
        """
        Invokes the registered Logic App with the given JSON payload.
        """
        <...>
```

### 2.2 Python Wrapper Function
A helper function, `create_weather_forecast_function`, acts as a bridge between the AI agent and the `AzureLogicAppTool`:
* Accepting a `location` parameter from the agent.
* Constructing the necessary JSON `payload`.
* Calling the `invoke_logic_app` method from the `AzureLogicAppTool` class.
* Formating the result into a standardized JSON response that the agent can easily parse.

``` Python
def create_weather_forecast_function(logic_app_tool: AzureLogicAppTool, logic_app_name: str) -> Callable[[str], str]:
    """
    Creates a weather forecast function that can be used by the AI agent.
    """
    
    def get_weather_forecast(location: str) -> str:
        """
        Gets weather forecast for a specific location using the Logic App.
        
        :param location: The location to get weather forecast for (e.g., "London", "New York", "Tokyo")
        :return: A JSON string containing the weather forecast information
        """
        payload = {"Location": location}
        
        try:
            result = logic_app_tool.invoke_logic_app(logic_app_name, payload)
            
            if result.get("result") == "success":
                weather_info = result.get("weather_data", result.get("data", "Weather data received"))
                return json.dumps({
                    "status": "success",
                    "location": location,
                    "forecast": weather_info
                })
            else:
                return json.dumps({
                    "status": "error",
                    "location": location,
                    "error": result.get("error", "Unknown error occurred")
                })
                
        except Exception as e:
            return json.dumps({
                "status": "error",
                "location": location,
                "error": str(e)
            })
    
    return get_weather_forecast
```

### 2.3 Agent Configuration
With the Python wrapper function defined, you can now configure the AI agent:
1.  **Define Agent Tools**: The `get_weather_forecast` function is added to a `FunctionTool` instance. A separate `get_current_datetime` function is also included to demonstrate the agent's ability to use multiple tools.
2.  **Enable Function Calling**: The `agents_client.enable_auto_function_calls()` method is used to instruct the agent's model to automatically call the defined tools when appropriate.
3.  **Create the Agent**: An agent is created with a clear instruction set that guides it to use the `get_weather_forecast` tool when asked about the weather.

The output of AI agent's setup may look like this:

``` JSON
Setting up agent tools...
Creating weather agent...
Created weather agent with ID: asst_P24uYTY3zZhH2VFnP6on62lt
Creating conversation thread...
Created thread with ID: thread_au3fmRys3MadIZTnCppkDHmf
Agent is ready! You can now interact with it.
```

***

## Part 3: Configuring Logic App as a Tool using OpenAPI Schema

> [!ATTENTION]
> *Coming soon! This section will describe how to configure an agent to use a Logic App via an OpenAPI schema, which offers a more declarative approach to defining tool capabilities.*
