# React Usage & Examples

Here's how to use the `<UIResourceRenderer />` component from `@mcp-ui/client` in a React environment.

## Installation

Make sure you have `@mcp-ui/client` and its peer dependencies installed in your project:

```bash
npm i @mcp-ui/client
```

## Rendering Remote DOM Resources

This example shows how to render a `remoteDom` resource. This requires a `remoteElements` and `componentLibrary` (minimal default provided)  to be passed to the `UIResourceRenderer`.

```tsx
import React, { useState } from 'react';
import { 
  UIResourceRenderer, 
  UIActionResult,
  basicComponentLibrary,
  remoteTextDefinition,
  remoteButtonDefinition
} from '@mcp-ui/client';

const remoteDomScript = `
  const button = document.createElement('ui-button');
  button.setAttribute('label', 'Click me for a tool call!');
  button.addEventListener('press', () => {
    window.parent.postMessage({ type: 'tool', payload: { toolName: 'uiInteraction', params: { action: 'button-click', from: 'remote-dom' } } }, '*');
  });
  root.appendChild(button);
`;

// This mocks the resource as received from the server SDK
const remoteDomResource = {
  type: 'resource',
  resource: {
    uri: 'ui://remote-component/action-button',
    mimeType: 'application/vnd.mcp-ui.remote-dom+javascript; framework=react',
    text: remoteDomScript,
  },
};

const AppWithRemoteDOM: React.FC = () => {
  const [lastAction, setLastAction] = useState<any>(null);

  const handleGenericMcpAction = async (result: UIActionResult) => {
    if (result.type === 'tool') {
      setLastAction({ tool: result.payload.toolName, params: result.payload.params });
    }
    return { status: 'Action handled' };
  };

  return (
    <div>
      <UIResourceRenderer
        resource={remoteDomResource.resource}
        onUIAction={handleGenericMcpAction}
        remoteDomProps={{
          library: basicComponentLibrary,
          remoteElements: [remoteButtonDefinition, remoteTextDefinition],
        }}
      />
      {lastAction && (
        <div style={{ marginTop: 20, border: '1px solid green', padding: 10 }}>
          <h3>Last Action Received by Host:</h3>
          <pre>{JSON.stringify(lastAction, null, 2)}</pre>
        </div>
      )}
    </div>
  );
};
```

## Rendering HTML Resources

