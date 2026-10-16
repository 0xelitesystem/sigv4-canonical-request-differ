# SigV4 Canonical Request Differ

Paste the canonical request and string to sign from an AWS SignatureDoesNotMatch error next to the ones your code built; the page aligns them line by line and names the documented SigV4 rule each mismatched line breaks. Offline, and it never asks for a secret key.

**Live demo:** https://0xelitesystem.github.io/sigv4-canonical-request-differ/

## Use

1. In the left box, paste what AWS sent back. Three shapes are read: an S3 style XML error body with `<CanonicalRequest>` and `<StringToSign>` elements, the "The canonical string for this request should have been '...'" message that Query style APIs return, or a bare canonical request. XML entities such as `&amp;` are decoded and the page says so.
2. In the right box, paste the canonical request your signer built, optionally followed by its string to sign.
3. Pick the service family: `s3`, `s3express` or `other`. It decides which path encoding is expected.
4. Read the result. The string to sign is compared first, because a different credential scope or date means a different request, not an encoding bug. Then each side's string to sign is checked against the SHA-256 of its own canonical request, and the canonical requests are lined up part by part: method, URI, query, each header by name, SignedHeaders and the payload hash. Every differing row names a rule (K1 to K16, listed on the page) or says "differs; no catalogued rule matched". It never guesses.

There is no secret key field and there never will be. The only cryptography on the page is SHA-256 through the browser's WebCrypto API, used to check each string to sign against its canonical request. Nothing is signed and no signing key is derived.

The page opens on example P1: an S3 object key `photos/my file+v1.txt` that AWS canonicalised once as `/photos/my%20file%2Bv1.txt` while the signer encoded it twice as `/photos/my%2520file%252Bv1.txt`, the non-S3 behaviour. Seven more examples cover a `+` for a space, lowercase hex escapes, an unsorted query, a multi-value header joined with a space, a signed header that arrived empty, an empty body hash against a real one, and the Query API message format.

### When this is useful

- An S3 or API Gateway call fails with SignatureDoesNotMatch and the error body includes AWS's canonical request, but reading two long blocks by eye has not found the difference.
- A request signs fine from a laptop and fails behind a corporate proxy, a service mesh or a CI runner, and you suspect something on the path changes a signed header.
- You are writing or porting a SigV4 signer and want each canonicalisation rule checked against a known-good request.
- A teammate needs to see which documented rule the signer breaks, with the AWS wording next to it.

## The rules

