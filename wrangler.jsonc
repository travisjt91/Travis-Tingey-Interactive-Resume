// ============================================================
// Travis Tingey Resume — Anthropic Proxy Worker
// Deploy to Cloudflare Workers.
//
// REQUIRED SETUP:
//   1. Set secret: ANTHROPIC_API_KEY
//   2. Create KV namespace: RATE_LIMIT_KV
//   3. Bind KV namespace with variable name: RATE_LIMIT_KV
// ============================================================

const RATE_LIMIT_PER_IP  = 15;   // max requests per IP per day
const DAILY_GLOBAL_CAP   = 50;   // max requests across ALL visitors per day
const MAX_INPUT_LENGTH   = 2500; // ~500 tokens — reject oversized inputs
const MAX_OUTPUT_TOKENS  = 400;  // cap response length to control costs
const ALLOWED_ORIGIN     = "https://travisjt91.github.io";

export default {
  async fetch(request, env) {

    // --- CORS preflight ---
    if (request.method === "OPTIONS") {
      return corsResponse("", 204);
    }

    // --- Only allow your GitHub Pages domain ---
    const origin = request.headers.get("Origin");
    if (origin !== ALLOWED_ORIGIN) {
      return corsResponse(JSON.stringify({ error: "Forbidden" }), 403);
    }

    // --- Only allow POST ---
    if (request.method !== "POST") {
      return corsResponse(JSON.stringify({ error: "Method not allowed" }), 405);
    }

    const today = new Date().toISOString().slice(0, 10); // "YYYY-MM-DD"

    // --- Per-IP rate limit ---
    const ip    = request.headers.get("CF-Connecting-IP") || "unknown";
    const ipKey = `rl:${ip}:${today}`;
    let ipCount = parseInt(await env.RATE_LIMIT_KV.get(ipKey) || "0");

    if (ipCount >= RATE_LIMIT_PER_IP) {
      return corsResponse(
        JSON.stringify({ error: "You've reached the daily request limit for this demo. Try again tomorrow!" }),
        429
      );
    }

    // --- Global daily cap ---
    const globalKey   = `global:${today}`;
    let globalCount   = parseInt(await env.RATE_LIMIT_KV.get(globalKey) || "0");

    if (globalCount >= DAILY_GLOBAL_CAP) {
      return corsResponse(
        JSON.stringify({ error: "This demo has reached its daily limit. Check back tomorrow!" }),
        429
      );
    }

    // --- Parse request body ---
    let body;
    try {
      body = await request.json();
    } catch {
      return corsResponse(JSON.stringify({ error: "Invalid request." }), 400);
    }

    // --- Guard against oversized inputs ---
    const inputText = JSON.stringify(body.messages || "");
    if (inputText.length > MAX_INPUT_LENGTH) {
      return corsResponse(
        JSON.stringify({ error: "Input too long. Please shorten your message." }),
        400
      );
    }

    // --- Forward to Anthropic, injecting key + capping tokens ---
    const payload = {
      ...body,
      max_tokens: Math.min(body.max_tokens || MAX_OUTPUT_TOKENS, MAX_OUTPUT_TOKENS),
    };

    // Remove any key the client may have tried to send (safety measure)
    const upstream = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "Content-Type":      "application/json",
        "x-api-key":         env.ANTHROPIC_API_KEY,
        "anthropic-version": "2023-06-01",
      },
      body: JSON.stringify(payload),
    });

    // --- Increment both counters only after a successful forward ---
    await Promise.all([
      env.RATE_LIMIT_KV.put(ipKey,     String(ipCount + 1),     { expirationTtl: 86400 }),
      env.RATE_LIMIT_KV.put(globalKey, String(globalCount + 1), { expirationTtl: 86400 }),
    ]);

    const data = await upstream.json();
    return corsResponse(JSON.stringify(data), upstream.status);
  }
};

function corsResponse(body, status) {
  return new Response(body, {
    status,
    headers: {
      "Content-Type":                  "application/json",
      "Access-Control-Allow-Origin":   ALLOWED_ORIGIN,
      "Access-Control-Allow-Methods":  "POST, OPTIONS",
      "Access-Control-Allow-Headers":  "Content-Type",
    },
  });
}
