# Public QLoRA Evaluation Dashboard

This is a new, de-identified visualization built exclusively from a whitelist of aggregate measurements. It is **not** the original private dashboard and contains **no raw prompts, SQL, model answers, training examples, internal configuration, or private endpoints**.

## Open the dashboard

- **Interactive HTML:** [`index.html`](index.html). Open the downloaded HTML in a browser for the full dashboard. GitHub's source-code viewer does not execute the page.
- **Inspectable data:** [`aggregate_metrics.json`](aggregate_metrics.json). The embedded chart values exactly match this aggregate-only JSON.
- **Research report:** [Extended Evaluation, 2026-10-13](../EXTENDED_EVALUATION_20261013.md).

The dashboard uses inline CSS, JavaScript and SVG. It requires no build system, external libraries, trackers, or API/network requests. A live Cloudflare Pages link can be added here after deployment.

## Interpretation boundaries

- Quality classifications come from one AI-assisted reviewer grading an application-level assistant, including prompting, retrieval and possible data tools. They are **not** isolated LoRA accuracy.
- The 8-stream result (**110.8 tok/s**) is **combined throughput** from artificial, fixed-length, reasoning-OFF requests, not single-user speed or validated production capacity.
- At 32 streams the highest observed TTFT was **83.32 seconds**; higher aggregate throughput is not the same as better user experience.
- The 170,979-token observation is **one needle-retrieval test**; it does not prove long-context reasoning reliability.
- The 134-case holdout was **not scored**, and five verbatim training-question overlaps were detected.

The de-identified research summary was authorized for publication by the project owner; private records and source systems are not distributed.
