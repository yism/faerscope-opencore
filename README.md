# @faerscope/opencore

Open-source pharmacovigilance opencore library for FDA FAERS adverse-event signal detection via the [openFDA API](https://open.fda.gov/apis/drug/event/).

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Node.js](https://img.shields.io/badge/node-%3E%3D18.0.0-brightgreen.svg)](https://nodejs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.4%2B-blue.svg)](https://www.typescriptlang.org/)

---

## What This Library Does

`@faerscope/opencore` provides the **core statistical and data-retrieval primitives** for building drug safety signal detection tools on top of the FDA's Adverse Event Reporting System (FAERS). It is the open-source foundation of [FAERScope](https://faerscope.com), extracted as a standalone package under the Apache-2.0 license.

This library is designed for:

- **Pharmacovigilance researchers** building custom FAERS analysis pipelines
- **Health data scientists** who need validated disproportionality calculations
- **Open-source contributors** building pharmacoepidemiology tools
- **Educators** teaching signal detection methodology

> **Important:** Disproportionality metrics quantify *statistical association* in spontaneous reports. They do **not** establish causation, incidence rates, or comparative safety. Always interpret results in clinical context. See [Limitations](#limitations).

---

## Installation

```bash
npm install @faerscope/opencore
```

```bash
pnpm add @faerscope/opencore
```

```bash
yarn add @faerscope/opencore
```

---

## Quick Start

```typescript
import {
  fetchTrend,
  fetchTopReactions,
  computeDisproportionality,
  applyBenjaminiHochberg,
  getSoc,
  titleCaseTerm,
} from "@faerscope/opencore";

// 1. Fetch trend data from openFDA
const trend = await fetchTrend({
  drug: "metformin",
  dateFrom: "20200101",
  dateTo: "20231231",
});

// 2. Get top adverse reactions
const reactions = await fetchTopReactions(
  { drug: "metformin", dateFrom: "20200101", dateTo: "20231231" },
  20
);

// 3. Compute disproportionality for a reaction
//    (You need to build the 2x2 table from openFDA counts)
const table = { a: 150, b: 4850, c: 3000, d: 992000, n: 1000000 };
const score = computeDisproportionality("NAUSEA", 150, table);

console.log(score.prr);           // 10.0
console.log(score.isSignal);      // true (Evans criteria)
console.log(score.isStrongSignal); // true (ROR CI > 1, IC025 > 0)

// 4. Apply FDR correction across all tested reactions
const corrected = applyBenjaminiHochberg([score, /* ...more scores */]);

// 5. Map reactions to organ classes
console.log(getSoc("NAUSEA"));         // "Gastrointestinal disorders"
console.log(titleCaseTerm("NAUSEA"));  // "Nausea"
```

---

## API Reference

### Modules

| Module | Description |
|---|---|
| [`disproportionality`](#disproportionality-analysis) | PRR, ROR, IC, Chi-squared, BH-FDR, time-series statistics |
| [`openfda`](#openfda-api-client) | Lightweight openFDA fetch utilities |
| [`soc-mapping`](#soc-mapping) | MedDRA Preferred Term to System Organ Class mapping |
| [`manifest`](#session-manifest) | Deterministic SHA-256 hashing and Study ID generation |
| [`utils`](#utilities) | Title-casing, label helpers, date formatting |
| [`types`](#types) | TypeScript type definitions |

---

### Disproportionality Analysis

The core signal detection engine. All functions are **pure** (no side effects, no API calls) and can safely run in Web Workers or server-side.

#### `computeDisproportionality(reaction, count, table)`

The main entry point. Computes all disproportionality metrics for a single drug-reaction pair.

```typescript
import { computeDisproportionality } from "@faerscope/opencore";
import type { ContingencyTable } from "@faerscope/opencore";

const table: ContingencyTable = {
  a: 150,      // drug + reaction
  b: 4850,     // drug + NOT reaction
  c: 3000,     // NOT drug + reaction
  d: 992000,   // NOT drug + NOT reaction
  n: 1000000,  // total
};

const result = computeDisproportionality("NAUSEA", 150, table);
```

**Returns** a `DisproportionalityScore` with:

| Field | Type | Description |
|---|---|---|
| `prr` | `number` | Proportional Reporting Ratio |
| `prrChi2` | `number` | Chi-squared statistic (1 df) |
| `prrPValue` | `number` | P-value from chi-squared test |
| `ror` | `number` | Reporting Odds Ratio |
| `rorLower95` | `number` | ROR 95% CI lower bound (Woolf logit) |
| `rorUpper95` | `number` | ROR 95% CI upper bound |
| `ic` | `number` | Information Component (log2 scale) |
| `ic025` | `number` | IC lower 2.5% credibility bound |
| `ic975` | `number` | IC upper 97.5% credibility bound |
| `isSignal` | `boolean` | Evans criteria: PRR >= 2, Chi2 >= 4, a >= 3 |
| `isStrongSignal` | `boolean` | ROR lower CI > 1 AND IC025 > 0 |

#### Individual Metrics

Each metric is also available as a standalone function:

```typescript
import {
  computePRR,
  computeROR,
  computeROR_CI,
  computeIC,
  computeIC025,
  computeIC975,
  computeChi2,
  chi2PValue1df,
} from "@faerscope/opencore";

const prr = computePRR(table);           // Proportional Reporting Ratio
const ror = computeROR(table);           // Reporting Odds Ratio
const { lower, upper } = computeROR_CI(table); // 95% CI for ROR
const ic = computeIC(table);             // Information Component
const ic025 = computeIC025(table);       // IC lower credibility bound
const ic975 = computeIC975(table);       // IC upper credibility bound
const chi2 = computeChi2(table);         // Chi-squared statistic
const pValue = chi2PValue1df(chi2);      // P-value from chi-squared
```

#### `applyBenjaminiHochberg(scores)`

Apply [Benjamini-Hochberg FDR correction](https://doi.org/10.1111/j.2517-6161.1995.tb02031.x) to control the false discovery rate when testing multiple drug-reaction pairs simultaneously.

```typescript
import { applyBenjaminiHochberg } from "@faerscope/opencore";

const corrected = applyBenjaminiHochberg(scores);

// Filter to FDR-significant signals
const significant = corrected.filter(s => (s.fdrPValue ?? 1) < 0.05);
```

**Algorithm:**
1. Rank p-values from smallest to largest
2. Adjusted p(i) = min(p(i) * m / rank(i), 1)
3. Enforce monotonicity by stepping from largest to smallest rank

#### Time-Series Statistics

```typescript
import {
  detectSpikes,
  detectChangepoints,
  movingAverage,
  rollingZScore,
  yearOverYear,
} from "@faerscope/opencore";
```

##### `detectSpikes(data, window?, threshold?)`

Detect statistical spikes in a trend time series using z-score thresholding. Combines moving average smoothing with rolling z-score computation.

```typescript
const annotated = detectSpikes(trendData, 12, 2.0);
const spikes = annotated.filter(p => p.isSpike);
console.log(`Found ${spikes.length} reporting spikes`);
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `data` | `TrendPoint[]` | required | Time-series data |
| `window` | `number` | `12` | Moving average window size |
| `threshold` | `number` | `2.0` | Z-score threshold for spike flag |

##### `detectChangepoints(data, threshold?)`

[CUSUM](https://en.wikipedia.org/wiki/CUSUM) (Cumulative Sum) changepoint detection. Identifies structural breaks where the mean reporting rate shifts.

```typescript
const counts = trendData.map(d => d.count);
const changepoints = detectChangepoints(counts);
// Returns indices where structural breaks were detected
```

##### `movingAverage(data, window)`

Simple moving average over a numeric array. Returns `undefined` for points before the window is full.

##### `rollingZScore(data, window)`

Rolling centered z-score computation. Useful for outlier detection.

##### `yearOverYear(data)`

Group time-series data by year and align to month indices for year-over-year comparison charts.

```typescript
const yoy = yearOverYear(trendData);
for (const [year, months] of yoy) {
  console.log(`${year}: ${months.length} months of data`);
}
```

---

### openFDA API Client

Lightweight, environment-agnostic fetch utilities for the [openFDA Drug Adverse Event API](https://open.fda.gov/apis/drug/event/).

> **Note:** This module does **not** include rate limiting. The openFDA API allows 40 requests/minute without a key and 240 requests/minute with a key. Your application is responsible for implementing appropriate throttling. See [openFDA Authentication](https://open.fda.gov/apis/authentication/).

#### Configuration

All fetch functions accept an optional `OpenFDAConfig` object:

```typescript
import type { OpenFDAConfig } from "@faerscope/opencore";

const config: OpenFDAConfig = {
  baseUrl: "https://my-proxy.example.com/fda", // Optional: route through a proxy
  apiKey: "your-openfda-api-key",               // Optional: increases rate limit to 240/min
};
```

#### `fetchTrend(params, config?)`

Fetch the time-series trend of adverse event reports for a drug.

```typescript
import { fetchTrend } from "@faerscope/opencore";

const trend = await fetchTrend({
  drug: "metformin",
  dateFrom: "20200101",
  dateTo: "20231231",
  serious: true, // optional: restrict to serious reports
});

// trend[0] = { time: "20200102", count: 42, label: "2020-01-02" }
```

#### `fetchTopReactions(params, limit?, config?)`

Fetch the most frequently reported adverse reactions for a drug.

```typescript
import { fetchTopReactions } from "@faerscope/opencore";

const reactions = await fetchTopReactions(
  { drug: "metformin", dateFrom: "20200101", dateTo: "20231231" },
  20,  // top 20 reactions
);

// reactions[0] = { term: "NAUSEA", count: 1234 }
```

#### `fetchTotalReports(params, config?)`

Get the total number of reports matching a search query.

```typescript
import { fetchTotalReports } from "@faerscope/opencore";

const total = await fetchTotalReports({
  drug: "metformin",
  dateFrom: "20200101",
  dateTo: "20231231",
});
```

#### `buildSearchString(params)`

Build an openFDA search query string from parameters. Useful if you need to construct custom queries.

```typescript
import { buildSearchString } from "@faerscope/opencore";

const search = buildSearchString({
  drug: "metformin",
  dateFrom: "20200101",
  dateTo: "20231231",
  serious: true,
});
// '(patient.drug.openfda.generic_name:"METFORMIN"+patient.drug.openfda.brand_name:"METFORMIN")+AND+receivedate:[20200101+TO+20231231]+AND+serious:1'
```

#### Date Utilities

```typescript
import {
  getDatePreset,
  formatDate,
  formatDateForInput,
  parseDateInput,
} from "@faerscope/opencore";

getDatePreset("3y");           // { from: "20230224", to: "20260224" }
getDatePreset("all");          // { from: "20040101", to: "20260224" }
formatDate(new Date());        // "20260224"
formatDateForInput("20240101"); // "2024-01-01"
parseDateInput("2024-01-01");  // "20240101"
```

---

### SOC Mapping

Maps [MedDRA](https://www.meddra.org/) Preferred Terms to their primary System Organ Class. Covers the ~250 most commonly reported PTs in FAERS (>90% of reactions in typical queries).

> **Limitation:** This is a best-effort mapping based on publicly available MedDRA documentation, not the official MedDRA hierarchy. Some PTs map to multiple SOCs; we assign the primary SOC.

#### `getSoc(pt)`

Look up the SOC for a Preferred Term.

```typescript
import { getSoc } from "@faerscope/opencore";

getSoc("NAUSEA");           // "Gastrointestinal disorders"
getSoc("HEADACHE");         // "Nervous system disorders"
getSoc("RASH");             // "Skin and subcutaneous tissue disorders"
getSoc("UNKNOWN TERM XYZ"); // "Uncategorized"
```

#### `PT_TO_SOC`

The raw mapping object (uppercase PT keys to SOC strings).

```typescript
import { PT_TO_SOC } from "@faerscope/opencore";

console.log(PT_TO_SOC["NAUSEA"]); // "Gastrointestinal disorders"
```

#### `ALL_SOCS`

Sorted array of all unique SOC names in the mapping.

```typescript
import { ALL_SOCS } from "@faerscope/opencore";

console.log(ALL_SOCS);
// ["Blood and lymphatic system disorders", "Cardiac disorders", ...]
```

---

### Session Manifest

Deterministic SHA-256 hashing and Study ID generation for reproducible analyses. Enables any researcher to verify they are examining the same analysis by comparing Study IDs.

#### `generateStudyId(params, comparison?)`

Generate a deterministic Study ID from analysis parameters. The same parameters always produce the same ID.

```typescript
import { generateStudyId } from "@faerscope/opencore";

const { studyId, fullHash } = await generateStudyId({
  drug: "metformin",
  dateFrom: "20200101",
  dateTo: "20231231",
  serious: false,
  dedupMode: "filtered",
  charMode: "suspect",
});

console.log(studyId);  // "FS-a1b2c3d4"
console.log(fullHash); // "a1b2c3d4e5f6..." (64-char hex)
```

#### `generateManifest(input)`

Generate a complete reproducibility manifest with parameters, dataset metadata, results summary, and compliance disclaimers.

```typescript
import { generateManifest } from "@faerscope/opencore";

const manifest = await generateManifest({
  params: searchParams,
  signals: computedScores,
  totalReports: 5000,
  fdrEnabled: true,
});

// manifest.studyId, manifest.disclaimer, manifest.limitations, etc.
```

#### `sha256(message)`

Compute SHA-256 hash using the Web Crypto API. Works in browsers, Node.js 18+, Deno, and Cloudflare Workers.

```typescript
import { sha256 } from "@faerscope/opencore";

const hash = await sha256("hello world");
// "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9"
```

#### `canonicalParamsJSON(params, comparison?)`

Build a deterministic JSON string from analysis parameters (excludes API keys and timestamps).

---

### Utilities

#### `titleCaseTerm(term)`

Convert ALL-CAPS MedDRA terms to title case for display.

```typescript
import { titleCaseTerm } from "@faerscope/opencore";

titleCaseTerm("GASTROOESOPHAGEAL REFLUX DISEASE");
// "Gastrooesophageal Reflux Disease"
```

#### `dedupLabel(mode)` / `charLabel(mode)`

Get human-readable labels for configuration values.

```typescript
import { dedupLabel, charLabel } from "@faerscope/opencore";

dedupLabel("filtered"); // "Duplicate-flagged removed"
charLabel("suspect");   // "Suspect role only"
```

#### `formatDateRange(from, to)`

Format a YYYYMMDD date range as a readable string.

```typescript
import { formatDateRange } from "@faerscope/opencore";

formatDateRange("20200101", "20231231"); // "2020-01-01 to 2023-12-31"
```

---

### Types

All types are exported from the package root:

```typescript
import type {
  // Search & configuration
  DedupMode,
  CharacterizationMode,
  SearchParams,
  StudioSearchParams,

  // Data structures
  TrendPoint,
  TrendWithStats,
  ReactionCount,
  ProductCount,

  // Disproportionality
  ContingencyTable,
  DisproportionalityScore,

  // Manifest
  SessionManifest,
  ComparisonConfig,
  ManifestInput,

  // openFDA
  OpenFDAConfig,
  OpenFDAMeta,
  OpenFDACountResult,
  OpenFDAResponse,
} from "@faerscope/opencore";
```

---

## Building a Complete Signal Detection Pipeline

Here's how you might use this library to build a full analysis:

```typescript
import {
  fetchTopReactions,
  fetchTotalReports,
  buildSearchString,
  computeDisproportionality,
  applyBenjaminiHochberg,
  getSoc,
  titleCaseTerm,
  generateManifest,
} from "@faerscope/opencore";
import type { ContingencyTable, StudioSearchParams } from "@faerscope/opencore";

const params: StudioSearchParams = {
  drug: "metformin",
  dateFrom: "20200101",
  dateTo: "20231231",
  serious: false,
  dedupMode: "filtered",
  charMode: "suspect",
};

// Step 1: Get total reports for the drug and globally
const drugTotal = await fetchTotalReports(params);
const globalN = 1000000; // You'd fetch this from openFDA too

// Step 2: Get top reactions
const reactions = await fetchTopReactions(params, 50);

// Step 3: For each reaction, build a contingency table and compute scores
// (In practice, you'd fetch 'a' and 'a+c' from openFDA for each reaction)
const scores = reactions.map(r => {
  const table: ContingencyTable = {
    a: r.count,
    b: drugTotal - r.count,
    c: 500,  // reaction total across all drugs (fetched from openFDA)
    d: globalN - drugTotal - 500 + r.count,
    n: globalN,
  };
  return computeDisproportionality(r.term, r.count, table);
});

// Step 4: Apply FDR correction
const corrected = applyBenjaminiHochberg(scores);

// Step 5: Filter to significant signals
const signals = corrected.filter(s => s.isSignal);
const strongSignals = corrected.filter(s => s.isStrongSignal);

// Step 6: Group by organ class
const bySOC = new Map<string, typeof signals>();
for (const s of signals) {
  const soc = getSoc(s.reaction);
  if (!bySOC.has(soc)) bySOC.set(soc, []);
  bySOC.get(soc)!.push(s);
}

// Step 7: Generate reproducibility manifest
const manifest = await generateManifest({
  params,
  signals: corrected,
  totalReports: drugTotal,
  fdrEnabled: true,
});

console.log(`Study ID: ${manifest.studyId}`);
console.log(`${signals.length} signals detected (${strongSignals.length} strong)`);
```

---

## Statistical Background

### Disproportionality Methods

| Metric | Formula | Signal Threshold | Reference |
|---|---|---|---|
| **PRR** | (a/(a+b)) / (c/(c+d)) | >= 2 (Evans) | [Evans et al., 2001](https://doi.org/10.1002/pds.677) |
| **ROR** | (a*d) / (b*c) | Lower 95% CI > 1 | [Rothman et al., 2004](https://doi.org/10.1002/pds.905) |
| **IC** | log2(observed / expected) | IC025 > 0 | [Bate et al., 1998](https://doi.org/10.1007/BF03256158) |
| **Chi-squared** | (ad-bc)^2*N / ((a+b)(c+d)(a+c)(b+d)) | >= 4 (Evans) | Standard 2x2 test |

### Evans Signal Criteria

A drug-reaction pair is classified as a **signal** when all three conditions are met:
1. PRR >= 2
2. Chi-squared >= 4
3. Case count (cell `a`) >= 3

### Strong Signal Criteria

A signal is classified as **strong** when both of these are met:
1. ROR lower 95% CI > 1 (frequentist evidence)
2. IC025 > 0 (Bayesian evidence)

### Benjamini-Hochberg FDR

When testing many reactions simultaneously, the BH procedure controls the **expected proportion of false discoveries** among rejected hypotheses. An FDR threshold of 0.05 means you expect no more than 5% of your "signals" to be false positives.

---

## Limitations

This library and the underlying FAERS data have important limitations:

1. **FAERS data are voluntary spontaneous reports** subject to under-reporting, stimulated reporting, and duplicate submissions.
2. **Deduplication at the API level is approximate** — true case-level dedup requires raw data access.
3. **Disproportionality metrics assume independence** between drug-reaction pairs, which may not hold.
4. **Small cell counts (<5) produce unstable estimates** with wide confidence intervals.
5. **openFDA aggregation endpoints return approximate counts** that may differ from raw FAERS data.
6. **No denominator data** (prescriptions dispensed) is available — cannot compute incidence rates.
7. **Reporter qualification and country of origin** introduce heterogeneity in report quality.
8. **MedDRA coding variations** may split or merge related reactions.
9. **Temporal trends may reflect changes in reporting behavior** rather than true safety signals.
10. **This is screening-level analysis** — any finding requires formal pharmacovigilance review.

---

## Environment Support

| Environment | Supported | Notes |
|---|---|---|
| Node.js 18+ | Yes | Uses `crypto.subtle` (available since Node 15, stable in 18+) |
| Modern Browsers | Yes | All modern browsers support Web Crypto and Fetch |
| Deno | Yes | Native Web Crypto and Fetch support |
| Cloudflare Workers | Yes | Web Crypto and Fetch available |
| Bun | Yes | Full Web API compatibility |

---

## Related Projects

- **[FAERScope Studio](https://faerscope.org/studio)** — Full research workbench with advanced features (NMF, E-values, CUSUM, co-medication networks, auth, session persistence).
- **[openFDA](https://open.fda.gov)** — The FDA's open API providing access to FAERS and other datasets.
- **[MedDRA](https://www.meddra.org/)** — Medical Dictionary for Regulatory Activities.

---

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Key areas where help is needed:
- Expanding the SOC mapping beyond the current ~250 PTs
- Adding unit tests for edge cases in disproportionality calculations
- Performance benchmarks across different runtimes
- Documentation improvements and tutorials

---

## License

[Apache-2.0](LICENSE) -- Copyright 2024-2026 FAERScope Contributors
