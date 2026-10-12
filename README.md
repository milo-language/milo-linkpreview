# linkpreview

This is a package for the [Milo language](https://milo-language.github.io/milo/).

## Overview

Turn a URL into a link card: title, description, site name, image with its
dimensions, favicon, and what kind of page it is.

```milo
let p = fetchPreview("https://github.com/pypa/pipx", PreviewOptions.defaults())!
print(p.title ?? p.url)
```

`fetchPreview` is built to be pointed at URLs your users chose. It fetches only
public http(s) addresses, re-checks every redirect hop, pins each connection to
the address it checked, and reads at most 256 KB. `parseHtml` is the same
extraction with no network, for HTML you already have.

## Installation

```bash
milo pkg add github.com/milo-language/milo-linkpreview            # latest release
milo pkg add github.com/milo-language/milo-linkpreview@v0.1.0     # or pin a tag
```

```milo
from "linkpreview" import { fetchPreview, parseHtml, Preview, PreviewOptions, PreviewError }
```

Needs a Milo whose `std/fetch` has `fetchAt` (main from 2026-10-12 on).

## API

```milo
@derive(Json)
pub struct Preview {
    url: string,            // the final URL, after redirects
    canonicalUrl: string?,  // <link rel=canonical>, else og:url
    title: string?,
    description: string?,
    siteName: string?,
    image: string?,
    imageWidth: i64?,
    imageHeight: i64?,
    imageAlt: string?,
    kind: string?,          // article, video, music, book, profile, website, repo, paper, image
    favicon: string?,
    oembedUrl: string?,     // the page's oEmbed endpoint, or a known provider's
    oembedHtml: string?,    // the provider's embed markup: untrusted HTML
}

pub fn parseHtml(baseUrl: &string, html: &string): Preview
pub fn fetchPreview(url: &string, opts: &PreviewOptions): Result<Preview, PreviewError>
pub fn mergeOembed(p: &mut Preview, oembedJson: &string)

pub fn isPublicIp4(ip: u32): bool          // std/net byte order
pub fn isPublicIp6(addr: &[u8; 16]): bool
pub fn isPublicAddress(text: &string): bool  // "10.0.0.1", "::1", "[fe80::1]"
pub fn decodeEntities(s: &string): string
```

Every URL in a `Preview` is absolute http(s); every text field is decoded,
whitespace-collapsed, valid UTF-8, and capped (title 300 bytes, description 600,
with an ellipsis where it was cut). `Preview` derives `Json`, so
`p.toJson()` / `Preview.fromJson(s)` store and reload it.

### Where each field comes from

First source that has it wins:

| Field | Sources, in order |
|---|---|
| title | `og:title`, `twitter:title`, JSON-LD `headline`/`name`, `<title>` |
| description | `og:description`, `twitter:description`, JSON-LD `description`, `<meta name=description>` |
| siteName | `og:site_name`, `application-name`, JSON-LD `publisher.name`, then oEmbed `provider_name` |
| image | `og:image` (with `og:image:width/height/alt`), `twitter:image`, JSON-LD `image`/`thumbnailUrl`, `<link rel=image_src>`, the first content-sized `<img>`, then oEmbed `thumbnail_url` |
| favicon | `<link rel=icon>` (32px or larger preferred), `apple-touch-icon`, else `/favicon.ico` |
| kind | `citation_*` tags or arxiv.org/abs and huggingface.co/papers: `paper`; github.com/owner/repo: `repo`; else `og:type` / JSON-LD `@type`, folded (`video.other` is `video`) |

Relative URLs resolve against the final URL after redirects, or the page's
`<base href>`. Both `property=` and `name=` spellings of `og:*` are read. The
fallback `<img>` skips inline `data:` images, anything named like a logo, icon,
avatar or tracking pixel, SVG and GIF, and anything declared under 120 px.

### Options

```milo
var opts = PreviewOptions.defaults()
opts.userAgent = "mybot/1.0 (+https://example.com/bot)"
opts.fetchOembed = true      // one more request, through the same checks
```

| Field | Default | |
|---|---|---|
| `userAgent` | `milo-linkpreview/0.1` | sent on every request |
| `maxBytes` | 262144 | body bytes read; parsing sees only these |
| `connectMs` | 5000 | each TCP connect plus TLS handshake |
| `totalMs` | 10000 | the whole call: every hop and the oEmbed request |
| `maxRedirects` | 5 | 301, 302, 303, 307, 308 |
| `fetchOembed` | false | GET the oEmbed endpoint and fill gaps from it |
| `allowAnyPort` | false | ports other than 80/443 |
| `allowLoopback` | false | 127.0.0.0/8, for tests and local development only |

Errors: `BadUrl`, `Blocked`, `Network`, `Timeout`, `HttpStatus(code)`,
`NotHtml(contentType)`, `TooManyRedirects`; `e.message()` reads well in a log. A
URL that serves an image previews as itself (`kind: "image"`).

## Safety

`fetchPreview` assumes the URL is hostile.

* **Only http and https**, with no credentials in the authority
  (`http://trusted@10.0.0.1/` is refused, not parsed), an ASCII host (punycode
  for IDN), and the scheme's default port unless `allowAnyPort`.
* **Address check after resolution.** The host is resolved and the address must
  be public: no 0/8, 10/8, 100.64/10 (CGNAT), 127/8, 169.254/16 (link-local,
  including cloud metadata at 169.254.169.254), 172.16/12, 192.0.0/24,
  192.0.2/24, 192.168/16, 198.18/15, 198.51.100/24, 203.0.113/24, 224/4
  (multicast), 240/4 or broadcast. Checking the resolved address rather than
  the spelling covers `localhost`, `2130706433`, `0x7f000001` and DNS names that
  point inward. IPv6 literals are refused when they are `::`, `::1`, fc00::/7,
  fe80::/10, fec0::/10, ff00::/8, 2001:db8::/32, or an IPv4-mapped,
  IPv4-compatible or NAT64 (64:ff9b::/96) form of a refused IPv4 address.
* **No DNS-rebinding window.** Each hop is resolved once, and the connection goes
  to exactly that address (std/fetch `fetchAt`), with the original host name
  kept for the Host header and for TLS certificate verification. A resolver that
  answers "public" to the check and "private" to the connect gets no second
  question.
* **Every redirect is a new URL** and goes through all of the above before
  anything is sent to it, up to `maxRedirects`.
* **Bounded.** At most `maxBytes` of body is read and buffered, the whole call
  is bounded by `totalMs`, and only `text/html` / `application/xhtml+xml` is
  parsed.
* **The oEmbed endpoint is untrusted too**: it came from the page, so it is
  fetched through the same checks, and `oembedHtml` is the provider's markup as
  sent. Sanitize it or render it in a sandboxed iframe; never insert it as is.
* **Image and favicon URLs are not fetched** by this package. If you proxy or
  download them server side, they need the same checks: `isPublicAddress` and
  a pinned connection, or run them through your own copy of this flow.

Residual risk, stated plainly:

* Resolution is IPv4 only (std/net `resolve` asks for A records), so a host with
  only AAAA records fails with `Network` rather than being fetched. Nothing is
  exposed by this, but such sites get no preview.
* The first A record is the one used. A name with one public and one private
  record is fetched from whichever the resolver lists first, and that address
  is checked, so this cannot reach a private address; it can only fail.
* DNS lookups themselves go out for any name a user submits. They leak the name
  to your resolver, nothing more.
* `allowLoopback` and `allowAnyPort` exist for tests. Turning either on in a
  server that fetches user URLs gives those users the server's own ports.

## Notes

* **Pages with a huge `<head>`.** YouTube's watch page puts its `<head>` some
  770 KB in, past any sane cap, so `parseHtml` sees none of it. For YouTube and
  Vimeo URLs `oembedUrl` is filled from the URL alone, and with `fetchOembed`
  the title and thumbnail come from there.
* **Character sets.** UTF-8 is assumed; `windows-1252` and Latin-1 labels (from
  the header or a `<meta charset>`) are transcoded. Anything else passes through
  with invalid sequences replaced by U+FFFD, so the output is always valid UTF-8.
* **Entities.** Numeric references and the ~90 named ones that show up in
  titles (`&mdash;`, `&rsquo;`, `&eacute;`, ...) decode. A name not in the table,
  or one missing its `;`, is left as written, which is what keeps `&image=` in
  a query string intact.

## Examples

```bash
milo run examples/preview.milo https://www.youtube.com/watch?v=dQw4w9WgXcQ
milo run examples/dump.milo https://arxiv.org/abs/1706.03762 tests/pages/arxiv-abs.html
```

```milo
from "linkpreview" import { parseHtml }

fn main(): i32 {
    let html = "<head><title>Fallback</title><meta property=\"og:title\" content=\"Caf&eacute; &amp; Bar\"><meta property=\"og:image\" content=\"/card.png\"></head>"
    let p = parseHtml("https://example.com/menu/", html)
    print(p.title ?? "")   // Café & Bar
    print(p.image ?? "")   // https://example.com/card.png
    return 0
}
```

## Tests

```bash
milo test tests/linkpreview_test.milo      # no network: saved pages + 127.0.0.1 servers
milo run --release scripts/fuzz.milo 5000  # mutation fuzzing of parseHtml
```

The parser is tested on real captures (`tests/pages/`: a Phoronix article, a
GitHub repo, a YouTube watch page, a Hugging Face paper, an arXiv abstract) and
on synthetic edge cases; the fetcher on every refused range, redirects into
private space, redirect loops, a body that never ends, a server that never
answers, wrong content types and Latin-1 pages.
