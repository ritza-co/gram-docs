---
title: Deploying an xmcp server to Gram using functions
description: Deploying an xmcp server to Gram using functions
sidebar:
  order: 1
---

MCP servers let AI systems connect to external tools and data sources, but building them requires handling low-level protocol details. Deploying them adds another layer of complexity: infrastructure management, authentication, monitoring, and scaling.

This guide shows you how to build and deploy an MCP server using xmcp (a TypeScript framework that handles the protocol) and Gram Functions (a serverless platform that handles deployment). You'll create a server with an email-sending tool and email list resource, then deploy it to production.

## What is xmcp?

[xmcp](https://xmcp.dev/) is a TypeScript framework from basement.studio for building MCP servers. It removes protocol boilerplate so you focus on building functionality.

Key features:

- **File-System Based Architecture with auto-discovery**: Place tools in `tools/`, resources in `resources/`, and xmcp auto-discovers and registers them
- **Hot Reloading**: Changes appear immediately during development
- **Type Safety**: Full TypeScript support with type inference from Zod schemas
- **Flexible Deployment**: Built-in adapters for Next.js, Express, Vercel, and serverless platforms

xmcp provides abstractions for tools, resources, and prompts, provides a middleware, and auto-discovery.

### 1. Tools

[Tools](https://xmcp.dev/docs/core-concepts/tools) are functions that AI agents can call to perform actions. In xmcp, defining a tool is straightforward:

```typescript
import { z } from "zod";
import { type InferSchema } from "xmcp";

export const schema = {
  name: z.string().describe("The name of the person to greet"),
};

export const metadata = {
  name: "greet",
  description: "Greet someone special",
};

export default async function greet({ name }: InferSchema<typeof schema>) {
  return `Hello, ${name}!`;
}
```

Each tool consists of:

- A **schema** defining input parameters using Zod validation
- **metadata** with the tool's name and description
- An **async handler function** that implements the tool's logic

The `InferSchema` utility extracts TypeScript types from the Zod schema, giving you full type safety without writing types twice.

### 2. Resources

[Resources](https://xmcp.dev/docs/core-concepts/resources) provide read-only data to AI agents. They can represent files, API responses, database records, or any contextual information:

```typescript
import { type ResourceMetadata } from "xmcp";

export const metadata: ResourceMetadata = {
  name: "a-cool-photo",
  title: "A Cool Photo",
  description: "This photo is really something",
  mimeType: "image/jpg",
};

export default async function photoResource(uri: URL) {
  const res = await fetch("https://picsum.photos/200/300.jpg");
  const arrayBuffer = await res.arrayBuffer();
  const buffer = Buffer.from(arrayBuffer);

  return {
    contents: [
      {
        uri: uri.href,
        blob: buffer.toString("base64"),
      },
    ],
  };
}
```

### 3. Prompts

[Prompts](https://xmcp.dev/docs/core-concepts/prompts) are parameterized instruction templates that provide structured ways for users to interact with AI agents. They enable consistent, reusable prompt patterns.

### 4. Middlewares

[Middlewares](https://xmcp.dev/docs/core-concepts/middlewares) are processing layers for handling authentication, logging, rate limiting, and other cross-cutting concerns.


## What is Gram?

[Gram](https://getgram.ai) is a platform for deploying MCP servers with enterprise-grade infrastructure. It handles hosting, scaling, security, and monitoring so you focus on building tools. It provides the following features:

- **Serverless Scaling**: Sub-second cold starts with pay-per-use pricing
- **Security**: OAuth, SSO, and role-based access controls
- **Observability**: Built-in monitoring showing actual request/response pairs
- **Preview Deployments**: Automatic deployments from GitHub commits and pull requests
- **Multiple Client Support**: Works with Claude Web, Claude Desktop, Cursor, and other MCP-compatible clients

Gram supports two approaches for creating MCP servers:

- **OpenAPI documents**: You upload an OpenAPI spec, and Gram converts operations into tools automatically.
- **Gram Functions**: You write tools in TypeScript using a straightforward API, and Gram handles protocol details and deployment. This is what you will use for this guide.

## Why use xmcp and Gram?

xmcp handles MCP protocol complexity while Gram handles infrastructure and deployment. Together, they let you build and ship production-ready MCP servers without protocol boilerplate or DevOps work.

![xmcp and Gram architecture](/img/guides/xmcp/xmcp-gram-architecture.png)

## Building an MCP server with xmcp and Gram

You'll build an MCP server with an email-sending tool and an email list resource, then deploy it to Gram. You can find the code for this guide in the [Speakeasy example repository](https://github.com/speakeasy-api/examples), in the `xgram` directory.

## Prerequisites

You'll need:

- A [Gram account](https://getgram.ai)
- Node.js installed
- Basic TypeScript knowledge
- A [Resend account](https://resend.com/) (for the email API)

## Project setup

Create a new Gram Functions project:

```bash
pnpm create @gram-ai/function@latest --template gram
```

Answer the prompts:

- What do you want to call your project? -> `x-gram`
- What directory should we create the project in? -> `x-gram`
- Initialize a git repository? -> `Yes`
- Install dependencies with pnpm? -> `Yes`
- Install the Gram CLI? Required to deploy tools to Gram. -> `Yes`

This creates a Node.js project with the following structure:

```txt
.
├── package.json
├── pnpm-lock.yaml
├── README.md
├── src
│   ├── gram.ts
│   ├── server.ts
└── tsconfig.json
```

Install `xmcp` and dependencies:

```bash
pnpm i xmcp@^0.5.4 zod@^3 resend@^6.6.0 
```

### Adding the send email tool

Create a `tools` directory within `src`. In the tools directory, create a file `src/tools/send_email.ts`.

```ts 
// src/tools/send_email.ts

import { z } from "zod";
import { type InferSchema } from "xmcp";
import { Resend } from "resend";

export const schema = {
  email: z.string().email().describe("Recipient email address"),
  subject: z.string().min(1).describe("Email subject"),
  text: z.string().min(1).describe("Email body (plain text)"),
  from: z
    .string()
    .optional()
    .describe('Optional sender like "Acme <onboarding@resend.dev>"'),
};

export const metadata = {
  name: "send_email",
  description: "Send an email using Resend",
};

export default async function sendEmail(input: InferSchema<typeof schema>) {
  const apiKey = process.env["RESEND_API_KEY"];
  if (!apiKey) {
    return {
      success: false,
      error: "Missing RESEND_API_KEY",
    };
  }

  const from = input.from ?? "Acme <onboarding@resend.dev>";

  try {
    const resend = new Resend(apiKey);
    const result = await resend.emails.send({
      from,
      to: [input.email],
      subject: input.subject,
      html: `<p>${input.text}</p>`,
      text: input.text,
    });

    if (result.error) {
      return {
        success: false,
        error: result.error.message,
      };
    }

    return {
      success: true,
      id: result.data?.id,
      message: "Email sent successfully",
    };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : String(error),
    };
  }
}
```

The tool has three components:

- **schema** - Zod validation defining input parameters with descriptions
- **metadata** - The tool's name and description for AI agents
- **handler** - The async function implementing the tool's logic

The tool reads the  `RESEND_API_KEY` from environment variables (Gram injects environment variables securely at runtime) and validates inputs with the Zod schema. The tool returns error objects instead of throwing because MCP tools must return valid responses with errors communicated through the response structure. The `success` boolean lets AI agents check if the operation succeeded.

`InferSchema<typeof schema>` extracts TypeScript types from the Zod schema, giving the application full type safety.

### Registering the tool

Inside the `src/gram.ts` file, register the tool we created. Replace the existing code.

```ts 
// src/gram.ts

import { withGram } from "@gram-ai/functions/mcp";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

// Import xmcp tools
import sendEmailHandler, {
  schema as sendEmailSchema,
  metadata as sendEmailMetadata,
} from "./tools/send_email.ts";


// Create MCP server instance
const server = new McpServer(
  {
    name: "xmcp-gram",
    version: "0.1.0",
  },
  {
    capabilities: {
      tools: {},
      resources: {},
    },
  }
);

server.registerTool(
  sendEmailMetadata.name,
  {
    title: sendEmailMetadata.name,
    description: sendEmailMetadata.description,
    inputSchema: sendEmailSchema,
  },
  async (args) => {
    const result = await sendEmailHandler(args);
    return {
      content: [
        {
          type: "text",
          text: typeof result === "string" ? result : JSON.stringify(result),
        },
      ],
    };
  }
);

export { server };

// Wrap with Gram Functions
export default withGram(server, {
  variables: {
    RESEND_API_KEY: { description: "API key for Resend" },
  },
});
```

The `withGram()` wrapper transforms the MCP server into a serverless function that Gram deploys. It handles HTTP-to-MCP protocol conversion, authentication, and observability. When you run `pnpm build`, Gram bundles `gram.ts` as the entry point. When you run `pnpm push`, Gram deploys this bundle.

## Running the MCP server locally

Because we wrapped the Gram server in the `withGram` function, we need to modify the `src/server.ts` file, which is the server that can run locally. Replace the existing code.

```ts
// src/server.ts

import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { server } from "./gram.ts";

async function run() {

  console.error("Starting MCP server with stdio...");
  const stdio = new StdioServerTransport();
  await server.connect(stdio);

  const quit = async () => {
    console.error("\nShutting down MCP server...");
    await server.close();
    process.exit(0);
  };
  process.once("SIGINT", quit);
  process.once("SIGTERM", quit);
}

run();
```

Use this command to run the server: 

```
pnpm dev
```

You should be redirected to the MCP Inspector in your default browser.

![MCP Inspector](/img/guides/xmcp/mcp-inspector.png)

## Building and deploying

Build the application to create a deployment bundle:

```bash
pnpm build
```

Once it's done, run the following command to deploy to Gram.

```bash
pnpm push
```

After deployment completes, you'll see a new source called `x-gram` on the **Toolsets** page.

![Deployed source on Gram](/img/guides/xmcp/gram-deployed-source.png)

### Creating a toolset

Click **+ CREATE A TOOLSET**, name it `x-gram`, and confirm. This creates an empty toolset.

Click **ADD TOOLS** and select the `send_email` tool from the `x-gram` toolset.

![Adding tools to toolset](/img/guides/xmcp/gram-add-tools.png)

### Configuring authentication

Navigate to the **Auth** tab and add the Resend API key into an environment and attach that enrivonment to the toolset:

- **Variable name**: `RESEND_API_KEY`
- **Value**: Your Resend API key

### Testing the tool

Open the **Playground**, select the `x-gram` toolset and the environment where you added the API key. Enter a prompt like:

```txt
Send a merry christmas email to yourtestemail@example.com
```

The tool executes and sends the email.

![Testing in playground](/img/guides/xmcp/gram-playground-test.png)


## Adding resources

While tools perform actions, resources provide data. They're read-only sources that are injected into AI agents context and can present information such as user profiles, recent activity, configuration, or any information that helps the AI make better decisions.

Resources follow the same pattern as tools. Create `src/resources/emails.ts`:

```ts
// src/resources/emails.ts

import { type ResourceMetadata } from "xmcp";

export const metadata: ResourceMetadata = {
  name: "emails",
  title: "Email List",
  description: "A list of recent emails from the inbox",
  mimeType: "application/json",
};

export default async function emailsResource(uri: URL) {
  const emails = [
    {
      id: "1",
      from: "alice@example.com",
      subject: "Project Update",
      body: "Here's the latest update...",
      date: "2025-01-15T10:30:00Z",
      read: false,
    },
    // ... more emails
  ];

  return {
    contents: [
      {
        uri: uri.href,
        text: JSON.stringify(emails, null, 2),
        mimeType: "application/json",
      },
    ],
  };
}
```

The handler receives a `URL` parameter for parameterized resources. You can check `uri.pathname` to return different data, `resources://emails/inbox` vs `resources://emails/sent`, for example. The return format uses a `contents` array with `text` for string data (JSON) or `blob` for binary data (images, PDFs) encoded as base64.

### Registering the resource

Open `src/gram.ts` and add the resource import and registration:

```ts
// src/gram.ts

import emailsResourceHandler, { metadata as emailsMetadata } from "./resources/emails.ts";
// ... existing imports

// ... existing server setup

// Register the emails resource
server.registerResource(
  emailsMetadata.name,
  `resources://${emailsMetadata.name}`,
  {
    mimeType: emailsMetadata.mimeType,
    description: emailsMetadata.description,
    title: emailsMetadata.title,
  },
  async (uri) => {
    const result = await emailsResourceHandler(uri);
    return result;
  }
);

// ... existing withGram wrapper
```

Build and deploy:

```bash
pnpm build
pnpm push
```

The resource will appear in the **Toolset → Resources** tab, ready to add to the toolset.

![Resources tab](/img/guides/xmcp/gram-resources-tab.png)

## Conclusion

By combining xmcp's protocol abstraction with Gram's deployment infrastructure, you can build and ship MCP servers in hours instead of weeks.

xmcp handles the protocol complexity with file-system based auto-discovery, type safety, and hot reloading. Gram handles the operational complexity with serverless scaling, enterprise security, and observability.

You now have a working MCP server with an email-sending tool and email list resource deployed to production. You can extend it by adding more tools in `src/tools/` or resources in `src/resources/`.

## Further reading

Now that you have a working MCP server, explore these Gram Features to enhance it:

- **[Add OAuth authentication](https://www.getgram.ai/docs/gram-functions/add-oauth)**
- **[Monitor with logs](https://www.getgram.ai/docs/gram-functions/logs)**
- **[Functions framework reference](https://www.getgram.ai/docs/gram-functions/functions-framework)**
