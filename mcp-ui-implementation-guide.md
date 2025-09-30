# Complete Guide to Implementing MCP-UI in TypeScript MCP Servers

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites & Setup](#prerequisites--setup)
3. [Basic Implementation](#basic-implementation)
4. [Intermediate Patterns](#intermediate-patterns)
5. [Advanced Design Patterns](#advanced-design-patterns)
6. [Non-Obvious High-Value Patterns](#non-obvious-high-value-patterns)

---

## Introduction

MCP-UI is a framework for delivering interactive UI components through the Model Context Protocol (MCP). It enables MCP servers to generate rich, interactive interfaces that clients can render seamlessly, bridging the gap between server-side logic and client-side user experience.

### Key Components

- **@mcp-ui/server**: Server-side utilities for generating UI resources
- **@mcp-ui/client**: Client-side components for rendering UI resources
- **UIResource**: The core data structure representing a UI component

---

## Prerequisites & Setup

### Required Tools

- Node.js v22.x or later
- pnpm v9+ (or npm/yarn)
- TypeScript 5.0+

### Installation

```bash
# Install core MCP SDK
npm install @modelcontextprotocol/sdk

# Install MCP-UI packages
npm install @mcp-ui/server @mcp-ui/client

# Install additional dependencies (if using Express)
npm install express cors
npm install -D @types/express @types/cors @types/node
```

### Project Structure

```
my-mcp-server/
├── src/
│   ├── server.ts          # Main server setup
│   ├── tools/             # Tool implementations
│   ├── ui/                # UI resource factories
│   └── types/             # Type definitions
├── package.json
└── tsconfig.json
```

---

## Basic Implementation

### 1. Simple Raw HTML UI Resource

The most straightforward way to create a UI resource is using inline HTML:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { createUIResource } from '@mcp-ui/server';

const server = new McpServer({
  name: "simple-ui-server",
  version: "1.0.0"
});

// Register a tool that returns a simple HTML UI
server.registerTool('show-welcome', {
  title: 'Show Welcome Message',
  description: 'Displays a welcome message UI',
  inputSchema: {
    type: 'object',
    properties: {
      username: {
        type: 'string',
        description: 'User name to display'
      }
    },
    required: ['username']
  }
}, async (args) => {
  const username = args.username as string;
  
  const htmlContent = `
    <div style="padding: 20px; border: 2px solid #4CAF50; border-radius: 8px;">
      <h1>Welcome, ${username}!</h1>
      <p>This is your personalized dashboard.</p>
    </div>
  `;

  const uiResource = createUIResource({
    uri: `ui://welcome/${username}`,
    content: { 
      type: 'rawHtml', 
      htmlString: htmlContent 
    },
    encoding: 'text',
  });

  return {
    content: [uiResource],
  };
});
```

### 2. External URL UI Resource

For embedding external content:

```typescript
server.registerTool('show-dashboard', {
  title: 'Show Dashboard',
  description: 'Displays an external dashboard',
  inputSchema: {
    type: 'object',
    properties: {}
  }
}, async () => {
  const uiResource = createUIResource({
    uri: 'ui://dashboard',
    content: { 
      type: 'externalUrl', 
      iframeUrl: 'https://example.com/dashboard'
    },
    encoding: 'text',
  });

  return {
    content: [uiResource],
  };
});
```

### 3. Basic Client-Side Rendering

```tsx
import React from 'react';
import { UIResourceRenderer } from '@mcp-ui/client';

interface AppProps {
  uiResource: any; // UIResource type
}

function App({ uiResource }: AppProps) {
  const handleUIAction = (action: any) => {
    console.log('UI Action received:', action);
    // Handle actions triggered from the UI
  };

  return (
    <div className="app-container">
      <UIResourceRenderer
        resource={uiResource}
        onUIAction={handleUIAction}
      />
    </div>
  );
}

export default App;
```

---

## Intermediate Patterns

### 1. UI Resource Factory Pattern

Create reusable factories for common UI patterns:

```typescript
// ui/factories.ts
import { createUIResource } from '@mcp-ui/server';

export interface CardUIOptions {
  title: string;
  content: string;
  variant?: 'info' | 'warning' | 'error' | 'success';
  actions?: Array<{ label: string; id: string }>;
}

export class UIResourceFactory {
  static createCard(options: CardUIOptions) {
    const { title, content, variant = 'info', actions = [] } = options;
    
    const colors = {
      info: '#2196F3',
      warning: '#FF9800',
      error: '#F44336',
      success: '#4CAF50'
    };

    const actionsHtml = actions.map(action => 
      `<button onclick="parent.postMessage({type: 'action', id: '${action.id}'}, '*')" 
               style="margin: 5px; padding: 8px 16px; cursor: pointer;">
        ${action.label}
      </button>`
    ).join('');

    const htmlContent = `
      <!DOCTYPE html>
      <html>
        <head>
          <style>
            body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; margin: 0; }
            .card {
              border-left: 4px solid ${colors[variant]};
              padding: 20px;
              background: white;
              box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            }
            .card h2 { margin-top: 0; color: ${colors[variant]}; }
            .actions { margin-top: 15px; }
          </style>
        </head>
        <body>
          <div class="card">
            <h2>${title}</h2>
            <p>${content}</p>
            ${actions.length > 0 ? `<div class="actions">${actionsHtml}</div>` : ''}
          </div>
        </body>
      </html>
    `;

    return createUIResource({
      uri: `ui://card/${Date.now()}`,
      content: { type: 'rawHtml', htmlString: htmlContent },
      encoding: 'text',
    });
  }

  static createDataTable(data: Array<Record<string, any>>) {
    if (data.length === 0) {
      return this.createCard({
        title: 'No Data',
        content: 'No data available to display',
        variant: 'info'
      });
    }

    const columns = Object.keys(data[0]);
    const headerHtml = columns.map(col => `<th>${col}</th>`).join('');
    const rowsHtml = data.map(row => 
      `<tr>${columns.map(col => `<td>${row[col]}</td>`).join('')}</tr>`
    ).join('');

    const htmlContent = `
      <!DOCTYPE html>
      <html>
        <head>
          <style>
            body { font-family: sans-serif; margin: 0; padding: 20px; }
            table { width: 100%; border-collapse: collapse; }
            th, td { padding: 12px; text-align: left; border-bottom: 1px solid #ddd; }
            th { background-color: #f5f5f5; font-weight: 600; }
            tr:hover { background-color: #f9f9f9; }
          </style>
        </head>
        <body>
          <table>
            <thead><tr>${headerHtml}</tr></thead>
            <tbody>${rowsHtml}</tbody>
          </table>
        </body>
      </html>
    `;

    return createUIResource({
      uri: `ui://table/${Date.now()}`,
      content: { type: 'rawHtml', htmlString: htmlContent },
      encoding: 'text',
    });
  }
}
```

### 2. Stateful UI Resources with Actions

Implement interactive UIs that communicate back to the server:

```typescript
// tools/interactive-form.ts
import { UIResourceFactory } from '../ui/factories';

server.registerTool('create-feedback-form', {
  title: 'Create Feedback Form',
  description: 'Creates an interactive feedback form',
  inputSchema: {
    type: 'object',
    properties: {
      topic: { type: 'string' }
    },
    required: ['topic']
  }
}, async (args) => {
  const topic = args.topic as string;

  const htmlContent = `
    <!DOCTYPE html>
    <html>
      <head>
        <style>
          body { font-family: sans-serif; padding: 20px; max-width: 500px; margin: 0 auto; }
          .form-group { margin-bottom: 15px; }
          label { display: block; margin-bottom: 5px; font-weight: 600; }
          input, textarea { width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
          button { background: #2196F3; color: white; border: none; padding: 10px 20px; 
                   border-radius: 4px; cursor: pointer; }
          button:hover { background: #1976D2; }
        </style>
      </head>
      <body>
        <h2>Feedback: ${topic}</h2>
        <form id="feedbackForm">
          <div class="form-group">
            <label for="rating">Rating (1-5):</label>
            <input type="number" id="rating" min="1" max="5" required />
          </div>
          <div class="form-group">
            <label for="comments">Comments:</label>
            <textarea id="comments" rows="4" required></textarea>
          </div>
          <button type="submit">Submit Feedback</button>
        </form>
        <div id="status"></div>

        <script>
          document.getElementById('feedbackForm').addEventListener('submit', (e) => {
            e.preventDefault();
            const data = {
              type: 'feedback_submit',
              topic: '${topic}',
              rating: document.getElementById('rating').value,
              comments: document.getElementById('comments').value
            };
            
            // Send action to parent (MCP client)
            parent.postMessage(data, '*');
            
            document.getElementById('status').innerHTML = 
              '<p style="color: green;">✓ Feedback submitted successfully!</p>';
          });
        </script>
      </body>
    </html>
  `;

  return {
    content: [createUIResource({
      uri: `ui://feedback/${topic}`,
      content: { type: 'rawHtml', htmlString: htmlContent },
      encoding: 'text',
    })],
  };
});
```

### 3. Dynamic UI Generation Based on Data

```typescript
interface DatabaseQuery {
  query: string;
  results: Array<Record<string, any>>;
  executionTime: number;
}

server.registerTool('execute-query', {
  title: 'Execute Database Query',
  description: 'Executes a query and returns results with visualization',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'SQL query to execute' }
    },
    required: ['query']
  }
}, async (args) => {
  const query = args.query as string;
  
  // Simulate query execution
  const startTime = Date.now();
  const results = await executeQuery(query); // Your DB logic
  const executionTime = Date.now() - startTime;

  // Generate appropriate UI based on results
  if (results.length === 0) {
    return {
      content: [UIResourceFactory.createCard({
        title: 'Query Executed',
        content: `No results returned. Execution time: ${executionTime}ms`,
        variant: 'info'
      })],
    };
  }

  // For large datasets, create a paginated view
  if (results.length > 100) {
    return {
      content: [createPaginatedTableUI(results, executionTime)],
    };
  }

  // For small datasets, show simple table
  return {
    content: [UIResourceFactory.createDataTable(results)],
  };
});
```

---

## Advanced Design Patterns

### 1. Remote DOM Rendering Pattern

Use Remote DOM for consistency with host application styling:

```typescript
// This approach sends JavaScript that the client executes
// using its own component library

server.registerTool('show-remote-dom-ui', {
  title: 'Show Remote DOM UI',
  description: 'Renders using host components',
  inputSchema: { type: 'object', properties: {} }
}, async () => {
  // Remote DOM script that uses host's component library
  const remoteDomScript = `
    import { h } from '@remote-dom/core';

    export function render() {
      return h('div', { className: 'container' }, [
        h('h1', {}, 'Remote DOM Example'),
        h('p', {}, 'This uses the host application components'),
        h('button', { 
          onClick: () => emitAction({ type: 'button_click' })
        }, 'Click Me')
      ]);
    }
  `;

  return {
    content: [createUIResource({
      uri: 'ui://remote-dom-example',
      content: { 
        type: 'remoteDom', 
        script: remoteDomScript 
      },
      encoding: 'text',
      mimeType: 'application/vnd.mcp-ui.remote-dom+javascript'
    })],
  };
});
```

### 2. Lazy-Loading UI Components

Optimize performance by loading heavy UIs on demand:

```typescript
class LazyUIManager {
  private cache = new Map<string, any>();

  async getOrCreateUI(key: string, factory: () => Promise<any>) {
    if (this.cache.has(key)) {
      return this.cache.get(key);
    }

    const ui = await factory();
    this.cache.set(key, ui);
    return ui;
  }

  invalidate(key: string) {
    this.cache.delete(key);
  }

  clear() {
    this.cache.clear();
  }
}

const lazyUIManager = new LazyUIManager();

server.registerTool('show-chart', {
  title: 'Show Data Chart',
  description: 'Displays a chart (loaded lazily)',
  inputSchema: {
    type: 'object',
    properties: {
      datasetId: { type: 'string' }
    },
    required: ['datasetId']
  }
}, async (args) => {
  const datasetId = args.datasetId as string;

  const ui = await lazyUIManager.getOrCreateUI(
    `chart-${datasetId}`,
    async () => {
      // Heavy computation or data fetching
      const data = await fetchLargeDataset(datasetId);
      return createChartUI(data);
    }
  );

  return { content: [ui] };
});
```

### 3. Composite UI Pattern

Combine multiple UI resources into a cohesive interface:

```typescript
interface DashboardConfig {
  widgets: Array<{
    type: 'chart' | 'table' | 'stats' | 'alerts';
    config: any;
  }>;
  layout: 'grid' | 'stack';
}

class DashboardBuilder {
  async buildDashboard(config: DashboardConfig) {
    const widgetPromises = config.widgets.map(widget => 
      this.createWidget(widget.type, widget.config)
    );

    const widgets = await Promise.all(widgetPromises);
    
    return this.composeDashboard(widgets, config.layout);
  }

  private async createWidget(type: string, config: any) {
    switch (type) {
      case 'chart':
        return this.createChartWidget(config);
      case 'table':
        return this.createTableWidget(config);
      case 'stats':
        return this.createStatsWidget(config);
      case 'alerts':
        return this.createAlertsWidget(config);
      default:
        throw new Error(`Unknown widget type: ${type}`);
    }
  }

  private composeDashboard(widgets: string[], layout: string) {
    const layoutStyles = {
      grid: 'display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px;',
      stack: 'display: flex; flex-direction: column; gap: 20px;'
    };

    const htmlContent = `
      <!DOCTYPE html>
      <html>
        <head>
          <style>
            body { padding: 20px; background: #f5f5f5; margin: 0; }
            .dashboard { ${layoutStyles[layout]} }
            .widget { background: white; padding: 20px; border-radius: 8px; 
                      box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
          </style>
        </head>
        <body>
          <div class="dashboard">
            ${widgets.map(w => `<div class="widget">${w}</div>`).join('')}
          </div>
        </body>
      </html>
    `;

    return createUIResource({
      uri: `ui://dashboard/${Date.now()}`,
      content: { type: 'rawHtml', htmlString: htmlContent },
      encoding: 'text',
    });
  }

  private async createChartWidget(config: any): Promise<string> {
    // Implementation details...
    return `<canvas id="chart"></canvas><script>/* Chart.js code */</script>`;
  }

  private async createTableWidget(config: any): Promise<string> {
    // Implementation details...
    return `<table>/* Table HTML */</table>`;
  }

  private async createStatsWidget(config: any): Promise<string> {
    // Implementation details...
    return `<div class="stats">/* Stats HTML */</div>`;
  }

  private async createAlertsWidget(config: any): Promise<string> {
    // Implementation details...
    return `<div class="alerts">/* Alerts HTML */</div>`;
  }
}

server.registerTool('show-composite-dashboard', {
  title: 'Show Dashboard',
  description: 'Displays a composite dashboard',
  inputSchema: {
    type: 'object',
    properties: {
      config: { type: 'object' }
    },
    required: ['config']
  }
}, async (args) => {
  const config = args.config as DashboardConfig;
  const builder = new DashboardBuilder();
  const dashboard = await builder.buildDashboard(config);

  return { content: [dashboard] };
});
```

---

## Non-Obvious High-Value Patterns

### 1. Progressive Disclosure Pattern

**Use Case**: Complex workflows where showing everything at once is overwhelming.

This pattern reveals UI elements progressively based on user actions, reducing cognitive load while maintaining power-user capabilities.

```typescript
interface WorkflowStep {
  id: string;
  title: string;
  description: string;
  inputSchema: any;
  condition?: (previousAnswers: Record<string, any>) => boolean;
}

class ProgressiveWorkflowUI {
  constructor(private steps: WorkflowStep[]) {}

  generateUI(currentStep: number = 0, answers: Record<string, any> = {}) {
    const step = this.steps[currentStep];
    const isLastStep = currentStep === this.steps.length - 1;
    
    // Determine which steps should be shown based on conditions
    const visiblePreviousSteps = this.steps
      .slice(0, currentStep)
      .filter((s, i) => !s.condition || s.condition(answers));

    const htmlContent = `
      <!DOCTYPE html>
      <html>
        <head>
          <style>
            body { font-family: sans-serif; max-width: 700px; margin: 0 auto; padding: 20px; }
            .progress-bar { 
              height: 4px; 
              background: #e0e0e0; 
              border-radius: 2px; 
              overflow: hidden;
              margin-bottom: 30px;
            }
            .progress-fill { 
              height: 100%; 
              background: linear-gradient(90deg, #2196F3, #4CAF50);
              width: ${((currentStep + 1) / this.steps.length) * 100}%;
              transition: width 0.3s ease;
            }
            .previous-steps { margin-bottom: 30px; }
            .step-summary { 
              background: #f5f5f5; 
              padding: 15px; 
              margin-bottom: 10px; 
              border-radius: 4px;
              border-left: 3px solid #4CAF50;
            }
            .step-summary h4 { margin: 0 0 8px 0; color: #4CAF50; }
            .step-summary p { margin: 0; font-size: 14px; color: #666; }
            .current-step { 
              background: white; 
              padding: 25px; 
              border-radius: 8px;
              box-shadow: 0 2px 8px rgba(0,0,0,0.1);
            }
            .current-step h3 { margin-top: 0; color: #2196F3; }
            .form-group { margin-bottom: 20px; }
            label { display: block; margin-bottom: 8px; font-weight: 600; }
            input, select, textarea { 
              width: 100%; 
              padding: 10px; 
              border: 2px solid #e0e0e0; 
              border-radius: 4px;
              font-size: 14px;
            }
            input:focus, select:focus, textarea:focus {
              outline: none;
              border-color: #2196F3;
            }
            .button-group { display: flex; gap: 10px; margin-top: 20px; }
            button { 
              padding: 12px 24px; 
              border: none; 
              border-radius: 4px; 
              cursor: pointer;
              font-size: 14px;
              font-weight: 600;
            }
            .btn-primary { background: #2196F3; color: white; }
            .btn-secondary { background: #e0e0e0; color: #333; }
            .btn-primary:hover { background: #1976D2; }
            .btn-secondary:hover { background: #d0d0d0; }
          </style>
        </head>
        <body>
          <div class="progress-bar">
            <div class="progress-fill"></div>
          </div>

          ${visiblePreviousSteps.length > 0 ? `
            <div class="previous-steps">
              <h3>Completed Steps</h3>
              ${visiblePreviousSteps.map((s, i) => `
                <div class="step-summary">
                  <h4>${i + 1}. ${s.title}</h4>
                  <p><strong>Answer:</strong> ${this.formatAnswer(answers[s.id])}</p>
                </div>
              `).join('')}
            </div>
          ` : ''}

          <div class="current-step">
            <h3>Step ${currentStep + 1}: ${step.title}</h3>
            <p>${step.description}</p>
            
            <form id="stepForm">
              ${this.generateFormFields(step.inputSchema)}
              
              <div class="button-group">
                ${currentStep > 0 ? `
                  <button type="button" class="btn-secondary" onclick="goBack()">
                    ← Previous
                  </button>
                ` : ''}
                <button type="submit" class="btn-primary">
                  ${isLastStep ? 'Complete ✓' : 'Next →'}
                </button>
              </div>
            </form>
          </div>

          <script>
            const currentAnswers = ${JSON.stringify(answers)};
            const currentStep = ${currentStep};

            document.getElementById('stepForm').addEventListener('submit', (e) => {
              e.preventDefault();
              const formData = new FormData(e.target);
              const stepAnswer = {};
              for (let [key, value] of formData.entries()) {
                stepAnswer[key] = value;
              }

              parent.postMessage({
                type: 'workflow_step_complete',
                step: currentStep,
                stepId: '${step.id}',
                answer: stepAnswer,
                allAnswers: { ...currentAnswers, '${step.id}': stepAnswer },
                isLastStep: ${isLastStep}
              }, '*');
            });

            function goBack() {
              parent.postMessage({
                type: 'workflow_step_back',
                step: currentStep
              }, '*');
            }
          </script>
        </body>
      </html>
    `;

    return createUIResource({
      uri: `ui://workflow/${currentStep}`,
      content: { type: 'rawHtml', htmlString: htmlContent },
      encoding: 'text',
    });
  }

  private generateFormFields(schema: any): string {
    // Generate form fields based on JSON schema
    const fields = Object.entries(schema.properties || {}).map(([key, prop]: [string, any]) => {
      return `
        <div class="form-group">
          <label for="${key}">${prop.description || key}:</label>
          ${this.getInputElement(key, prop)}
        </div>
      `;
    });
    return fields.join('');
  }

  private getInputElement(name: string, prop: any): string {
    if (prop.enum) {
      return `
        <select name="${name}" required>
          <option value="">Select...</option>
          ${prop.enum.map((v: string) => `<option value="${v}">${v}</option>`).join('')}
        </select>
      `;
    }
    
    switch (prop.type) {
      case 'string':
        return `<input type="text" name="${name}" required />`;
      case 'number':
        return `<input type="number" name="${name}" required />`;
      case 'boolean':
        return `<input type="checkbox" name="${name}" />`;
      default:
        return `<input type="text" name="${name}" />`;
    }
  }

  private formatAnswer(answer: any): string {
    if (typeof answer === 'object') {
      return Object.entries(answer).map(([k, v]) => `${k}: ${v}`).join(', ');
    }
    return String(answer);
  }
}

// Usage in MCP server
server.registerTool('start-deployment-workflow', {
  title: 'Start Deployment Workflow',
  description: 'Initiates a progressive deployment configuration workflow',
  inputSchema: {
    type: 'object',
    properties: {
      currentStep: { type: 'number', default: 0 },
      answers: { type: 'object', default: {} }
    }
  }
}, async (args) => {
  const workflow = new ProgressiveWorkflowUI([
    {
      id: 'environment',
      title: 'Select Environment',
      description: 'Choose the deployment environment',
      inputSchema: {
        properties: {
          environment: {
            type: 'string',
            enum: ['development', 'staging', 'production'],
            description: 'Target environment'
          }
        }
      }
    },
    {
      id: 'strategy',
      title: 'Deployment Strategy',
      description: 'Configure how the deployment should proceed',
      inputSchema: {
        properties: {
          strategy: {
            type: 'string',
            enum: ['rolling', 'blue-green', 'canary'],
            description: 'Deployment strategy'
          }
        }
      }
    },
    {
      id: 'canary-config',
      title: 'Canary Configuration',
      description: 'Configure canary deployment parameters',
      inputSchema: {
        properties: {
          percentage: {
            type: 'number',
            description: 'Initial traffic percentage'
          }
        }
      },
      condition: (answers) => answers.strategy?.strategy === 'canary'
    },
    {
      id: 'confirmation',
      title: 'Review & Confirm',
      description: 'Review your deployment configuration',
      inputSchema: {
        properties: {
          confirmed: {
            type: 'boolean',
            description: 'I have reviewed the configuration'
          }
        }
      }
    }
  ]);

  const currentStep = (args.currentStep as number) || 0;
  const answers = (args.answers as Record<string, any>) || {};

  return {
    content: [workflow.generateUI(currentStep, answers)],
  };
});
```

### 2. Context-Aware Adaptive UI Pattern

**Use Case**: UIs that adapt based on user expertise, preferences, or environmental context.

This pattern creates UIs that intelligently adjust their complexity and features based on contextual signals.

```typescript
interface UserContext {
  expertiseLevel: 'beginner' | 'intermediate' | 'expert';
  preferences: {
    verbosity: 'minimal' | 'normal' | 'detailed';
    theme: 'light' | 'dark' | 'auto';
  };
  recentActions: string[];
  environmentalContext?: {
    screenSize: 'mobile' | 'tablet' | 'desktop';
    bandwidth: 'low' | 'medium' | 'high';
  };
}

class AdaptiveUIGenerator {
  generateAdaptiveUI(
    data: any,
    context: UserContext,
    purpose: string
  ) {
    const complexity = this.determineComplexity(context);
    const features = this.selectFeatures(context, purpose);
    const styling = this.getAdaptiveStyling(context);

    return this.composeAdaptiveUI(data, complexity, features, styling);
  }

  private determineComplexity(context: UserContext): 'simple' | 'moderate' | 'advanced' {
    // Use multiple signals to determine appropriate complexity
    if (context.expertiseLevel === 'beginner') return 'simple';
    if (context.expertiseLevel === 'expert') return 'advanced';
    
    // Intermediate users get complexity based on usage patterns
    const recentComplexActions = context.recentActions.filter(
      action => ['advanced_filter', 'bulk_edit', 'custom_query'].includes(action)
    );
    
    return recentComplexActions.length > 3 ? 'advanced' : 'moderate';
  }

  private selectFeatures(context: UserContext, purpose: string): string[] {
    const baseFeatures = ['display', 'basic_controls'];
    
    const featureMap: Record<string, string[]> = {
      beginner: ['tooltips', 'guided_actions', 'help_links'],
      intermediate: ['shortcuts', 'bulk_actions', 'filters'],
      expert: ['advanced_filters', 'bulk_edit', 'custom_scripts', 'api_access']
    };

    // Adapt to bandwidth - reduce heavy features on low bandwidth
    if (context.environmentalContext?.bandwidth === 'low') {
      return [...baseFeatures, ...featureMap[context.expertiseLevel].filter(
        f => !['custom_scripts', 'api_access'].includes(f)
      )];
    }

    return [...baseFeatures, ...featureMap[context.expertiseLevel]];
  }

  private getAdaptiveStyling(context: UserContext) {
    const theme = context.preferences.theme === 'auto' 
      ? this.detectSystemTheme() 
      : context.preferences.theme;

    const themes = {
      light: {
        bg: '#ffffff',
        text: '#333333',
        primary: '#2196F3',
        secondary: '#f5f5f5',
        border: '#e0e0e0'
      },
      dark: {
        bg: '#1e1e1e',
        text: '#e0e0e0',
        primary: '#64B5F6',
        secondary: '#2d2d2d',
        border: '#404040'
      }
    };

    // Adjust font sizes and spacing based on expertise
    const spacingMultiplier = context.expertiseLevel === 'expert' ? 0.8 : 1;
    
    return {
      colors: themes[theme],
      spacing: {
        base: 16 * spacingMultiplier,
        compact: 8 * spacingMultiplier
      }
    };
  }

  private detectSystemTheme(): 'light' | 'dark' {
    // In real implementation, this would detect via media query
    return 'light';
  }

  private composeAdaptiveUI(
    data: any,
    complexity: string,
    features: string[],
    styling: any
  ) {
    const { colors, spacing } = styling;

    // Generate different UI structures based on complexity
    const uiStructures = {
      simple: this.generateSimpleUI(data, features, styling),
      moderate: this.generateModerateUI(data, features, styling),
      advanced: this.generateAdvancedUI(data, features, styling)
    };

    const htmlContent = `
      <!DOCTYPE html>
      <html>
        <head>
          <meta name="viewport" content="width=device-width, initial-scale=1.0">
          <style>
            :root {
              --bg-color: ${colors.bg};
              --text-color: ${colors.text};
              --primary-color: ${colors.primary};
              --secondary-color: ${colors.secondary};
              --border-color: ${colors.border};
              --spacing-base: ${spacing.base}px;
              --spacing-compact: ${spacing.compact}px;
            }
            
            body {
              font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
              background: var(--bg-color);
              color: var(--text-color);
              margin: 0;
              padding: var(--spacing-base);
              transition: background 0.3s, color 0.3s;
            }

            .adaptive-container {
              max-width: ${complexity === 'advanced' ? '1400px' : '900px'};
              margin: 0 auto;
            }

            ${this.generateFeatureStyles(features)}
          </style>
        </head>
        <body>
          <div class="adaptive-container">
            ${uiStructures[complexity]}
          </div>
          
          <script>
            ${this.generateAdaptiveScripts(features)}
          </script>
        </body>
      </html>
    `;

    return createUIResource({
      uri: `ui://adaptive/${Date.now()}`,
      content: { type: 'rawHtml', htmlString: htmlContent },
      encoding: 'text',
    });
  }

  private generateSimpleUI(data: any, features: string[], styling: any): string {
    return `
      <div class="simple-view">
        <h2>Data Overview</h2>
        ${features.includes('tooltips') ? 
          '<p class="help-text">💡 Tip: Click on items to see details</p>' : ''}
        <div class="data-cards">
          ${this.renderAsCards(data)}
        </div>
      </div>
    `;
  }

  private generateModerateUI(data: any, features: string[], styling: any): string {
    return `
      <div class="moderate-view">
        <div class="toolbar">
          <h2>Data Management</h2>
          ${features.includes('filters') ? this.renderFilterControls() : ''}
          ${features.includes('bulk_actions') ? this.renderBulkActions() : ''}
        </div>
        <div class="data-table">
          ${this.renderAsTable(data, features.includes('shortcuts'))}
        </div>
      </div>
    `;
  }

  private generateAdvancedUI(data: any, features: string[], styling: any): string {
    return `
      <div class="advanced-view">
        <div class="sidebar">
          ${features.includes('advanced_filters') ? this.renderAdvancedFilters() : ''}
          ${features.includes('custom_scripts') ? this.renderScriptEditor() : ''}
        </div>
        <div class="main-content">
          <div class="command-bar">
            ${features.includes('api_access') ? this.renderAPIConsole() : ''}
          </div>
          <div class="data-grid">
            ${this.renderAsAdvancedGrid(data)}
          </div>
        </div>
      </div>
    `;
  }

  private generateFeatureStyles(features: string[]): string {
    let styles = '';
    
    if (features.includes('tooltips')) {
      styles += `
        .tooltip { position: relative; cursor: help; }
        .tooltip:hover::after { content: attr(data-tip); /* tooltip styles */ }
      `;
    }
    
    if (features.includes('advanced_filters')) {
      styles += `
        .sidebar { width: 250px; padding: var(--spacing-base); 
                   background: var(--secondary-color); }
        .advanced-view { display: flex; gap: var(--spacing-base); }
      `;
    }

    return styles;
  }

  private generateAdaptiveScripts(features: string[]): string {
    let scripts = `
      // Base tracking
      function trackAction(action) {
        parent.postMessage({ type: 'track_action', action }, '*');
      }
    `;

    if (features.includes('shortcuts')) {
      scripts += `
        // Keyboard shortcuts for power users
        document.addEventListener('keydown', (e) => {
          if (e.ctrlKey || e.metaKey) {
            switch(e.key) {
              case 'f': e.preventDefault(); /* focus filter */; break;
              case 'a': e.preventDefault(); /* select all */; break;
            }
          }
        });
      `;
    }

    return scripts;
  }

  // Helper rendering methods
  private renderAsCards(data: any): string {
    // Implementation...
    return '<div>Card view</div>';
  }

  private renderAsTable(data: any, withShortcuts: boolean): string {
    // Implementation...
    return '<table>Table view</table>';
  }

  private renderAsAdvancedGrid(data: any): string {
    // Implementation...
    return '<div>Advanced grid</div>';
  }

  private renderFilterControls(): string {
    return '<div class="filters">Basic filters</div>';
  }

  private renderBulkActions(): string {
    return '<div class="bulk-actions">Bulk action buttons</div>';
  }

  private renderAdvancedFilters(): string {
    return '<div class="advanced-filters">Advanced filter panel</div>';
  }

  private renderScriptEditor(): string {
    return '<div class="script-editor">Script editor panel</div>';
  }

  private renderAPIConsole(): string {
    return '<div class="api-console">API console</div>';
  }
}

// Usage in MCP server
server.registerTool('show-adaptive-data-view', {
  title: 'Show Adaptive Data View',
  description: 'Displays data with UI adapted to user context',
  inputSchema: {
    type: 'object',
    properties: {
      data: { type: 'object' },
      userContext: { type: 'object' },
      purpose: { type: 'string' }
    },
    required: ['data', 'userContext']
  }
}, async (args) => {
  const generator = new AdaptiveUIGenerator();
  const ui = generator.generateAdaptiveUI(
    args.data,
    args.userContext as UserContext,
    (args.purpose as string) || 'general'
  );

  return { content: [ui] };
});
```

### 3. Real-Time Collaborative UI Pattern

**Use Case**: Multiple users working on shared data or coordinated workflows.

This pattern enables real-time collaborative interfaces where multiple users can see live updates and coordinate actions.

```typescript
interface CollaborativeSession {
  sessionId: string;
  participants: Array<{
    id: string;
    name: string;
    color: string;
    cursor?: { x: number; y: number };
  }>;
  sharedState: any;
  operations: Array<{ userId: string; timestamp: number; operation: any }>;
}

class CollaborativeUIManager {
  private sessions = new Map<string, CollaborativeSession>();
  private wsConnections = new Map<string, WebSocket>();

  createCollaborativeUI(sessionId: string, initialData: any) {
    const session = this.sessions.get(sessionId) || this.createSession(sessionId, initialData);

    const htmlContent = `
      <!DOCTYPE html>
      <html>
        <head>
          <style>
            body { 
              font-family: sans-serif; 
              margin: 0; 
              padding: 20px; 
              background: #f5f5f5;
            }
            
            .collab-header {
              background: white;
              padding: 15px;
              border-radius: 8px;
              margin-bottom: 20px;
              display: flex;
              justify-content: space-between;
              align-items: center;
              box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            }

            .participants {
              display: flex;
              gap: 10px;
            }

            .participant {
              width: 36px;
              height: 36px;
              border-radius: 50%;
              display: flex;
              align-items: center;
              justify-content: center;
              color: white;
              font-weight: 600;
              font-size: 14px;
              position: relative;
            }

            .participant.active::after {
              content: '';
              position: absolute;
              bottom: 0;
              right: 0;
              width: 12px;
              height: 12px;
              background: #4CAF50;
              border: 2px solid white;
              border-radius: 50%;
            }

            .cursor {
              position: absolute;
              width: 20px;
              height: 20px;
              pointer-events: none;
              transition: transform 0.1s ease;
              z-index: 1000;
            }

            .cursor svg {
              filter: drop-shadow(0 2px 4px rgba(0,0,0,0.3));
            }

            .cursor-label {
              position: absolute;
              left: 20px;
              top: 0;
              padding: 2px 8px;
              border-radius: 3px;
              font-size: 12px;
              white-space: nowrap;
              color: white;
            }

            .workspace {
              background: white;
              border-radius: 8px;
              padding: 20px;
              min-height: 500px;
              position: relative;
              box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            }

            .operation-indicator {
              position: absolute;
              top: 10px;
              right: 10px;
              padding: 5px 10px;
              background: #2196F3;
              color: white;
              border-radius: 4px;
              font-size: 12px;
              opacity: 0;
              transition: opacity 0.3s;
            }

            .operation-indicator.active {
              opacity: 1;
            }

            .editable {
              outline: none;
              padding: 10px;
              border: 2px solid transparent;
              border-radius: 4px;
              transition: border-color 0.2s;
            }

            .editable:focus {
              border-color: #2196F3;
            }

            .editable.being-edited {
              border-color: #FF9800;
              background: #FFF3E0;
            }

            .conflict-indicator {
              position: absolute;
              background: #F44336;
              color: white;
              padding: 10px;
              border-radius: 4px;
              box-shadow: 0 2px 8px rgba(0,0,0,0.2);
            }
          </style>
        </head>
        <body>
          <div class="collab-header">
            <div>
              <h3 style="margin: 0;">Collaborative Session</h3>
              <p style="margin: 5px 0 0 0; font-size: 14px; color: #666;">
                Session: ${sessionId}
              </p>
            </div>
            <div class="participants" id="participants">
              <!-- Populated dynamically -->
            </div>
          </div>

          <div class="workspace" id="workspace">
            <div class="operation-indicator" id="operationIndicator">
              Synchronizing...
            </div>
            
            <!-- Collaborative content area -->
            <div id="content" contenteditable="true" class="editable">
              ${JSON.stringify(initialData, null, 2)}
            </div>
          </div>

          <!-- Cursors container -->
          <div id="cursors"></div>

          <script>
            const sessionId = '${sessionId}';
            let ws = null;
            let myUserId = null;
            let participants = new Map();

            // Initialize WebSocket connection
            function connectWebSocket() {
              // In real implementation, connect to actual WebSocket server
              // ws = new WebSocket('ws://your-server.com/collab/' + sessionId);
              
              // Simulated connection for this example
              console.log('Connecting to collaborative session:', sessionId);
              
              // Send cursor position updates
              document.addEventListener('mousemove', (e) => {
                const x = e.clientX / window.innerWidth;
                const y = e.clientY / window.innerHeight;
                sendCursorUpdate(x, y);
              });

              // Track content changes
              const contentEl = document.getElementById('content');
              let changeTimer = null;
              contentEl.addEventListener('input', (e) => {
                clearTimeout(changeTimer);
                changeTimer = setTimeout(() => {
                  sendContentUpdate(contentEl.textContent);
                }, 300); // Debounce
              });
            }

            function sendCursorUpdate(x, y) {
              const message = {
                type: 'cursor_move',
                sessionId,
                userId: myUserId,
                position: { x, y }
              };
              parent.postMessage(message, '*');
            }

            function sendContentUpdate(content) {
              showOperationIndicator();
              const message = {
                type: 'content_update',
                sessionId,
                userId: myUserId,
                content,
                timestamp: Date.now()
              };
              parent.postMessage(message, '*');
            }

            function updateParticipants(participantsList) {
              participants = new Map(participantsList.map(p => [p.id, p]));
              renderParticipants();
              renderCursors();
            }

            function renderParticipants() {
              const container = document.getElementById('participants');
              container.innerHTML = Array.from(participants.values())
                .map(p => \`
                  <div class="participant active" 
                       style="background: \${p.color};"
                       title="\${p.name}">
                    \${p.name.substring(0, 2).toUpperCase()}
                  </div>
                \`).join('');
            }

            function renderCursors() {
              const container = document.getElementById('cursors');
              container.innerHTML = Array.from(participants.values())
                .filter(p => p.id !== myUserId && p.cursor)
                .map(p => \`
                  <div class="cursor" 
                       style="transform: translate(\${p.cursor.x * window.innerWidth}px, 
                                                    \${p.cursor.y * window.innerHeight}px);">
                    <svg width="20" height="20" viewBox="0 0 20 20">
                      <path d="M0 0 L0 16 L6 12 L10 20 L12 19 L8 11 L16 10 Z" 
                            fill="\${p.color}" />
                    </svg>
                    <div class="cursor-label" style="background: \${p.color};">
                      \${p.name}
                    </div>
                  </div>
                \`).join('');
            }

            function showOperationIndicator() {
              const indicator = document.getElementById('operationIndicator');
              indicator.classList.add('active');
              setTimeout(() => {
                indicator.classList.remove('active');
              }, 1000);
            }

            // Handle incoming messages from server
            window.addEventListener('message', (event) => {
              const msg = event.data;
              
              switch(msg.type) {
                case 'participant_joined':
                case 'participant_left':
                  updateParticipants(msg.participants);
                  break;
                
                case 'cursor_update':
                  if (msg.userId !== myUserId) {
                    const participant = participants.get(msg.userId);
                    if (participant) {
                      participant.cursor = msg.position;
                      renderCursors();
                    }
                  }
                  break;
                
                case 'content_update':
                  if (msg.userId !== myUserId) {
                    showOperationIndicator();
                    // Apply operational transform to merge changes
                    applyRemoteUpdate(msg.content, msg.operations);
                  }
                  break;
              }
            });

            function applyRemoteUpdate(content, operations) {
              const contentEl = document.getElementById('content');
              const currentPos = window.getSelection().getRangeAt(0).startOffset;
              
              // Simple conflict resolution: last-write-wins with merge
              // In production, use Operational Transform or CRDT
              contentEl.textContent = content;
              
              // Restore cursor position (simplified)
              const range = document.createRange();
              const sel = window.getSelection();
              try {
                range.setStart(contentEl.firstChild, Math.min(currentPos, content.length));
                range.collapse(true);
                sel.removeAllRanges();
                sel.addRange(range);
              } catch (e) {
                // Position out of range
              }
            }

            // Initialize
            connectWebSocket();
            myUserId = 'user-' + Math.random().toString(36).substr(2, 9);
          </script>
        </body>
      </html>
    `;

    return createUIResource({
      uri: `ui://collab/${sessionId}`,
      content: { type: 'rawHtml', htmlString: htmlContent },
      encoding: 'text',
    });
  }

  private createSession(sessionId: string, initialData: any): CollaborativeSession {
    const session: CollaborativeSession = {
      sessionId,
      participants: [],
      sharedState: initialData,
      operations: []
    };
    this.sessions.set(sessionId, session);
    return session;
  }

  addParticipant(sessionId: string, user: { id: string; name: string }) {
    const session = this.sessions.get(sessionId);
    if (!session) return;

    const colors = ['#2196F3', '#4CAF50', '#FF9800', '#9C27B0', '#F44336'];
    const color = colors[session.participants.length % colors.length];

    session.participants.push({
      ...user,
      color
    });
  }

  handleOperation(sessionId: string, userId: string, operation: any) {
    const session = this.sessions.get(sessionId);
    if (!session) return;

    session.operations.push({
      userId,
      timestamp: Date.now(),
      operation
    });

    // Apply operational transform logic here
    this.applyOperation(session, operation);
    
    // Broadcast to all participants
    this.broadcastToSession(sessionId, {
      type: 'operation_applied',
      operation,
      userId
    });
  }

  private applyOperation(session: CollaborativeSession, operation: any) {
    // Implement Operational Transform or CRDT logic
    // This is a simplified version
    switch (operation.type) {
      case 'insert':
        // Apply insert operation
        break;
      case 'delete':
        // Apply delete operation
        break;
      case 'update':
        // Apply update operation
        session.sharedState = { ...session.sharedState, ...operation.data };
        break;
    }
  }

  private broadcastToSession(sessionId: string, message: any) {
    // In real implementation, broadcast via WebSocket to all connected clients
    console.log(`Broadcasting to session ${sessionId}:`, message);
  }
}

// Usage in MCP server
const collabManager = new CollaborativeUIManager();

server.registerTool('start-collaborative-session', {
  title: 'Start Collaborative Session',
  description: 'Creates a real-time collaborative UI session',
  inputSchema: {
    type: 'object',
    properties: {
      sessionId: { type: 'string' },
      initialData: { type: 'object' },
      userId: { type: 'string' },
      userName: { type: 'string' }
    },
    required: ['sessionId', 'initialData', 'userId', 'userName']
  }
}, async (args) => {
  const sessionId = args.sessionId as string;
  const initialData = args.initialData;
  
  collabManager.addParticipant(sessionId, {
    id: args.userId as string,
    name: args.userName as string
  });

  const ui = collabManager.createCollaborativeUI(sessionId, initialData);

  return { content: [ui] };
});
```

---

## Best Practices & Tips

1. **Performance**: Cache UI resources when possible, especially for data that doesn't change frequently

2. **Security**: Always sanitize user input before including it in HTML. Use proper escaping:
   ```typescript
   function escapeHtml(unsafe: string): string {
     return unsafe
       .replace(/&/g, "&amp;")
       .replace(/</g, "&lt;")
       .replace(/>/g, "&gt;")
       .replace(/"/g, "&quot;")
       .replace(/'/g, "&#039;");
   }
   ```

3. **Accessibility**: Include ARIA labels and keyboard navigation in your UIs

4. **Error Handling**: Wrap UI generation in try-catch blocks and provide fallback UIs

5. **Testing**: Create test utilities to validate UI resources:
   ```typescript
   function validateUIResource(resource: any): boolean {
     return resource.uri && 
            resource.content && 
            resource.encoding;
   }
   ```

6. **Documentation**: Document the actions your UIs can emit so clients know what to handle

---

## Conclusion

MCP-UI enables powerful, interactive interfaces within the Model Context Protocol. Starting with simple HTML resources, you can progressively adopt more sophisticated patterns like progressive disclosure, context-aware adaptation, and real-time collaboration. The key is to match the pattern to your specific use case while maintaining simplicity where possible.

The non-obvious patterns (progressive workflows, adaptive UIs, and collaborative interfaces) shine in specific scenarios:
- **Progressive Disclosure**: Complex multi-step processes (deployments, configurations, onboarding)
- **Context-Aware UIs**: Applications with diverse user expertise levels or varying usage contexts
- **Collaborative UIs**: Team workflows, shared document editing, real-time coordination

Choose patterns that add genuine value to your users' experience while keeping implementation complexity manageable.