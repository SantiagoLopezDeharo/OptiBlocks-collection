# Document Module Microservice — integration reference

Stateless HTTP service with two jobs. It converts data between JSON, XML, CSV, TSV, TOML, YAML, MessagePack, XLSX and Java `.properties`, and it renders PDF, DOCX and XLSX files from a JSON list of actions. It has no database, no auth, no state between requests and no outbound calls. Every response is fully computed in memory before it is sent.

- Image: `slincsilver/document-module-microservice:1.0.1` (https://hub.docker.com/r/slincsilver/document-module-microservice), `linux/amd64` + `linux/arm64`, `FROM scratch`, UID `65532`
- Repo: https://github.com/SantiagoLopezDeharo/document-module-microservice

## When to use / when not

Use it for:
- generating PDFs (invoices, receipts, reports), DOCX letters or XLSX exports from backend data, without adding a PDF or Office library to the backend;
- converting uploads or exports between formats, for example a CSV/XLSX import turned into JSON, a JSON export turned into CSV/XLSX/YAML, or config turned into TOML/YAML/`.properties`;
- any place where hostile input must not crash the caller. Sizes, nesting, zip bombs and document growth are all bounded.

Do not use it for:
- PDFs with non-ASCII text, such as accents, `€`, CJK or emoji. It uses only the PDF core fonts with no Unicode translation, so `"Café €"` renders as `CafÃ© â‚¬`. Use DOCX, or a PDF library with TrueType fonts.
- HTML-to-PDF, templates, images, tables-as-objects, headers/footers, hyperlinks or charts. PDF supports only text cells, lines and colors. DOCX supports only paragraphs of styled text runs.
- reading XLSX formatting, formulas or any sheet but the first. XLSX input yields string rows from sheet 1 only.
- payloads over 32 MiB by default, or jobs longer than about 5 minutes (see timeouts).
- exposure to browsers or the public internet. It has no auth.

## Run it

```shell
docker run -d --name document-service -p 127.0.0.1:8080:8080 \
  --read-only --tmpfs /tmp:size=512m,mode=1777 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  --memory=1g --cpus=2 --pids-limit=256 \
  slincsilver/document-module-microservice:1.0.1
```

Compose service to paste into an existing stack (no `ports:`, so only services on the same network can reach it):

```yaml
  document-service:
    image: slincsilver/document-module-microservice:1.0.1
    restart: unless-stopped
    expose:
      - "8080"               # reachable as http://document-service:8080 from other services only
    read_only: true
    tmpfs:
      - /tmp:size=512m,mode=1777
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 1g
    cpus: 2
    pids_limit: 256
```

Backend env: `DOCUMENT_SERVICE_URL=http://document-service:8080`.

- **No auth.** Anyone who can reach the port can use every endpoint. Keep it on the private network. Never publish its port publicly or route it through a public ingress. For local development, bind to `127.0.0.1` only, as above.
- **Writable `/tmp`.** `/tmp` must be writable when the root filesystem is read-only, hence the `tmpfs`. When `/convert` outputs XLSX, the excelize stream writer spills worksheet XML beyond 16 MiB to a temp file in `/tmp`. If `/tmp` is not writable it silently keeps the whole worksheet in memory instead. Nothing else touches disk.
- **No in-image health check.** The image is `scratch`: no shell, `curl` or `wget`, so a Docker `HEALTHCHECK` or compose `healthcheck` command cannot run inside it. Probe `GET /health` from the orchestrator instead: Kubernetes `httpGet` liveness/readiness on port 8080, path `/health`, or a load-balancer target check. With compose, have the backend tolerate the service starting late (retry on connection errors) rather than using `depends_on: condition: service_healthy`.
- Starts in well under a second and needs no configuration.
- On `SIGTERM`/`SIGINT` it stops accepting connections and waits up to 30 s for in-flight requests. Give it a stop grace period of at least 30 s.
- Optional image verification: `cosign verify slincsilver/document-module-microservice:1.0.1 --certificate-identity=santiagolhgpro@gmail.com --certificate-oidc-issuer=https://accounts.google.com`.

## Configuration

All settings are optional environment variables.

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_ADDRESS` | `:8080` (the image also sets `ENV SERVER_ADDRESS=:8080`) | Listen address |
| `MAX_BODY_BYTES` | `33554432` (32 MiB) | Largest request body. Larger bodies get `413`. Applies to every `POST` route. |
| `MAX_ACTIONS` | `10000` | Most actions per PDF/DOCX/XLSX request. More gets `413`. |
| `MAX_XLSX_CELLS` | `200000` | Worksheet budget for `/convert/xlsx`: rows up to the highest row touched, plus cells up to the furthest column touched in each row. Total cells visited across all actions is capped at 4× this value. Over budget gets `413`. |
| `MAX_CONCURRENT_REQUESTS` | `2 × GOMAXPROCS` | Requests decoded or rendered at once. Extra requests wait, with no `429`/`503`. Go honors the container CPU limit, so `cpus: 2` gives 4 slots. |
| `PPROF_ADDR` | unset (off) | Address of an extra, unauthenticated `net/http/pprof` listener, e.g. `127.0.0.1:6060`. Leave unset in production. |

Validation: each numeric variable must be an integer `>= 1`, and surrounding whitespace is trimmed. Invalid values stop the process at startup with exit status `1`, and every problem is logged at once, e.g. `MAX_ACTIONS must be >= 1, got 0`. Logs are `log/slog` text on stderr.

Hard-coded limits (not configurable):
- nesting: 10 000 levels in every format;
- uploaded XLSX: may inflate to at most 256 MiB;
- request headers: at most 64 KiB, larger gets `431`;
- header read timeout 10 s, full request read timeout 5 min, response write timeout 5 min, keep-alive idle timeout 120 s.

## Auth model

There is none: no API key, no JWT, no TLS, no CORS handling. Every request on the port is served. Integrate as follows:

- Run it only on a private network: a compose network, a Kubernetes `ClusterIP` service, or a VPC-internal address. Do not publish the port and do not put it behind a public ingress or API gateway route.
- Only backends call it. Browsers and mobile apps never call it directly. Proxy through the backend, which applies its own authentication and authorization, then calls the service.
- If network isolation is not possible, put it behind a reverse proxy or service mesh that enforces mTLS or an auth header. The service itself cannot check credentials.
- Treat its output as derived from caller input. The service does not sanitize business content, so validate or escape user data in the backend.
- Never set `PPROF_ADDR` to a non-loopback address. If you set it, the service exposes `/debug/pprof` on that listener only. The main port always returns `404` for `/debug/pprof`.

## API contract

All routes are exact and case-sensitive. A trailing slash or a different case gets `404`. A wrong method on a known path gets `405` with an `Allow` header. Error bodies are `text/plain; charset=utf-8`, a single line `<prefix>: <detail>`.

| Method | Path | Request body | Success response |
|--------|------|--------------|------------------|
| `POST` | `/convert` | data in the input format | `200`, data in the output format |
| `POST` | `/convert/pdf` | JSON `PdfRequest` | `200`, `application/pdf` |
| `POST` | `/convert/docx` | JSON `DocxRequest` | `200`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| `POST` | `/convert/xlsx` | JSON `XlsxRequest` | `200`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| `GET` (also `HEAD`) | `/health` | – | `200`, `text/plain`, body `OK` |

Every success response sets `Content-Length`. The response is buffered server-side and never chunked. Document responses also set `Content-Disposition: attachment; filename=output.pdf` (or `output.docx` / `output.xlsx`, the filename is fixed). `/convert` sets no `Content-Disposition`.

### `POST /convert`: format selection

- Input format: query `input`, else the `Content-Type` header.
- Output format: query `output`, else the `Accept` header.
- The query wins over headers. Query values are case-insensitive (`csv`, `CSV`, `Csv`).
- Header values are matched by **substring**, checked in this order: `application/json`, then the XLSX media type, then `xml`, `csv`, `toml`, `tab-separated-values`, YAML, MessagePack, properties. The first match wins.
- **There is no `Accept` negotiation.** q-values and lists are not parsed, and `*/*` matches nothing. curl's default `Accept: */*` therefore means "no output format", which gets `400`.
- Media types that are **not** recognized: `application/yaml`, `application/problem+json`, `application/ld+json`, and `application/x-www-form-urlencoded` (curl `-d` default). Any type containing `xml` is taken as XML, e.g. `application/xhtml+xml`.
- Recommendation: always use the query string (`/convert?input=csv&output=json`). It is unambiguous and the request `Content-Type` is then ignored.
- Errors: missing format gets `400 Input or Output format not specified`. Unknown query value gets `400 unsupported format: FOO`.

| Format | Query value | Recognized request header values | Response `Content-Type` |
|--------|-------------|----------------------------------|-------------------------|
| JSON | `json` | `application/json` | `application/json` |
| XML | `xml` | anything containing `xml` (`application/xml`, `text/xml`) | `application/xml` |
| CSV | `csv` | anything containing `csv` (`text/csv`) | `text/csv` |
| TSV | `tsv` | `text/tab-separated-values` | `text/tab-separated-values` |
| TOML | `toml` | anything containing `toml` (`application/toml`) | `application/toml` |
| YAML | `yaml` | `application/x-yaml`, `text/yaml` | `application/x-yaml` |
| MessagePack | `msgpack` | `application/msgpack`, `application/x-msgpack` | `application/msgpack` |
| XLSX | `xlsx` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | same |
| Properties | `properties` | `text/x-java-properties`, `text/properties` | `text/x-java-properties` |

Send binary inputs (XLSX, MessagePack) as raw bytes: curl `--data-binary @file`, never `-d`, which strips newlines. The response `Content-Type` carries no charset parameter. Text output is UTF-8.

### Document requests: common rules

- The body is a JSON object, parsed with Go `encoding/json`. The request `Content-Type` is ignored. **Unknown fields are silently ignored**, so a typo such as `"text"` instead of `"txt"` renders an empty value with `200`. Omitted fields take zero values (`0`, `""`, `false`).
- `actions` is an array replayed in order. `type` is case-sensitive.
- The action count is checked first: more than `MAX_ACTIONS` gets `413 Invalid document: limit exceeded: N actions, at most M allowed`.
- The first failing action aborts the render: `400 Invalid document: action <index> (<type>): <reason>`, with a 0-based index. Unknown type gives `<reason>` = `unknown action type`. Over the XLSX cell budget gets `413 Invalid document: action <i> (<type>): limit exceeded: worksheet would exceed 200000 cells`.
- A body that is not valid JSON gets `400 Invalid JSON: <detail>`.
- An empty `actions` array is valid. PDF gives one blank page, DOCX an empty document, XLSX an empty workbook with `Sheet1`.

### `PdfRequest` (go-pdf/fpdf)

```ts
{
  orientation?: "P" | "L";                 // default "P"
  unit?: "pt" | "mm" | "cm" | "in";        // default "mm"; also "point", "inch"
  size?: "A1"|"A2"|"A3"|"A4"|"A5"|"A6"|"Letter"|"Legal"|"Tabloid"; // default "A4", case-insensitive
  actions: PdfAction[];
}
```

An unknown unit or size gets `400 Invalid document: invalid request: page setup: incorrect unit furlong` / `unknown page size b5`. Margins are 1 cm. Automatic page breaks are on, with a 2 cm bottom margin.

| `type` | Fields | Semantics |
|--------|--------|-----------|
| `AddPage` | – | Starts a page. **Must come first**: text before it is not rendered. |
| `SetFont` | `family`, `style`, `size` (points) | `family` (case-insensitive) is one of `Arial` (= Helvetica), `Helvetica`, `Times`, `Courier`, `Symbol` (= ZapfDingbats), `ZapfDingbats`. Anything else gets `400 ... undefined font: <name>`. `style` is `""` or any combination of `B`, `I`, `U` (underline), `S` (strikeout). **Required before any `Cell`/`MultiCell`**, otherwise `400 ... font has not been set; unable to render text`. |
| `Cell` | `w`, `h`, `txt`, `border`, `ln`, `align` | Single-line cell. `w: 0` extends to the right margin. `h: 0` draws nothing but advances x. `border`: `""` none, `"1"` full, or any of `L`,`T`,`R`,`B`. `ln`: `0` stay right, `1` next line, `2` below. `align`: `L`/`C`/`R` plus optional vertical `T`/`M`/`B`/`A`. The default is left-middle. |
| `MultiCell` | `w`, `h`, `txt`, `border`, `align` | Wrapping text block. `h` is the line height. `align`: `L`,`C`,`R`,`J`, and `""` means `J` (justified). The cursor moves to the next line afterwards. `\n` in `txt` breaks lines. |
| `Ln` | `h` | Line break: x to the left margin, y += `h`. A negative `h` uses the last cell height, and an omitted `h` (`0`) moves no vertical distance. |
| `SetXY` | `x`, `y` | Absolute position in `unit`. |
| `SetTextColor` | `r`, `g`, `b` (0–255) | Text color. |
| `SetDrawColor` | `r`, `g`, `b` | Border/line color. |
| `SetFillColor` | `r`, `g`, `b` | Accepted, but **no visible effect**: cells are always drawn without fill. |

### `DocxRequest` (vendored gingfrederik/docx)

| `type` | Fields | Semantics |
|--------|--------|-----------|
| `AddParagraph` | – | Starts a new paragraph. |
| `AddText` | `txt`, `size`, `color`, `bold`, `italic` | Appends a run to the current paragraph, and opens one if none exists. `size` is in points, `1`–`1638`. `0` or omitted means the Word default, and other values get `400 ... size must be between 1 and 1638`. `color` is a 6-digit hex with optional `#`, or `auto`, otherwise `400`. Text is XML-escaped and full UTF-8 is fine. |

There are no headings, lists, tables, images, page breaks, fonts or alignment.

### `XlsxRequest` (xuri/excelize)

The workbook starts with a single sheet named `Sheet1`. Cell references are A1-style (`B2`, `XFD1048576`). Errors from excelize become `400`, e.g. `sheet Nope does not exist`, or `cannot convert cell "A0" to coordinates`.

| `type` | Fields | Semantics |
|--------|--------|-----------|
| `NewSheet` | `sheet` | Adds a sheet (max 31 chars, Excel naming rules). |
| `SetActiveSheet` | `sheet`, or `index` (0-based) when `sheet` is empty | The sheet shown on open. An unknown sheet or index gets `400`. |
| `SetCellValue` | `sheet`, `axis`, `value` | `value` is any JSON value. Numbers become numeric cells, booleans become booleans, strings are stored as text. A string starting with `=` is **not** a formula. Counts against the cell budget. |
| `SetColWidth` | `sheet`, `start_col`, `end_col`, `width` | Column letters, e.g. `"A"`, `"C"`. |
| `SetRowHeight` | `sheet`, `row` (1-based), `height` | `row < 1` gets `400`. Counts against the budget. |
| `MergeCell` | `sheet`, `h_cell`, `v_cell` | Opposite corners of the range. Counts against the budget. |
| `NewStyle` | `style`, optional `style_ref` | `style` is required (`400 ... style is required`). It is an excelize `Style` object. Field names match the Go struct case-insensitively, because it has no JSON tags: `font` {`bold`,`italic`,`underline`,`family`,`size`,`strike`,`color`}, `fill` {`type`:"pattern",`pattern`:1,`color`:["#RRGGBB"]}, `border` [{`type`:"left",`color`,`style`}], `alignment` {`horizontal`,`vertical`,`wraptext`}, `numfmt` (int). Snake case such as `num_fmt` or `wrap_text` is **silently ignored**. `style_ref` names the style for later `SetCellStyle`. |
| `SetCellStyle` | `sheet`, `h_cell`, `v_cell`, `style_ref` or `style_id` | Applies a style to a range. An unknown `style_ref` gets `400`, and neither field gets `400 ... style_id or style_ref is required`. Counts against the budget. |

### Status codes

| Status | Cause | Body prefix |
|--------|-------|-------------|
| `200` | success | – |
| `400` | missing format | `Input or Output format not specified` |
| `400` | unknown query format | `unsupported format: X` |
| `400` | input does not parse: empty body (`Decode error: EOF`), trailing data after a JSON document, CSV row with a different field count than the header, JSON/YAML nesting over 10 000 levels (`exceeded max depth`) | `Decode error:` |
| `400` | XLSX upload inflating beyond 256 MiB (zip bomb), corrupt workbook | `Decode error:` |
| `400` | document body not JSON | `Invalid JSON:` |
| `400` | invalid document action or page setup | `Invalid document:` |
| `404` | unknown path (including trailing slash, wrong case) | `404 page not found` |
| `405` | wrong method on a known path | `Method Not Allowed` |
| `413` | body over `MAX_BODY_BYTES` | `Error reading body: http: request body too large` |
| `413` | XML, TOML, MessagePack or `.properties` nesting over 10 000 levels | `Decode error: document nesting exceeds 10000 levels` |
| `413` | more than `MAX_ACTIONS` actions, XLSX cell budget exceeded | `Invalid document: ... limit exceeded: ...` |
| `422` | parsed input the output format cannot represent | `Encode error:` |
| `431` | request headers over 64 KiB | (Go default) |
| `500` | unexpected render failure; detail only in the service log | `Failed to generate document` |

Examples of `422`: a non-object list such as `[1,2]` to CSV/TSV/XLSX; a non-object to `properties`; a key that is not a valid XML name (`"bad key"`) to XML; an unsigned integer above `int64` to TOML.

Retry policy: every endpoint is a pure function of its input, so retries are safe. Retry on network errors, timeouts and `500`, with backoff. Never retry `4xx`: the request is wrong or too large and will fail again.

## Backend integration recipe

`documentClient.ts` uses only `fetch` (Node 18+) and has no dependencies. Formats always go in the query string. Errors become `DocumentServiceError` with the status and the service's message. Use `render` to get a `Buffer`, for example to store it or attach it to an email. Use `renderStream` to forward the document to an HTTP response without holding it twice.

```ts
import type { ReadableStream as NodeReadableStream } from "node:stream/web";

export type DataFormat =
  | "json" | "xml" | "csv" | "tsv" | "toml" | "yaml" | "msgpack" | "xlsx" | "properties";

export type DocumentKind = "pdf" | "docx" | "xlsx";

export type PdfFont = "Arial" | "Helvetica" | "Times" | "Courier" | "Symbol" | "ZapfDingbats";

export type PdfAction =
  | { type: "AddPage" }
  | { type: "SetFont"; family: PdfFont; style?: string; size: number }
  | { type: "Cell"; w: number; h: number; txt?: string; border?: string; ln?: 0 | 1 | 2; align?: string }
  | { type: "MultiCell"; w: number; h: number; txt?: string; border?: string; align?: "L" | "C" | "R" | "J" }
  | { type: "Ln"; h: number }
  | { type: "SetXY"; x: number; y: number }
  | { type: "SetTextColor" | "SetDrawColor" | "SetFillColor"; r: number; g: number; b: number };

export interface PdfRequest {
  orientation?: "P" | "L";
  unit?: "pt" | "mm" | "cm" | "in";
  size?: "A1" | "A2" | "A3" | "A4" | "A5" | "A6" | "Letter" | "Legal" | "Tabloid";
  actions: PdfAction[];
}

export type DocxAction =
  | { type: "AddParagraph" }
  | { type: "AddText"; txt: string; size?: number; color?: string; bold?: boolean; italic?: boolean };

export interface DocxRequest {
  actions: DocxAction[];
}

export type XlsxAction =
  | { type: "NewSheet"; sheet: string }
  | { type: "SetActiveSheet"; sheet?: string; index?: number }
  | { type: "SetCellValue"; sheet: string; axis: string; value: unknown }
  | { type: "SetColWidth"; sheet: string; start_col: string; end_col: string; width: number }
  | { type: "SetRowHeight"; sheet: string; row: number; height: number }
  | { type: "MergeCell"; sheet: string; h_cell: string; v_cell: string }
  | { type: "NewStyle"; style: Record<string, unknown>; style_ref?: string }
  | { type: "SetCellStyle"; sheet: string; h_cell: string; v_cell: string; style_ref?: string; style_id?: number };

export interface XlsxRequest {
  actions: XlsxAction[];
}

export interface RenderRequests {
  pdf: PdfRequest;
  docx: DocxRequest;
  xlsx: XlsxRequest;
}

export interface RenderedDocument {
  body: NodeReadableStream<Uint8Array>;
  contentType: string;
  contentDisposition: string;
  contentLength?: number;
}

export class DocumentServiceError extends Error {
  readonly status: number;

  constructor(status: number, detail: string) {
    super(`document service responded ${status}: ${detail}`);
    this.name = "DocumentServiceError";
    this.status = status;
  }
}

export interface DocumentClientOptions {
  baseUrl: string;
  timeoutMs?: number;
}

export function createDocumentClient({ baseUrl, timeoutMs = 120_000 }: DocumentClientOptions) {
  async function post(path: string, body: Uint8Array | string, contentType: string): Promise<Response> {
    const res = await fetch(new URL(path, baseUrl), {
      method: "POST",
      headers: { "Content-Type": contentType },
      body: body as BodyInit,
      signal: AbortSignal.timeout(timeoutMs),
    });
    if (!res.ok) {
      throw new DocumentServiceError(res.status, (await res.text()).trim());
    }
    return res;
  }

  async function convert(data: Uint8Array | string, from: DataFormat, to: DataFormat): Promise<Buffer> {
    const query = new URLSearchParams({ input: from, output: to });
    const res = await post(`/convert?${query}`, data, "application/octet-stream");
    return Buffer.from(await res.arrayBuffer());
  }

  async function renderStream<K extends DocumentKind>(kind: K, request: RenderRequests[K]): Promise<RenderedDocument> {
    const res = await post(`/convert/${kind}`, JSON.stringify(request), "application/json");
    const length = res.headers.get("Content-Length");
    return {
      body: res.body as NodeReadableStream<Uint8Array>,
      contentType: res.headers.get("Content-Type") ?? "application/octet-stream",
      contentDisposition: res.headers.get("Content-Disposition") ?? `attachment; filename=output.${kind}`,
      contentLength: length === null ? undefined : Number(length),
    };
  }

  async function render<K extends DocumentKind>(kind: K, request: RenderRequests[K]): Promise<Buffer> {
    const res = await post(`/convert/${kind}`, JSON.stringify(request), "application/json");
    return Buffer.from(await res.arrayBuffer());
  }

  async function healthy(): Promise<boolean> {
    try {
      const res = await fetch(new URL("/health", baseUrl), { signal: AbortSignal.timeout(2_000) });
      return res.ok;
    } catch {
      return false;
    }
  }

  return { convert, render, renderStream, healthy };
}

export type DocumentClient = ReturnType<typeof createDocumentClient>;
```

Notes:
- `body as BodyInit` is there because TS 5.9 lib types reject `Uint8Array<ArrayBufferLike>`, which includes `Buffer`, as a fetch body. At runtime Node accepts it.
- `timeoutMs` should exceed the slowest expected render, plus queueing time when all slots are busy. Keep it below the service's 5-minute write timeout.
- The body is uploaded in one piece on purpose. The service reads the whole body before processing, so streaming the upload gains nothing.
- Usage:
  - `await documents.convert(csvBuffer, "csv", "json")` returns a JSON `Buffer`. Call `JSON.parse(buf.toString("utf8"))` on it.
  - `await documents.convert(JSON.stringify(rows), "json", "xlsx")` returns XLSX bytes.

Express: stream a generated PDF to the client, passing `Content-Type`, `Content-Disposition` and `Content-Length` through. Override `Content-Disposition` to set a meaningful filename; the service always sends `output.<ext>`.

```ts
import { Readable } from "node:stream";
import { pipeline } from "node:stream/promises";
import express from "express";
import { createDocumentClient } from "./documentClient";

const documents = createDocumentClient({ baseUrl: process.env.DOCUMENT_SERVICE_URL ?? "http://document-service:8080" });

export const app = express();

app.get("/invoices/:id/pdf", async (req, res, next) => {
  try {
    const doc = await documents.renderStream("pdf", {
      actions: [
        { type: "AddPage" },
        { type: "SetFont", family: "Arial", style: "B", size: 16 },
        { type: "Cell", w: 0, h: 10, txt: `Invoice ${req.params.id}`, ln: 1 },
      ],
    });
    res.setHeader("Content-Type", doc.contentType);
    res.setHeader("Content-Disposition", doc.contentDisposition);
    if (doc.contentLength !== undefined) res.setHeader("Content-Length", doc.contentLength);
    await pipeline(Readable.fromWeb(doc.body), res);
  } catch (err) {
    next(err);
  }
});
```

NestJS equivalent. `StreamableFile` sets the headers from its options:

```ts
import { Readable } from "node:stream";
import { Controller, Get, Param, StreamableFile } from "@nestjs/common";
import { createDocumentClient } from "./documentClient";

const documents = createDocumentClient({ baseUrl: process.env.DOCUMENT_SERVICE_URL ?? "http://document-service:8080" });

@Controller("invoices")
export class InvoiceController {
  @Get(":id/pdf")
  async pdf(@Param("id") id: string): Promise<StreamableFile> {
    const doc = await documents.renderStream("pdf", {
      actions: [
        { type: "AddPage" },
        { type: "SetFont", family: "Arial", style: "B", size: 16 },
        { type: "Cell", w: 0, h: 10, txt: `Invoice ${id}`, ln: 1 },
      ],
    });
    return new StreamableFile(Readable.fromWeb(doc.body), {
      type: doc.contentType,
      disposition: doc.contentDisposition,
      length: doc.contentLength,
    });
  }
}
```

In Nest, register the client as a provider (`useFactory` reading config) instead of a module-level constant if the codebase uses DI for config. Map `DocumentServiceError` in an exception filter: `4xx` is a bug in the request your backend built, so log it and return `500`/`502` to your client; `5xx` or a timeout means the dependency is unavailable, so return `502`/`503`. Do not forward the service's error text to end users.

Import specifiers: use `./documentClient` or `./documentClient.js`, whichever the project's module resolution needs.

## Behaviour and gotchas

Numbers and types:
- JSON numbers decode to `float64`. Integers beyond ±2^53 stay exact (`int64`/`uint64`): `9007199254740993` survives JSON → YAML/MsgPack/JSON.
- **JSON → TOML writes integers as floats**: `{"port":8080}` becomes `port = 8080.0`.
- JSON → MessagePack also encodes `1` as a float64 (`cb 3f f0 ...`). Integers beyond 2^53 stay integers. A `uint64` above `int64` max to TOML gets `422`.
- Numbers outside the float64 range (e.g. `1e400`) give `400 Decode error`.
- CSV, TSV, XLSX and XML inputs produce **strings only**: `"id": "1"`, attributes `"-id": "7"`. Convert types in the backend.
- `.properties` input infers numbers and booleans: `a.b=1` becomes `1`, `x=true` becomes `true`.
- YAML input dates become timestamps: `2020-01-01` turns into `"2020-01-01T00:00:00Z"` in JSON.
- YAML output quotes ambiguous keys and scalars, e.g. key `"n"`.

Shapes per format:
- **JSON output**: indented 2 spaces, map keys sorted alphabetically, `<>&` not HTML-escaped, trailing newline.
- **CSV/TSV/XLSX output**:
  - needs a list of objects or one object;
  - columns are the sorted union of all keys, and nested objects are flattened to `parent.child` columns;
  - a missing key or `null` gives an empty cell;
  - **keys that contain a literal `.` lose their value**: `[{"a.b":1}]` → header `a.b`, empty cell. The column lookup splits on dots and looks for nested `a` → `b`. The same applies when re-encoding flattened data, so XLSX/CSV read back with a `user.name` column and written to CSV again comes out empty. Nest such data as objects or rename keys before converting;
  - **arrays inside a row are written in Go syntax**, `["a","b"]` → `[a b]`. Flatten arrays in the backend first;
  - an empty list gives `200` with an empty body (CSV) or an empty workbook (XLSX). XLSX writes to `Sheet1`.
- **CSV/TSV/XLSX input**: first row is the header. Empty cells are omitted from the row object. A CSV/TSV row with more or fewer fields than the header fails with `400 ... wrong number of fields`. XLSX reads the first sheet only and keeps only cell display values.
- **XML input**: the root element is unwrapped (`<book id="7"><title>Go</title></book>` gives `{"-id":"7","title":"Go"}`). Repeated `<List>` children of the root become a top-level list. Attributes are `-name` keys, mixed text is `#text`. External entities are never fetched.
- **XML output**:
  - one-key objects use that key as the root; multi-key objects are wrapped in `<doc>`; top-level lists become `<doc><List>…</List><List>…</List></doc>`;
  - `-name` keys become attributes and `#text` becomes element text;
  - keys must be valid XML names, otherwise `422`. Values are escaped.
- **TOML**: there are no top-level arrays, so a list is written as `root = [...]` and read back unwrapped.
- **Properties output**: needs an object. Keys are flattened (`db.host`, `db.ports[0]`) and sorted. Special characters are escaped. `${...}` is never expanded in either direction.
- Round trips through CSV/XLSX/XML/properties lose types and nesting details. JSON, YAML and MessagePack round-trip faithfully.

Documents:
- **PDF text must be ASCII.** Core fonts with no Unicode translator: UTF-8 bytes are drawn as cp1252 glyphs (`é` → `Ã©`). DOCX and XLSX handle full UTF-8.
- **PDF order matters**: `AddPage`, then `SetFont`, then text. Text before `AddPage` is silently dropped. Text before `SetFont` is a `400`.
- **`SetFillColor` has no visible effect**: `Cell` and `MultiCell` are always drawn unfilled. Use borders (`SetDrawColor`) or text color for emphasis.
- PDF page breaks happen automatically 2 cm above the bottom edge. Long `MultiCell` text flows onto new pages.
- Unknown JSON fields in document requests are ignored silently, so validate field names with the TS types above.
- XLSX style objects: lower-case Go field names only (`wraptext`, `numfmt`). Snake case is ignored.
- XLSX amplification guard: a single cell at `XFD1048576`, row `1048576` or a full-sheet merge/style range exceeds the 200 000-cell budget and gets `413` in milliseconds. Keep generated sheets dense and close to `A1`.

Limits, timeouts and errors:
- **Zip bombs return `400`, not `413`**: an XLSX upload inflating beyond 256 MiB gives `Decode error: ...`.
- Nesting deeper than 10 000 levels is rejected in every format, but the status differs. XML, TOML, MessagePack and `.properties` get `413`. JSON and YAML get `400` because their parsers enforce the limit. MessagePack containers that claim more items than the body holds get `400` before any allocation.
- The concurrency limit queues without a timeout. Excess requests wait until a slot frees up or the client disconnects. The client timeout is the only bound, and there is no `429`/`503` to back off on.
- A request whose handling exceeds the 5-minute write timeout is cut off without a status (connection reset). Uploads must finish within 5 minutes and headers within 10 s.

Memory:
- Every request holds its whole body, the decoded value and the whole output in memory at once.
- Peak memory is roughly `MAX_CONCURRENT_REQUESTS` × (body + decoded structure + output). The decoded structure is several times the input size for CSV/JSON.
- 1 GiB with the default 32 MiB body cap and 4 slots is the tested envelope. Raise `mem_limit` or lower `MAX_CONCURRENT_REQUESTS` if you raise `MAX_BODY_BYTES`.
- The `/tmp` tmpfs is RAM-backed and counts toward the container's memory cgroup. It is only used by XLSX output above 16 MiB of sheet XML.

Behaviour on failure:
- Output is never partial. A failure anywhere yields an error status, never a truncated `200`.
- `500` bodies never contain internals. `4xx` bodies contain parser messages that echo fragments of the input, so do not forward them to untrusted users verbatim.

## Smoke test checklist

Run these from a host where the service is reachable. For a published dev port, `B=http://127.0.0.1:8080`. Inside a compose network without a published port, use a throwaway curl container: `docker run --rm --network <project>_default curlimages/curl:8.10.1 ...` with `B=http://document-service:8080`. Each command prints the expected result in a comment.

```shell
B=http://127.0.0.1:8080

# 1. health -> 200 OK
curl -s -w ' %{http_code}\n' "$B/health"

# 2. CSV -> JSON (query formats) -> [{"city":"London","id":"1","name":"Ada"}] with string values
printf 'id,name,city\n1,Ada,London\n' | curl -s -X POST "$B/convert?input=csv&output=json" --data-binary @-

# 3. header formats + 64-bit integer preserved -> YAML with id: 9007199254740993
curl -s -X POST "$B/convert" -H 'Content-Type: application/json' -H 'Accept: application/x-yaml' \
  -d '{"id": 9007199254740993, "tags": ["go", "docs"]}'

# 4. JSON -> XLSX -> CSV round trip -> "id,name" then "1,Ada"
curl -s -X POST "$B/convert?input=json&output=xlsx" -d '[{"id":1,"name":"Ada"}]' -o /tmp/smoke.xlsx
curl -s -X POST "$B/convert?input=xlsx&output=csv" --data-binary @/tmp/smoke.xlsx

# 5. PDF -> 200 application/pdf, Content-Disposition: attachment; filename=output.pdf, file starts with %PDF-
curl -s -D - -o /tmp/smoke.pdf -X POST "$B/convert/pdf" -H 'Content-Type: application/json' \
  -d '{"actions":[{"type":"AddPage"},{"type":"SetFont","family":"Arial","style":"B","size":16},{"type":"Cell","w":0,"h":10,"txt":"Smoke test","ln":1,"align":"C"}]}' | grep -iE '^(HTTP|content-)'
head -c 5 /tmp/smoke.pdf; echo

# 6. DOCX and XLSX -> 200 each, files start with PK (zip)
curl -s -o /tmp/smoke.docx -w '%{http_code}\n' -X POST "$B/convert/docx" -d '{"actions":[{"type":"AddText","txt":"Olá","bold":true}]}'
curl -s -o /tmp/smoke-doc.xlsx -w '%{http_code}\n' -X POST "$B/convert/xlsx" \
  -d '{"actions":[{"type":"SetCellValue","sheet":"Sheet1","axis":"A1","value":"Hello"}]}'

# 7. negative checks
curl -s -w ' %{http_code}\n' -X POST "$B/convert" -d '{}'                                        # 400 Input or Output format not specified
curl -s -w ' %{http_code}\n' -X POST "$B/convert?input=json&output=csv" -d '[1,2]'               # 422 Encode error: csv/tsv requires an array of objects
curl -s -w ' %{http_code}\n' -X POST "$B/convert/pdf" -d '{"actions":[{"type":"Rotate"}]}'      # 400 Invalid document: action 0 (Rotate): unknown action type
curl -s -w ' %{http_code}\n' -X POST "$B/convert/xlsx" \
  -d '{"actions":[{"type":"SetCellValue","sheet":"Sheet1","axis":"XFD1048576","value":1}]}'      # 413 ... worksheet would exceed 200000 cells
curl -s -o /dev/null -w '%{http_code}\n' -X GET "$B/convert"                                      # 405
```

From the backend, also check that `await documents.healthy()` returns `true` and that one `render("pdf", ...)` call returns a `Buffer` starting with `%PDF-`.