```tsx
import React, { useState } from 'react';
import { 
  UIResourceRenderer, 
  UIActionResult,
  basicComponentLibrary,
  remoteTextDefinition,
  remoteButtonDefinition
} from '@mcp-ui/client';

// Simulate fetching an MCP UI resource
const fetchMcpResource = async (id: string): Promise<any> => {
  if (id === 'raw') {
    return {
      type: 'resource',
      resource: {
        uri: 'ui://example/raw-html',
        mimeType: 'text/html',
        text: "<h1>raw HTML via Text</h1><p>Content loaded rawly.</p><button onclick=\"window.parent.postMessage({ type: 'tool', payload: { toolName: 'uiInteraction', params: { action: 'rawClick', value: Date.now() } } }, '*')\">Click Me (raw)</button>",
      },
    };
  } else if (id === 'blob') {
    const html =
      "<h1>HTML from Blob</h1><p>Content was Base64 encoded.</p><button onclick=\"window.parent.postMessage({ type: 'tool', payload: { toolName: 'uiInteraction', params: { action: 'blobClick', value: 'test' } } }, '*')\">Click Me (Blob)</button>";
    return {
      type: 'resource',
      resource: {
        uri: 'ui://example/blob-html',
        mimeType: 'text/html',
        blob: btoa(html),
      },
    };
  } else if (id === 'external') {
    return {
      type: 'resource',
      resource: {
        uri: 'ui://example/external-site',
        mimeType: 'text/uri-list',
        text: 'https://vitepress.dev',
      },
    };
  }
  if (id === 'remote') {
    const remoteDomScript = `
      const button = document.createElement('ui-button');
      button.setAttribute('label', 'Click me for a tool call!');
      button.addEventListener('press', () => {
        window.parent.postMessage({ type: 'tool', payload: { toolName: 'uiInteraction', params: { action: 'button-click', from: 'remote-dom' } } }, '*');
      });
      root.appendChild(button);
    `;
    return {
      type: 'resource',
      resource: {
        uri: 'ui://remote-component/action-button',
        mimeType: 'application/vnd.mcp-ui.remote-dom+javascript; framework=react',
        text: remoteDomScript,
      },
    };
  }
  throw new Error('Unknown resource ID');
};

const App: React.FC = () => {
  const [uiResource, setUIResource] = useState<UIResource | null>(
    null,
  );
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [lastAction, setLastAction] = useState<any>(null);

  const loadResource = async (id: string) => {
    setLoading(true);
    setError(null);
    setUIResource(null);
    try {
      const block = await fetchMcpResource(id);
      setUIResource(block);
    } catch (e: any) {
      setError(e.message);
    }
    setLoading(false);
  };

  const handleGenericMcpAction = async (result: UIActionResult) => {
    if (result.type === 'tool') {
      console.log(`Action received in host app - Tool: ${result.payload.toolName}, Params:`, result.payload.params);
      setLastAction({ tool: result.payload.toolName, params: result.payload.params });
    } else if (result.type === 'prompt') {
      console.log(`Prompt received in host app:`, result.payload.prompt);
      setLastAction({ prompt: result.payload.prompt });
    } else if (result.type === 'link') {
      console.log(`Link received in host app:`, result.payload.url);
      setLastAction({ url: result.payload.url });
    } else if (result.type === 'intent') {
      console.log(`Intent received in host app:`, result.payload.intent);
      setLastAction({ intent: result.payload.intent });
    } else if (result.type === 'notify') {
      console.log(`Notification received in host app:`, result.payload.message);
      setLastAction({ message: result.payload.message });
    }
    return {
      status: 'Action handled by host application',
    };
  };

  return (
    <div>
      <h1>MCP-UI Client Demo</h1>
      <button onClick={() => loadResource('raw')}>
        Load raw HTML (Text)
      </button>
      <button onClick={() => loadResource('blob')}>
        Load raw HTML (Blob)
      </button>
      <button onClick={() => loadResource('external')}>
        Load External App (URL)
      </button>
      <button onClick={() => loadResource('remote')}>
        Load Remote DOM
      </button>

      {loading && <p>Loading resource...</p>}
      {error && <p style={{ color: 'red' }}>Error: {error}</p>}

      {uiResource && uiResource.resource && (
        <div style={{ marginTop: 20, border: '2px solid blue', padding: 10 }}>
          <h2>Rendering Resource: {uiResource.resource.uri}</h2>
          <UIResourceRenderer
            resource={uiResource.resource}
            onUIAction={handleGenericMcpAction}
            remoteDomProps={{
              library: basicComponentLibrary,
              remoteElements: [remoteButtonDefinition, remoteTextDefinition],
            }}
          />
        </div>
      )}

      {lastAction && (
        <div style={{ marginTop: 20, border: '1px solid green', padding: 10 }}>
          <h3>Last Action Received by Host:</h3>
          <pre>{JSON.stringify(lastAction, null, 2)}</pre>
        </div>
      )}
    </div>
  );
};

export default App;
```

---

## Handling Asynchronous Actions with Message IDs

When an action from the iframe requires asynchronous processing on the host, the `messageId` property can be used to track the action's lifecycle and result. This allows the iframe to fetch data from the host, present feedback to the user (e.g., loading indicators), success messages, etc.

### Communication Flow

