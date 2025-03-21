# Agents SDK for PHP

A lightweight yet powerful framework for building multi-agent workflows in PHP, inspired by the OpenAI Agents SDK.

[![PHP from Packagist](https://img.shields.io/packagist/php-v/swisnl/agents-sdk.svg)](https://packagist.org/packages/swisnl/agents-sdk)
[![Latest Version on Packagist](https://img.shields.io/packagist/v/swisnl/agents-sdk.svg)](https://packagist.org/packages/swisnl/agents-sdk)
[![Software License](https://img.shields.io/packagist/l/swisnl/agents-sdk.svg)](LICENSE.md)
[![Buy us a tree](https://img.shields.io/badge/Treeware-%F0%9F%8C%B3-lightgreen.svg)](https://plant.treeware.earth/swisnl/agents-sdk)
[![Build Status](https://img.shields.io/github/checks-status/swisnl/agents-sdk/master?label=tests)](https://github.com/swisnl/agents-sdk/actions/workflows/tests.yml)
[![Made by SWIS](https://img.shields.io/badge/%F0%9F%9A%80-made%20by%20SWIS-%230737A9.svg)](https://www.swis.nl)

## Overview

Agents SDK provides an elegant abstraction for creating AI agent systems in PHP, allowing you to:

- Build specialized agents for different tasks
- Connect agents using a handoff mechanism
- Define and use custom tools for external operations
- Stream LLM responses for real-time interactions
- Monitor agent behavior with observers and traces

The SDK is designed to be flexible, extensible, and easy to use while providing a robust foundation for building complex multi-agent based systems.

## Installation

```bash
composer require swisnl/agents-sdk
```

## Basic Usage

Here's a simple example of creating and running an Agent that can use a Tool for retrieving weather information:

```php
use Swis\Agents\Agent;
use Swis\Agents\Orchestrator;
use Swis\Agents\Tool;
use Swis\Agents\Tool\Required;
use Swis\Agents\Tool\ToolParameter;

// Define a custom tool
class WeatherTool extends Tool
{
    #[ToolParameter('The name of the city.'), Required]
    public string $city;

    protected ?string $toolDescription = 'Gets the current weather by city.';

    public function __invoke(): ?string
    {
        // Implementation to fetch weather data
        return "Current weather in {$this->city}: Sunny, 22°C";
    }
}

// Create an agent with the tool
$agent = new Agent(
    name: 'Weather Assistant',
    description: 'Provides weather information',
    instruction: 'You help users with weather-related questions. Use the WeatherTool to get accurate data.',
    tools: [new WeatherTool()]
);

// Set up the orchestrator
$orchestrator = new Orchestrator();

// Process a user message
$orchestrator->withUserInstruction('What\'s the weather like in Amsterdam?');

// Run the agent and get the response
$response = $orchestrator->run($agent);
echo $response;

// Or use streaming for real-time responses
$orchestrator->runStreamed($agent, function ($token) {
    echo $token;
});
```

## Creating Agents

Agents are the core components of the SDK. They encapsulate a specific role or capability and can use tools to perform actions.

```php
$agent = new Agent(
    name: 'Agent Name',             // Required: Unique identifier for the agent
    description: 'Description',     // Optional: Brief description of the agent's capabilities
    instruction: 'System prompt',   // Optional: Detailed instructions for the agent
    tools: [$tool1, $tool2],        // Optional: Array of tools the agent can use
    handoffs: [$otherAgent]         // Optional: Other agents this agent can hand off to
);
```

## Defining Tools

Tools are capabilities that agents can use to perform actions. To create a custom tool:

1. Extend the `Tool` class
2. Define parameters using attributes
3. Implement the `__invoke` method with your tool's logic

```php
class SearchTool extends Tool
{
    #[ToolParameter('The search query.'), Required]
    public string $query;
    
    #[ToolParameter('The number of results to return.')]
    public int $limit = 5;

    protected ?string $toolDescription = 'Searches for information.';

    public function __invoke(): ?string
    {
        // Implementation logic here
        return json_encode([
            'results' => [/* search results */]
        ]);
    }
}
```

## Multi-Agent Systems

The SDK supports creating systems of specialized agents that can hand off tasks to each other:

```php
// Create specialized agents
$weatherAgent = new Agent(
    name: 'Weather Agent',
    // ... configuration
);

$travelAgent = new Agent(
    name: 'Travel Agent',
    // ... configuration
    handoffs: [$weatherAgent]  // Travel agent can hand off to Weather agent
);

// Main triage agent
$triageAgent = new Agent(
    name: 'Triage Agent',
    description: 'Routes user requests to the appropriate specialized agent',
    handoffs: [$weatherAgent, $travelAgent]
);
```

## Orchestration

The `Orchestrator` class manages the conversation flow and agent execution:

```php
$orchestrator = new Orchestrator();

// Add a user message
$orchestrator->withUserInstruction("I need help with planning a trip to Amsterdam");

// Run with a specific agent
$response = $orchestrator->run($triageAgent);

// Or stream the response
$orchestrator->runStreamed($triageAgent, function ($token) {
    echo $token;
});
```

## Observability

The SDK provides an observer pattern to monitor agent behavior:

```php
$orchestrator->withAgentObserver(new class extends AgentObserver {
    public function beforeHandoff(AgentInterface $agent, AgentInterface $handoffToAgent, RunContext $context): void
    {
        echo "Handing off from {$agent->name()} to {$handoffToAgent->name()}\n";
    }

    public function onToolCall(AgentInterface $agent, Tool $tool, ToolCall $toolCall, RunContext $context): void
    {
        echo "Agent {$agent->name()} called tool: {$toolCall->tool}\n";
    }
});
```

## Tracing

The SDK includes built-in tracing support using OpenAI Tracing format by default. This helps with debugging and monitoring agent execution.

### Disabling Tracing

You can disable tracing in two ways:

1. On a per-orchestrator basis:
```php
$orchestrator = new Orchestrator();
$orchestrator->disableTracing();
```

2. Globally via environment variable:
```
AGENTS_SDK_DISABLE_TRACING=true
```

Disabling tracing can be useful in production environments or when you don't need the debugging information.

## Example Implementations

The repository includes examples for common use cases:

- `CustomerServiceAgent`: A multi-agent system for customer service with handoffs
- `WeatherAgent`: A simple agent that fetches weather information

Run the examples using:

```php
php examples/play.php
```

## Conversation Serialization

The SDK provides a way to serialize and deserialize conversation state, allowing you to continue conversations at a later time:

```php
use Swis\Agents\Helpers\ConversationSerializer;

// After running some conversation with an orchestrator
$data = ConversationSerializer::serializeFromOrchestrator($orchestrator);
saveToStorage($data); // Your storage implementation

// Later, when you want to continue the conversation
$data = retrieveFromStorage(); // Your retrieval implementation

// Create a new orchestrator with the serialized conversation
$orchestrator = new Orchestrator();
$orchestrator->withContextFromData($data);

$agent = new Agent(/* Create your agent */);

// Continue the conversation
$orchestrator->withUserInstruction('New message');
$response = $orchestrator->run($agent);
```

## Requirements

- PHP 8.2 or higher
- OpenAI API key for LLM access (or other OpenAI compatible API)
- Composer for dependency management

## Testing

The SDK includes a test suite built with PHPUnit. To run the tests:

```bash
# Install dependencies
composer install

# Run the tests
composer run test
```

### Test Structure

- **Unit Tests**: Test individual components in isolation
- **Integration Tests**: Test the full agent workflow with actual API calls (skipped by default)

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Security Vulnerabilities

Please review [our security policy](../../security/policy) on how to report security vulnerabilities.

## Credits

- [Joris Meijer](https://github.com/jormeijer)
- [Björn Brala](https://github.com/bbrala)
- [All Contributors](../../contributors)

## License

This package is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