Each rule below is quoted from the IAM User Guide page [Create a signed AWS API request](https://docs.aws.amazon.com/IAM/latest/UserGuide/create-signed-request.html) unless marked otherwise.

- **K1** "The space character is a reserved character and must be encoded as "%20" (and not as "+")."
- **K2** "Letters in the hexadecimal value must be uppercase, for example "%1A"."
- **K3** An unreserved character (A to Z, a to z, 0 to 9, `-`, `.`, `_`, `~`) was percent-encoded, or a reserved one was left raw.
- **K4** "You must also sort the parameters in the canonical query string alphabetically by key name. The sorting occurs after encoding." When keys mix upper and lower case, the page compares byte values and says so.
- **K5** A parameter without a value still needs its `=`; the IAM page's example is `UriEncode("acl") + "=" + ""`.
- **K6** Header names must be lowercase and the headers sorted by name.
- **K7** Header values: "trim any leading or trailing spaces. convert sequential spaces to a single space."
- **K8** "separate the values for a multi-value header using commas."
- **K9** SignedHeaders must list exactly the canonical header names, and both sides must sign the same set.
- **K10** A signed header has a value on one side and is empty on the other. The [troubleshooting page](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv-troubleshooting.html) says: "You can also verify that you didn't send the request through a proxy that modifies the headers or the request."
- **K11** "If there is no payload in the request, you compute a hash of the empty string". That hash is `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`, recomputed live by the self test. "For Amazon S3, include the literal string UNSIGNED-PAYLOAD when constructing a canonical request, and set the same value as the x-amz-content-sha256 header value when sending the request."
- **K12** The path is encoded once on one side and twice on the other (`%25XX` against `%XX`), or normalised on one side only. This rule is sourced from the AWS SDK for JavaScript signer's documentation in [smithy-typescript `SignatureV4Base.ts`](https://github.com/smithy-lang/smithy-typescript/blob/main/packages/signature-v4/src/SignatureV4Base.ts), not from AWS service documentation: path escaping "is required for every AWS service, except Amazon S3, as of late 2017", and "Non-S3 services, we normalize the path and then double URI encode it." For S3, "we shouldn't normalize the path".
- **K13** "Encode the forward slash character, '/', everywhere except in the object key name."
- **K14** A volatile header was signed. The IAM page lists headers not to sign, "including connection , x-amzn-trace-id , user-agent , keep-alive , transfer-encoding , TE , trailer , upgrade , proxy-authorization , and proxy-authenticate". Reported as a warning.
- **K15** A required header is not signed. The IAM page: "You must include the host header (HTTP/1.1) or the :authority header (HTTP/2), and any x-amz-* headers in the signature." and "The x-amz-content-sha256 header is required for Amazon S3 AWS requests."
- **K16** The last line of a string to sign is the SHA-256 hex of that side's canonical request. A mismatch on your side means your signer hashed something other than what you pasted; a mismatch on the AWS side usually means the paste was altered.

The troubleshooting page's own advice is the reason this page exists: "Compare the canonical request and string to your signed request with the value in the error message. Modify the signing process if there are any differences."

Also handled: CRLF line endings and a trailing newline (normalised and reported), a missing empty query line, header values that contain `:`, presigned URLs (`X-Amz-Signature` must not be in the canonical query string), and SigV4a (`AWS4-ECDSA-P256-SHA256`), which is recognised but not checked because its credential scope rule differs.

S3 error bodies commonly include `<CanonicalRequest>` and `<StringToSign>`; the element names are attested in public S3 error output such as the one pasted in aws/aws-sdk-js-v3#3277, which begins `<Code>SignatureDoesNotMatch</Code><Message>The request signature we calculated does not match the signature you provided. Check your key and signing method.</Message>`. Not every AWS service returns the canonical request. The "should have been" message format follows the example on the AWS re:Post Knowledge Center article [api-gateway-iam-sigv4-mismatch](https://repost.aws/knowledge-center/api-gateway-iam-sigv4-mismatch).

## Real cases these rules match

- curl/curl#23120, "aws-sigv4: S3 Express (s3express) paths get encoded twice" (K12).
- anthropics/anthropic-sdk-csharp#268, where the signer joined a repeated header with a space "while `HttpClient` put `anthropic-beta: beta-one, beta-two` on the wire and the service recomputed from that." (K8).
- openai/codex#50246, where "S3's error response showed an empty value for that header in its `CanonicalRequest`." The report adds: "We have not isolated the exact proxy component responsible." (K10).

## Self test

The page runs its self test on load and prints a live pass and fail count: one fixture per rule K1 to K16, the eight examples (each built with its string to sign hashed live, so both K16 checks must pass), the empty-string hash recomputed with WebCrypto, and a fuzz test that builds random canonical requests from a fixed seed, mutates each one by exactly one catalogued rule and requires the differ to name that rule. The last row is a positive control written to fail.

## Why this exists

The client signs what it believes it sent; AWS signs what arrived. A proxy that drops a header, an HTTP stack that joins repeated headers differently, or the S3 versus non-S3 path rule makes both computations correct for their own input. The error body hands you AWS's side; this page puts it beside yours and names the rule.

Single file, no build step, no network call, no tracking, MIT licensed. It never asks for a secret key.

## Privacy

Everything runs in your browser. The page makes no network call of any kind: no code, font, analytics or telemetry request, and nothing you paste is transmitted. A Content-Security-Policy in the page forbids network connections. There is no key field; the only computation on your text is SHA-256. Even so, redact access key IDs and session tokens before pasting request material anywhere, and the page warns when a paste contains something shaped like an access key ID. The only thing it stores is your light or dark theme choice, in your browser's local storage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/sigv4-canonical-request-differ.git
cd sigv4-canonical-request-differ
```

Open `index.html` in a browser. Or serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Independence and provenance

This is an independent tool. It is not affiliated with or endorsed by Amazon Web Services. Rules were reimplemented from AWS public documentation and from the documentation in the smithy-typescript signer; no code is reproduced here.

## Third-party notices

**AWS documentation** (IAM User Guide pages linked above, and the AWS re:Post Knowledge Center article). What was taken: the short rule sentences quoted on the page and in this README, with attribution, and the structure of the "should have been" error message, filled with other values.

**smithy-lang/smithy-typescript** (https://github.com/smithy-lang/smithy-typescript), licensed Apache-2.0. What was taken: three short phrases from comments in `packages/signature-v4/src/SignatureV4Base.ts`, quoted above with attribution. No code is copied.

**Issue and pull request text** from curl/curl, anthropics/anthropic-sdk-csharp, openai/codex and aws/aws-sdk-js-v3 on GitHub: titles and one sentence each, quoted as references.

## Related

- https://github.com/0xelitesystem/http-message-signature-base-builder
- https://github.com/0xelitesystem/webhook-signature-verifier
- https://github.com/0xelitesystem/iam-policy-analyzer

## License

MIT. Copyright (c) 2026 0xelitesystem. See [LICENSE](LICENSE).