1.  **Iframe to Host**: The iframe sends a message with a unique `messageId`.
2.  **Host Acknowledgment**: The `UIResourceRenderer` automatically sends a `ui-message-received` message back to the iframe to acknowledge receipt.
3.  **Host Processing**: The `onUIAction` function provided to the renderer is executed. This function can be `async` and performs the long-running task.
4.  **Host Response**:
    *   If the `onUIAction` promise resolves, `UIResourceRenderer` sends a `ui-message-response` with the resolved value as the `response`.
    *   If the `onUIAction` promise rejects (throws an error), it sends a `ui-message-response` with the error details as the `error`.
5.  **Iframe Updates**: The iframe listens for these messages and updates its UI accordingly.

### Example


#### 1. In the iframe: Initiating the request

First, the iframe needs to generate a unique ID for the request and send it to the host. It should also keep track of pending requests to handle responses when they arrive.

```typescript
// A Map to store callbacks for pending requests, keyed by messageId.
const requests = new Map<string, (response: any, error?: any) => void>();

function makeRequest() {
  const messageId = crypto.randomUUID();

  // Store a callback to handle the response for this specific request.
  requests.set(messageId, (response, error) => {
    if (error) {
      console.error('Request failed:', error);
      // Update UI to show error
    } else {
      console.log('Received response:', response);
      // Update UI with the response data
    }
  });

  // Send the request to the host.
  window.parent.postMessage(
    {
      type: "ui-request-data",
      messageId,
      payload: {
        requestType: "get-payment-methods",
        params: {
          // any params needed for the request
        },
      },
    },
    "*"
  );

  // You can update the UI to a loading state here.
}

// Example: trigger the request on a button click.
button.addEventListener("click", makeRequest);
```

#### 2. In the host: Receiving and processing the request

The host listens for messages from the iframe. When it receives a request with a `messageId`, it can optionally send an acknowledgment and then starts the asynchronous work.

```typescript
window.addEventListener("message", async (event) => {
  const { type, messageId, payload } = event.data;

  // Acknowledge receipt of the message
  if (messageId && event.source) {
    (event.source as Window).postMessage(
      {
        type: "ui-message-received",
        messageId: messageId,
      },
      { targetOrigin: "*" }
    );
  }

  if (type === "ui-request-data") {
    const { requestType, params } = payload;
    if (requestType === "get-payment-methods") {
      try {
        // 3. Perform the async operation
        const paymentMethods = await fetchPaymentMethods(params);

        // 4. Send a success response
        if (event.source) {
          (event.source as Window).postMessage(
            {
              type: "ui-message-response",
              messageId: messageId,
              payload: { response: { paymentMethods } },
            },
            { targetOrigin: "*" }
          );
        }
      } catch (error) {
        // 4. Send an error response
        if (event.source) {
          (event.source as Window).postMessage(
            {
              type: "ui-message-response",
              messageId: messageId,
              payload: { error },
            },
            { targetOrigin: "*" }
          );
        }
      }
    }
  }
});
```

#### 3. In the iframe: Handling the response

The iframe needs a listener to handle messages from the host. It can use the `messageId` to match responses to the original requests.

```typescript
window.addEventListener("message", (event) => {
  const { type, messageId, payload } = event.data;

  // Check if it's a response to a request we're waiting for.
  if (!messageId || !requests.has(messageId)) {
    return;
  }

  if (type === "ui-message-received") {
    // The host has acknowledged the request. You can update the UI.
    console.log(`Request ${messageId} is being processed...`);
    // e.g., show a more specific loading indicator.
  }

  if (type === "ui-message-response") {
    const { response, error } = payload;
    
    // Retrieve the original callback.
    const callback = requests.get(messageId);
    if (callback) {
      callback(response, error);
    }
    
    // Clean up the request from the map.
    requests.delete(messageId);
  }
});
```


This pattern is crucial for building responsive and user-friendly UIs, especially when interacting with potentially slow backend operations.

---

That's it! Just use `<UIResourceRenderer />` with the right props and you're ready to render interactive HTML from MCP resources in your React app. The `UIResourceRenderer` automatically detects the resource type and renders the appropriate component internally. If you need more details, check out the [UIResourceRenderer Component](./resource-renderer.md) page.
