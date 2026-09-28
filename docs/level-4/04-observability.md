---
description: "Observe Next.js in production: instrumentation.ts register and onRequestError, OpenTelemetry traces, structured logs with request IDs."
---

# 04 · Observability

When a user reports "the dashboard was blank at 10:14", you need to answer *what
happened* without reproducing it. That requires three kinds of telemetry: **logs**
(discrete events), **traces** (the timeline of one request through rendering, data
access and external calls) and **metrics** (rates and distributions over time). Next.js
gives you hooks for each.

## `instrumentation.ts`

A file named `instrumentation.ts` in the project root (or `src/`) is loaded once when a
server instance starts. It can export two functions:

```ts title="instrumentation.ts"
import type { Instrumentation } from "next";

export async function register() {
  // runs once per server process, before requests are handled
  if (process.env.NEXT_RUNTIME === "nodejs") {
    const { startTelemetry } = await import("./lib/telemetry");
    startTelemetry();
  }
}

export const onRequestError: Instrumentation.onRequestError = async (err, request, context) => {
  const e = err as Error & { digest?: string };
  console.error(
    JSON.stringify({
      level: "error",
      msg: e.message,
      digest: e.digest,
      path: request.path,
      method: request.method,
      routePath: context.routePath,   // e.g. "/tasks/[id]/edit"
      routeType: context.routeType,   // "render" | "route" | "action" | "proxy"
      renderSource: context.renderSource,
    }),
  );
};
```

`onRequestError` is called for errors the server captures while rendering, in Route
Handlers, Server Actions and Proxy. Remember the **digest**: in production, the browser
only sees a generic message plus the digest (Level 2 · 02); logging the digest here is
how you connect a user's "Reference: 12345" to the real stack trace.

## Traces with OpenTelemetry

Next.js is instrumented with **OpenTelemetry** (OTel), the vendor-neutral standard for
traces. Once you register an OTel SDK, Next.js emits spans for incoming requests, route
rendering, `generateMetadata`, `fetch` calls and more. Two common ways to register:

```bash
npm install @vercel/otel @opentelemetry/api
```

```ts title="instrumentation.ts"
import { registerOTel } from "@vercel/otel";

export function register() {
  registerOTel({ serviceName: "acme-dashboard" });
}
```

Despite the name, `@vercel/otel` is a small wrapper that works on self-hosted Node too;
alternatively configure `@opentelemetry/sdk-node` yourself for full control. Either way,
point the exporter at any OTel-compatible backend (Jaeger, Grafana Tempo, Honeycomb,
Datadog, New Relic…) using the standard `OTEL_EXPORTER_OTLP_ENDPOINT` environment
variable. We didn't connect a tracing backend while writing this lesson, so no trace
screenshots are shown; the Next.js OpenTelemetry guide lists the exact span names
emitted by your version.

Add your own spans around important work:

```ts title="lib/reports.ts"
import { trace } from "@opentelemetry/api";

const tracer = trace.getTracer("acme-reports");

export async function buildReport(teamId: string) {
  return tracer.startActiveSpan("buildReport", async (span) => {
    span.setAttribute("team.id", teamId);
    try {
      const rows = await loadRows(teamId);
      span.setAttribute("rows.count", rows.length);
      return summarise(rows);
    } catch (err) {
      span.recordException(err as Error);
      throw err;
    } finally {
      span.end();
    }
  });
}
```

Never put personal data (emails, tokens) in span attributes.

## Structured logs with a request ID

Plain `console.log("saved")` is useless at scale. Log JSON with consistent fields, and
include a request ID so all logs for one request can be found together:

```ts title="proxy.ts (request ID excerpt)"
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const id = request.headers.get("x-request-id") ?? crypto.randomUUID();
  const headers = new Headers(request.headers);
  headers.set("x-request-id", id);
  const res = NextResponse.next({ request: { headers } });
  res.headers.set("x-request-id", id); // also return it, so users/support can quote it
  return res;
}
```

```ts title="lib/log.ts"
import "server-only";
import { headers } from "next/headers";

export async function log(level: "info" | "warn" | "error", msg: string, extra: Record<string, unknown> = {}) {
  const requestId = (await headers()).get("x-request-id") ?? undefined;
  console[level === "info" ? "log" : level](JSON.stringify({ ts: new Date().toISOString(), level, msg, requestId, ...extra }));
}
```

Note that calling `headers()` makes the caller dynamic — use this logger in actions,
handlers and dynamic pages, not in code that should be prerendered. With OTel active,
you can use the trace ID as the correlation ID instead.

## Client-side errors and vitals

Server telemetry misses errors that happen only in the browser (a hydration failure, a
bug in an event handler). Options:

- An `error.tsx` boundary's `useEffect` can report the error to your endpoint.
- `instrumentation-client.ts` (a root file that runs early in the browser) is the
  place to initialise client monitoring SDKs.
- Error-tracking services (Sentry and others) provide Next.js integrations that wire
  up server, client and source maps together. Check their docs for your Next.js version.
- Web Vitals from Level 3 · 06 complete the picture.

## What to alert on

Useful starting signals:

| Signal | Why |
|---|---|
| Error rate per route type (render / action / route) | Regressions after deploys |
| p95 server duration for key routes | Slow queries, missing cache |
| Rate of `onRequestError` with the same digest | One bug hitting many users |
| Web Vitals p75 per page type | User-perceived performance |
| Cache hit ratio (if your cache handler exposes it) | Revalidation or cache-key bugs |

## How It Actually Works

When the server boots, Next.js imports `instrumentation.ts` and awaits `register()`
before accepting requests, so SDKs can patch modules and set up exporters first. The
framework's internals call the OpenTelemetry API (`@opentelemetry/api`) to start spans;
without a registered SDK those calls are no-ops, which is why tracing costs nothing
until you opt in. OTel propagates context through async calls, so a `fetch` inside a
Server Component automatically becomes a child span of the render span, and outgoing
requests carry a `traceparent` header that downstream services can continue.
`onRequestError` is invoked from the server's error-handling paths with the original
request metadata and a context describing which kind of route failed.

## Common mistakes

- **Logging `error.message` from the client** and wondering why it says nothing useful —
  log server-side with the digest.
- **Unstructured logs** you can't query.
- **PII in logs and span attributes.**
- **Tracing only in production** — test your instrumentation locally with a Jaeger
  container or console exporter first.
- **Calling `headers()` in shared utilities** used by static pages, making them dynamic.

## Exercise

1. Add `instrumentation.ts` with `onRequestError` logging JSON. Throw from a Server
   Component in a production build and find the log line whose digest matches the one
   shown in the browser.
2. Add the request-ID proxy and the logger, and log from a Server Action. Confirm the
   same ID appears in the response header and the log.
3. If you can run Docker, start Jaeger all-in-one, register OTel with an OTLP exporter,
   and inspect the spans for one request to your Level 3 dashboard.
