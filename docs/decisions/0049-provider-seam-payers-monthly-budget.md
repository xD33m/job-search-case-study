# Provider profiles, a payer on every call, and a hard monthly AI budget

*Adapted from ADR 0049 of the private repository (hosted edition, step H4a) and its amendment by ADR 0057.*

## Context

The multi-user hosted edition needed three things the single-user code didn't have:

- A second LLM provider with EU processing (Mistral) next to DeepSeek. The client was DeepSeek's in every detail: its strict endpoint, its reasoning switch, its cache usage fields, prices, peak windows and finish reasons.
- Separate accounting. The ledger counted one platform-wide day, so one user's personal calls ate into the budget for shared fact extraction.
- Plan checks that also hold for queued work. Pro features were checked only at the route, so a draft queued before a downgrade still ran.

## Options

| Question | Chosen | Why |
|---|---|---|
| A second provider | **One OpenAI-compatible client plus a `ProviderProfile`** (not a second client class) | Retries, fallbacks, ledger and tests stay one code path |
| Who pays for a call | **A payer on every reservation and usage row** | The platform's facts and a user's fits no longer share a cap |
| Payers in personal mode | One day over every row, as before | Counting apart would double the owner's daily spend |
| A hosted user's limit | **A monthly USD cap set by the platform** | Users must not set their own limit |
| Where Pro is checked | **At the route and right before the AI call** | Queued work may run after a revoke |

## Decision

- **`ProviderProfile`** is a frozen dataclass holding every provider difference: base URL, strict-mode style (tool call or `json_schema`), extra request body, usage reader, price table, peak windows, output cap and the finish reasons that mean "overloaded, retry".

  ```python
  @dataclass(frozen=True)
  class ProviderProfile:
      name: ProviderName
      base_url: str
      strict_mode: StrictMode          # "tool_beta" | "json_schema"
      extra_body: Mapping[str, object]
      usage_reader: UsageReader
      prices: PriceTable
      peak_windows: PeakWindows        # empty: no peak tier
      max_output_tokens: int
      retryable_finish_reasons: frozenset[str]
  ```

- **Keys only go to their own provider.** A configuration whose base URL or model names belong to a different known provider refuses to start, and the error names the variables, never their values. This closes a real trap: an old `.env` with one provider's URL plus a switch to the other provider would have sent the second provider's key to the first. In hosted mode, the base URL is locked to the provider's own.
- **Payers.** A nullable `workspace_id` on usage and reservation rows; NULL means the platform. The platform pays for facts, salary estimates and employer ratings. The workspace pays for fits, search terms, mail classification, drafts and notes. In hosted mode each payer's caps count only its own rows.
- **A monthly budget.** A ledger *policy* enum (`DAILY_ONLY` or `MONTHLY_PER_WORKSPACE`, so no mode can express an unused combination). The default is $2.00 per workspace per local calendar month, checked as `spent + held + estimate > cap` in the same write transaction as the reservation. A reservation's instant decides its month. Hosted users get no "go over the budget" override.
- **Checks at the effect.** Each service checks the plan right before it builds a client. Work refused after a revoke ends with the plan's explanation instead of running.
- **Cost measured, not guessed.** A tool replays fixture token counts from the provider spike through the real ledger for a simulated month.

## Consequences

- The cost replay puts one Pro user's own fits at about **$0.40/month on DeepSeek** (prices verified) and **$0.55/month on Mistral** (third-party prices, not yet checked with a real key). That is 20–27 % of the $2.00 budget. The shared platform work comes to about $4–6 a month at 200 new cards a day, whatever the number of users. Both figures are upper bounds, because salary calls were replayed at extraction-sized token counts.
- A later amendment made the provider a choice *per payer*: a workspace can use its own key, stored encrypted with AES-GCM, and its calls then count towards no cap.
- Mistral's request shape, output cap and quality are tested only against a mock transport so far. A real-key check is an open item in [EVALS.md](../../EVALS.md).
