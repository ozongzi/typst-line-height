<h1 align="center">
  <img alt="Typst" src="https://user-images.githubusercontent.com/17899797/226108480-722b770e-6313-40d7-84f2-26bebb55a281.png">
</h1>

<p align="center">
  <b>A vendored fork of <a href="https://github.com/typst/typst">Typst</a> tuned for absolute typographic quality.</b>
</p>

<p align="center">
  <a href="https://github.com/typst/typst"><img alt="Upstream" src="https://img.shields.io/badge/upstream-typst%2Ftypst-007aff"></a>
  <img alt="Base" src="https://img.shields.io/badge/base-v0.15.0-555">
  <a href="./LICENSE"><img alt="Apache-2 License" src="https://img.shields.io/badge/license-Apache%202-brightgreen"></a>
</p>

> [!IMPORTANT]
> This is **not** upstream Typst. It is a private/vendored fork that
> intentionally diverges from `typst/typst` in the paragraph model. Documents
> compiled here **will not render identically** to stock Typst, and some
> upstream set-rule options (`par.leading`, `par.linebreaks`) no longer exist.
> Use stock Typst if you need upstream-compatible output.

For everything the fork does *not* change — the language, the standard library,
math, packages, the CLI — refer to the upstream [documentation][docs] and
[tutorial][tutorial]. This README only documents where this fork deviates.

## Why this fork exists

Stock Typst makes a few pragmatic trade-offs in the paragraph engine. This fork
replaces them with the models used by professional typesetting systems
(InDesign, TeX), accepting more breaking changes and slower ragged-text layout
in exchange for more predictable spacing and higher line-breaking quality. The
guiding principle here is **absolute quality over compile speed**.

## Deviations from upstream

### 1. `par.line-height` replaces `par.leading`

Upstream `leading` measures the *edge-to-edge gap* between the bottom of one
line and the top of the next. When lines differ in height (inline equations,
mixed sizes, CJK/Latin runs), the baseline-to-baseline distance drifts, and the
only workaround — pinning `top-edge`/`bottom-edge` — holds only within a single
font size.

This fork removes `leading` entirely and adds **`line-height`**, a
**baseline-to-baseline** distance (the InDesign/CSS/`\baselineskip` model). The
gap inserted between two lines is

```
gap = max(0, line-height − depth(upper line) − height(lower line))
```

so baselines land exactly `line-height` apart, clamping to zero only to avoid
overlap. Verified: `#set par(line-height: 22pt)` yields baselines exactly 22 pt
apart across CJK/Latin mixing, inline math, and size changes.

- Default `line-height` is `1.3em` (≈ the old default look for the default
  font). Footnotes use `1.15em`.
- Math row spacing is **not** driven by `line-height` (equation rows want an
  edge gap, not a baseline distance) and uses its own constant.
- **Breaking:** `#set par(leading: …)` is now an unknown-argument error.

### 2. Line breaking is always Knuth–Plass

Upstream falls back to greedy first-fit for ragged (non-justified) paragraphs
and only runs the optimized breaker when justifying. This fork **removes the
greedy first-fit path** (and the `par.linebreaks` / `Linebreaks` option): every
paragraph is broken with the optimized total-fit algorithm, ragged text
included, for evenly balanced lines everywhere.

- **Breaking:** `#set par(linebreaks: "simple" | "optimized")` no longer exists.
- **Cost:** ragged-text layout is **~2.4× slower** than upstream's greedy path
  (measured on ~28k words). This is an accepted trade-off for quality; there is
  deliberately no greedy escape hatch.

### 3. Authoritative Knuth–Plass cost model

The optimized breaker's cost function is aligned with Knuth's original
implementation (`tex.web`) rather than the simplified form upstream uses:

- Demerits are `(line_penalty + badness)²` with penalties added **outside** the
  square (`±π²`, flagged/`double_hyphen_demerits`), not folded inside it.
- **Fitness classes** (very-loose / loose / decent / tight) and the
  adjacent-fitness demerit (`adj_demerits`) are implemented, with the dynamic
  program keeping the best entry **per fitness class** at each breakpoint.

Validated by differential testing against real `tex.web` (pdftex) on controlled
monospace corpora: in the regime where the two cost models coincide, break
positions agree **~97%** (remaining differences are float-vs-integer badness
near-ties). The per-fitness DP adds negligible cost (≈ +10% on justified
layout); the dominant cost is deviation #2 above.

> Note: fitness classes rarely change breaks for normal-looseness text (adjacent
> lines are usually all "decent"). They are included for faithfulness, not
> because they visibly improve typical documents.

## Building

Requires the [latest stable Rust][rust].

```sh
cargo build --release        # binary in target/release/typst
```

Usage is identical to upstream Typst (`typst compile file.typ`, `typst watch`,
etc.); see the upstream [docs][docs].

## Maintaining this fork

- **Base:** forked from `typst/typst` at **v0.15.0**. Rebasing onto a newer
  upstream will conflict primarily in `crates/typst-library/src/model/par.rs`
  and `crates/typst-layout/src/inline/`.
- **Reference tests:** because line spacing and ragged breaking change for
  essentially every multi-line document, the image references under
  `tests/ref/**` are regenerated for this fork's defaults. After any change that
  affects layout, rerun and update:
  ```sh
  cargo test --workspace                 # check
  cargo test --workspace --test tests -- --update   # regenerate references
  ```
- Keep the whole suite green before committing; failures should only ever be
  intended layout shifts, never panics or errors.

## License & attribution

This fork is distributed under the same **Apache-2.0** license as upstream
Typst; see [`LICENSE`](./LICENSE). Typst, its logo, and the original source are
© the Typst authors. All credit for the compiler belongs upstream; only the
deviations listed above are changed here.

[docs]: https://typst.app/docs/
[tutorial]: https://typst.app/docs/tutorial/
[rust]: https://rustup.rs/
