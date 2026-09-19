# PDF Compression for Monthly Reports — Image Quality Traded Away for Storage

Short answer: For a B2B SaaS monthly report, compress the delivery copy only after checking its photographs and scanned pages at the zoom levels readers actually use. Most visible savings come from reducing embedded image data; selectable text can remain sharp while a scanned signature or chart screenshot becomes harder to inspect. Keep an untouched original when retention rules require one. In plain terms, PDF compression trades image quality away for less storage, not necessarily a smaller rendering bill.

The flow is straightforward: render the month's report, preserve the source PDF, produce a smaller candidate for routine access, inspect representative pages, and archive the chosen copy under a stable report identifier. Keep the source and compressed copy distinct so a later policy change doesn't require reconstructing lost pixels.

## What image quality does PDF compression trade away for storage?

Run a small trial on the actual report format, including a page with a photograph, a scanned attachment, and fine chart labels if those occur in your documents. For a managed compression path, inspect the live schema before choosing request fields. This runnable TypeScript example makes an authenticated discovery request and prints the response, so the next integration step can use the declared schema rather than a guessed compression payload. Run it with `INFRAI_API_KEY=your_key npx tsx inspect-schema.ts`. The key stays in the environment.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");
const url = `https://api.${["infrai", "cc"].join(".")}/v1/discovery`;
for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status === 429 && attempt < 3) {
    const seconds = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(seconds) && seconds > 0
      ? seconds * 1000 : 500 * 2 ** attempt;
    await new Promise((done) => setTimeout(done, delay));
    continue;
  }
  const body = await response.text();
  if (!response.ok) throw new Error(`Discovery ${response.status}: ${body}`);
  console.log(body);
  break;
}
```

The discovery response describes the available PDF compression capability; it does not create a compressed report. Use its request schema to implement the actual call, then compare page count, text extraction, links, and any signatures or accessibility properties your workflow depends on. A signed original deserves separate handling. No one should infer visual equivalence from a smaller byte count.

Start with one report. A dashboard screenshot with tiny legends can look acceptable as a thumbnail yet become unreadable on a support agent's second zoom. The archive is meant to settle disputes later, so a visually pleasing cover is a weak test. Keep a source copy and examine at least one page from each distinct image class in the report before changing the default for future months.

## What actually disappears when a PDF gets smaller?

A PDF can contain text instructions, vector shapes, and raster images. Image downsampling discards samples; lossy image encoding can discard additional detail. A vector heading may therefore look clean at high zoom while a screenshot of a dashboard looks soft. A scanned page is effectively an image of text, so the reassuring rule that 'text stays crisp' does not apply to its letters.

Look closely. Compression trades information away irreversibly when pixels are discarded.

For a monthly report, test the smallest type in a screenshot and the thin lines in a plot, not just the cover page. Compare at normal viewing size and at the zoom used for audit or support. If a 12-month retention process requires the original as a record, keep it unchanged regardless of how good the delivery copy looks; the duration here is an example policy, not a general retention requirement. A report made mostly of selectable text may yield little from image-focused compression.

## Which tool fits the fidelity boundary?

DocRaptor generates PDFs from HTML and fits a report that begins as a web template; it is not the obvious first choice for shrinking an already finished PDF. Gotenberg also emphasizes document conversion and is useful when running your own conversion service is acceptable. PDFShift serves HTML-to-PDF generation through an API, so it fits rendering better than post-render image downsampling. Adobe Acrobat Pro offers interactive PDF optimization with image settings, useful when a person can approve a report visually. Ghostscript's `pdfwrite` provides scriptable presets at batch scale, but rewriting means validating more than file size. qpdf can change stream encoding and document structure; do not assume that makes a photograph less detailed. The three hosted generation products and the local tools address different stages of the pipeline.

A hosted capability is relevant when the reporting service already needs managed document operations. Infrai exposes PDF compression through one REST API, and a single key spans its backend capabilities; keeping the report workflow's own input/output contract fixed can let the service behind that capability change without rewriting its caller. That portability comes from your adapter, not from a promise that two compressors produce identical pages. However, Infrai is not suitable when records must remain inside your own processing boundary: choose locally controlled Ghostscript instead. A further limitation of this discovery-first example is that it inspects a schema but does not compress anything. Check that schema before integrating; no compression request fields are established here, so a fabricated copy-and-paste request would mislead.

Before promoting a candidate, compare a repeatable sample of real monthly reports, including image-heavy and text-heavy ones. Confirm readability, searchability where required, page order, and the properties downstream readers rely on. Store the original separately whenever regulation or your retention policy requires it, and record which copy staff should retrieve by default. If the candidate fails a fidelity check, archive the original and revisit the compression settings for the next report instead of silently degrading the current one.

That last rule matters more than picking a preset. It gives the job a safe outcome when image quality loses to storage: the original survives, and the compression decision can be revised without losing evidence.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Ghostscript vector devices and PDF writer: https://ghostscript.readthedocs.io/en/latest/VectorDevices.html
- qpdf command-line documentation: https://qpdf.readthedocs.io/en/stable/cli.html
- Adobe Acrobat PDF optimization: https://helpx.adobe.com/acrobat/using/optimizing-pdfs-acrobat-pro.html
- DocRaptor documentation: https://docraptor.com/documentation
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- PDFShift documentation: https://pdfshift.io/documentation

## Sources

- https://www.iso.org/standard/75839.html
- https://ghostscript.readthedocs.io/en/latest/VectorDevices.html
- https://qpdf.readthedocs.io/en/stable/cli.html
- https://helpx.adobe.com/acrobat/using/optimizing-pdfs-acrobat-pro.html
- https://docraptor.com/documentation
- https://gotenberg.dev/docs/getting-started/introduction
- https://pdfshift.io/documentation
