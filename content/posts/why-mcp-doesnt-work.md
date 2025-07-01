+++
date = '2025-07-01T01:00:00+01:00'
draft = false
title = 'Why MCP Doesn’t Work: Lessons from the Trenches at FINN'
mermaid = true
+++

When we first heard of function calling (MCP), we at the Customer Ops Team thought we’d found the silver bullet for customer care, finally! MCP (Model Context Protocol) promised a dream: giving our Slack chatbot (beloved Finny) the power to autonomously call any API, figure out exactly which endpoints to use, and seamlessly answer complex customer questions.

For context: FINN is a car subscription which handles the complexity of car ownership for B2C and B2B customers. Behind the scenes, that means hundreds of internal APIs powering everything from booking to billing, logistics to support.

As a software engineer in the Operations team, owning our Subscriptions-API and customer care tooling, I was pretty stoked. Until we started testing.

### The Dream: What MCP Promised

MCP would expose a interface for LLM's to dynamically explore and navigate APIs, combining responses on-the-fly to deliver perfect, human-like answers to any customer inquiry. Customer care agents could toss any question at Finny and relax while it flawlessly dug out the exact answers from the depths of our APIs. "I'll never build internal tooling again, everything Chatbot!" - I thought.

### So, What Exactly is MCP?

MCP, or Model Context Protocol, is a structured way for Large Language Models (LLMs) like GPT-4.1 to use API's or any other interface. In essence MCP is a way to expose interfaces to LLM's. It consists of an MCP-Server that hosts the interfaces and connects to data sources, an MCP client that uses these interfaces, and an LLM that instructs the MCP client on which resources to access.

### The Reality Check: Why It Didn’t Work

But, as the saying goes, “everyone has a plan until they get punched in the face.” After integrating just a handful of endpoints, our glorious vision began to collapse spectacularly.

#### Issue #1: Hallucinations and False Confidence

Finny, our overconfident AI intern, loved answering questions. The trouble? It often confidently gave completely wrong answers. With no clear understanding of its own limits, it misled customer care agents, who then had to untangle these creative hallucinations. For example, when asked if a customer’s car delivery was delayed, Finny checked the totally unrelated product info endpoint and cheerfully declared, “Nope, all good!” - spoiler alert: it wasn’t.

{{< figure src="/images/finny-delayed-yes-no.jpg" alt="Finny's mistaken response" caption="Figure: Finny has not access to GET /delays but confidenlty states there are no delays found." >}}

#### Issue #2: Overwhelmed by API Complexity

Our APIs are complex, each containing around 80-100 endpoints with granular, CRUD-heavy details. Finny quickly drowned in this sea of data, unable to piece together critical connections. To determine if a car delivery was delayed, Finny needed to compare subscription dates with delivery schedules. Something it never managed.

{{< mermaid >}}
graph TD
A[Start] --> B[Get Subscription by ID]
B --> C[Get car_id from Subscription]
C --> D[Get Car by car_id]
D --> E[List Deliveries<br/>filter by rideType handover]
E --> F[Filter out canceled deliveries]
F --> G[Compare delivery date vs preferred handover date]
G -->|Equal| H[No delay]
G -->|Unequal| I[Operational Delay]
{{< /mermaid >}}
_Figure: Flow to determine if a delivery was delayed based on subscription and delivery data._

#### Issue #3: Scalability Nightmare

With just five endpoints integrated and another 300 endpoints looming ominously, scaling MCP felt about responsible as hardcoding API keys. Every new endpoint we added just piled on complexity without making Finny smarter.

### Real-Life Code

#### Remote MCP Server (Cloud Deployment)

Yeah, our MCP gateway is in AWS (nice try script kiddies, it’s token and WAF protected):

```typescript {linenos=true style=monokai}
// MCP Gateway server setup (running remotely, secured via API token)
app.post("/mcp", checkApiKey, async (req: Request, res: Response) => {
  const transport = new StreamableHTTPServerTransport();
  res.on("close", () => {
    void transport.close();
    void server.close();
  });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});
```

#### Endpoint Definition - Organized, Still Not Enough

```typescript {linenos=true style=monokai}
server.setRequestHandler(ListToolsRequestSchema, () => ({
  tools: [
    {
      uri: "tools://get_subscription_by_id",
      name: "get_subscription_by_id",
      description: "Returns subscription details by ID.",
      inputSchema: zodToJsonSchema(z.object({ subscription_id: z.string() })),
      outputSchema: zodToJsonSchema(SubscriptionSchema),
    },
    // more endpoints ...
  ],
}));
```

#### When MCP Went Totally Rogue

Here’s where Finny confused itself spectacularly:

```typescript {linenos=true style=monokai}
export async function fetchWebsiteConfiguration(
  params: WebsiteConfigParamsType
) {
  const upstream = `${process.env.FLEET_CONFIG_API_BASE}/api/v1/websiteConfigurations/${params.finn_car_id}`;
  const response = await axios.get(upstream, {
    headers: {
      "x-finn-actor": "mcp-gateway",
      "x-api-key": process.env.FLEET_CONFIG_API_KEY,
    },
  });
  return {
    content: [{ type: "text", text: JSON.stringify(response.data, null, 2) }],
  };
}
```

Finny kept choosing this endpoint to answer delay-related questions simply because it was “car-related.” Oops.

### But Wait, Moments of Magic Did Happen!

There were a few golden moments when MCP actually worked:

Customer Care: “Did we accidentally charge €5 too much?”
Finny: Fetches invoice data and nails it immediately: “Yep, spotted the €5 discrepancy!”

This felt amazing but also highlighted the crucial point: MCP shines when APIs are crystal clear, queries are only CRUD and explicitly crafted for exact questions.

### So, What Actually Works?

The key lesson is straightforward: endpoints need to be specifically crafted for MCP. Rather than exposing every single granular API, create explicit, aggregated endpoints like “getCarDelays” that encapsulate logic internally.

At FINN, we’ve now use Decagon and Relevance-AI. These AI agent builder Lets us build custom Agent Operating Procedures (AOPs). These function like guarrails and expose the complex nature of the business process to the context of the LLM. This helps us to deliver accurate, predictable answers, minus the hallucinations.

### Conclusion: The Real Challenge Lies Beyond MCP

The issue isn't with MCP itself. While today's AI models, including LLMs, excel at generating text, they fall short in understanding and solving complex internal business processes just by examining API documentation. It's unrealistic to expect an AI or even a new hire to navigate these complexities without the deep, contextual knowledge that our team members possess.

At FINN, our strength lies in the ownership and expertise of our people, who understand how to integrate multiple API resources and calls into cohesive processes. We handle the complexity of car ownership internally, shielding our customers from these intricate processes. To truly address these "context-challenges", we'd need to document every employee's knowledge into an internal database, a task we all know is easier said than done. While integrating this knowledge into the context of an LLM might be an option, it's unlikely to be the short- or mid-term solution.

For now, the path forward is clear: focus on streamlining processes, making them watertight, and automating them wherever possible. While MCP holds promise for the future, we have significant groundwork to lay in organizing and structuring our APIs and internal knowledge first.

A promising use-case for MCP emerges when companies successfully encapsulate their complex business processes behind API endpoints. In such scenarios, MCP is better suited for external exposure rather than internal use. Imagine a world in 2030 where everyone has a fully integrated LLM that can manage tasks like getting a car subscription or booking a hotel in the south of France, acting as a personal assistant all powered by companies and their exposed MCP servers. This vision underscores the potential of MCP when complexity is effectively hidden and managed.
