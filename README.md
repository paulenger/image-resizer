# Resize & pad

Client-side image resizer, padder and file-type converter. One static `index.html`, no build step, deployed on Vercel.

Fork of [paulenger/image-resizer](https://github.com/paulenger/image-resizer) adding a **Conversion** tab for bulk file-type changes.

Three tabs:

| Tab | What it does |
| --- | --- |
| One image | Resize and pad a single file, with a crop box. |
| Batch | Resize and pad many files to one output size. |
| Conversion | Change the file type of many files, leaving pixel sizes alone. |

## Running it

There is nothing to install for the front end. Serve the directory:

```bash
python3 -m http.server 8000
```

Then open <http://localhost:8000/index.html>. Every image is decoded, drawn and encoded in the browser; nothing is uploaded.

`api/save-to-drive.js` is a Vercel serverless function that needs `GOOGLE_SERVICE_ACCOUNT_KEY` and the `googleapis` dependency (`npm install`). Nothing in `index.html` calls it today — it is left over from the Apps Script migration.

## The Conversion tab

Batch mode plus **Save as** already converted file types, but every image was drawn onto a target canvas first, so you could not ask for "these 200 PNGs as JPEG, sizes untouched".

The **Conversion** tab drops the output size entirely. The canvas becomes each image's own size, which retires width, height, fit mode, padding and cropping, so section 02 is the file type and there is no section 03. Drop a mixed folder, pick a type, get a ZIP.

It reuses the batch queue, so converting a single file is just a queue of one.

Behaviour worth knowing:

- **Files already in the target type are passed through untouched** (toggle under the file type). Re-encoding a JPEG as a JPEG only throws away quality, so those files are copied into the ZIP as they arrived, without being decoded at all.
- **Encoders are probed at startup.** `canvas.toBlob` silently falls back to PNG for a type the browser cannot encode, which would put PNG bytes inside a `.avif` file. Unsupported types are disabled in the dropdown instead.
- **JPEG has no alpha**, so transparent pixels are flattened onto a color you choose. That is the only padding-style control the tab keeps, and it only appears when JPEG is selected. Everything else keeps transparency.
- **GIFs become a still frame**, since the canvas only ever holds one.
- **Output names are de-duplicated.** Converting collapses every input extension onto one, so `photo.png` and `photo.jpg` both want to be `photo.jpg`; the second becomes `photo-2.jpg` rather than overwriting the first inside the ZIP.
- Results are held as blobs rather than data URLs. Base64 is about a third larger than the bytes it encodes, and the previous version kept one string per image in memory until the ZIP was built.

The One image and Batch tabs are unchanged.

## Formats

| | Input | Output |
| --- | --- | --- |
| PNG | yes | yes, lossless, keeps alpha |
| JPEG | yes | yes, quality slider, no alpha |
| WebP | yes | yes, quality slider, keeps alpha |
| AVIF | yes | where the browser can encode it |
| GIF | first frame | no |
