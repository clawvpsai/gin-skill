# File Uploads — Multipart Forms, Cloud Storage, Image Decode Safety

## Gin Multipart Form Handling

```go
import (
    "mime/multipart"
    "github.com/gin-gonic/gin"
)

// Single file upload
func uploadFile(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(400, gin.H{"error": "file required"})
        return
    }

    // Save file — use UUID for unique filename
    filename := fmt.Sprintf("%s-%s", uuid.New().String(), file.Filename)
    if err := c.SaveUploadedFile(file, "./uploads/"+filename); err != nil {
        c.JSON(500, gin.H{"error": "save failed"})
        return
    }

    c.JSON(200, gin.H{"url": "/uploads/" + filename})
}
```

### UUID Package — stdlib vs google/uuid

**Go 1.27+ (stdlib, preferred):**
```go
import "uuid"

// uuid.New() → UUID (v4, random)
// uuid.NewRandomV7() → UUID (v7, time-ordered — better for database indexes)
// uuid.Parse(string) → UUID, error
id := uuid.New()
id := uuid.NewRandomV7()
// uuid.UUID is [16]byte — compatible with github.com/google/uuid
```

**Go <1.27 (google/uuid):**
```go
import "github.com/google/uuid"

id := uuid.New()
```

The stdlib `uuid.UUID` type is `[16]byte` — identical to `github.com/google/uuid`, so migration is trivial:
```go
// Before (Go < 1.27)
import "github.com/google/uuid"
id := uuid.New() // type: uuid.UUID

// After (Go 1.27+)
import "uuid"
id := uuid.New() // type: uuid.UUID (same [16]byte, no code changes needed)
```

**Recommendation:** Use stdlib `uuid` on Go 1.27+. For cross-version compatibility, keep `github.com/google/uuid` in `go.mod` — the stdlib package is API-compatible. When dropping Go <1.27 support, remove the `github.com/google/uuid` dependency.

## Multiple Files

```go
func uploadMultiple(c *gin.Context) {
    form, err := c.MultipartForm()
    if err != nil {
        c.JSON(400, gin.H{"error": "multipart form required"})
        return
    }

    files := form.File["files"]
    urls := make([]string, 0, len(files))
    
    for _, file := range files {
        filename := fmt.Sprintf("%s-%s", uuid.New().String(), file.Filename)
        if err := c.SaveUploadedFile(file, "./uploads/"+filename); err != nil {
            continue
        }
        urls = append(urls, "/uploads/"+filename)
    }

    c.JSON(200, gin.H{"files": urls})
}
```

## File Size Limits

```go
// Set max multipart memory (default 32MB)
r.MaxMultipartMemory = 8 << 20 // 8 MB

// Validate in handler
func uploadFile(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(400, gin.H{"error": "file required"})
        return
    }

    // 10MB limit
    if file.Size > 10*1024*1024 {
        c.JSON(400, gin.H{"error": "file too large (max 10MB)"})
        return
    }
}
```

**Nginx config for body size:**
```nginx
client_max_body_size 10m;
```

## File Validation

```go
import (
    "mime/multipart"
    "net/http"
    "path/filepath"
    "strings"
)

func validateFile(file *multipart.FileHeader) error {
    // Check extension
    ext := strings.ToLower(filepath.Ext(file.Filename))
    allowed := map[string]bool{".jpg": true, ".jpeg": true, ".png": true, ".gif": true, ".pdf": true}
    if !allowed[ext] {
        return fmt.Errorf("invalid file type: %s", ext)
    }

    // Check MIME type (more reliable than extension)
    f, _ := file.Open()
    buffer := make([]byte, 512)
    f.Read(buffer)
    f.Close()

    contentType := http.DetectContentType(buffer)
    allowedTypes := map[string]bool{
        "image/jpeg":        true,
        "image/png":         true,
        "image/gif":         true,
        "application/pdf":   true,
    }
    if !allowedTypes[contentType] {
        return fmt.Errorf("invalid MIME type: %s", contentType)
    }

    return nil
}
```

## Cloud Storage (S3)

```go
import (
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
    "github.com/google/uuid"
)

// Upload to S3
func uploadToS3(ctx context.Context, file *multipart.FileHeader) (string, error) {
    cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion("us-east-1"))
    if err != nil {
        return "", err
    }

    client := s3.NewFromConfig(cfg)
    
    key := fmt.Sprintf("uploads/%s%s", uuid.New().String(), filepath.Ext(file.Filename))
    
    f, err := file.Open()
    if err != nil {
        return "", err
    }
    defer f.Close()

    _, err = client.PutObject(ctx, &s3.PutObjectInput{
        Bucket: aws.String(os.Getenv("S3_BUCKET")),
        Key:    aws.String(key),
        Body:   f,
    })
    if err != nil {
        return "", err
    }

    return fmt.Sprintf("https://%s.s3.amazonaws.com/%s", os.Getenv("S3_BUCKET"), key), nil
}
```

**Note:** `github.com/google/uuid` is shown above for Go <1.27. On Go 1.27+, use stdlib `uuid` instead — the import path changes, but the API (`uuid.New()`) and type (`uuid.UUID = [16]byte`) are identical.

## Presigned URLs (Private S3)

```go
import (
    "github.com/aws/aws-sdk-go-v2/service/s3"
    "github.com/aws/aws-sdk-go-v2/service/s3/presigned"
)

func getPresignedURL(ctx context.Context, key string) (string, error) {
    cfg, _ := config.LoadDefaultConfig(ctx)
    client := s3.NewFromConfig(cfg)

    presignClient := s3.NewPresignClient(client)
    
    request, err := presignClient.PresignGetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(os.Getenv("S3_BUCKET")),
        Key:    aws.String(key),
    }, func(opts *s3.PresignOptions) {
        opts.Expires = time.Duration(5 * time.Minute)
    })
    if err != nil {
        return "", err
    }
    
    return request.URL, nil
}
```

## Image Decode Safety (CVE-2026-42500 + CVE-2026-46602 + CVE-2026-46604)

**CVE-2026-42500** (published 2026-05-29, Medium severity): Decoding a paletted BMP file with an out-of-range palette index **panics** in `golang.org/x/image/bmp`. Affects any Gin handler that accepts image uploads and runs `image.Decode` / `image.DecodeConfig` without restricting formats. Fixed in `golang.org/x/image` v0.41.0+.

**CVE-2026-46602 / GO-2026-5062** (published 2026-06-25, Medium severity, CWE-789): The TIFF decoder in `golang.org/x/image/tiff` does not limit tile size in tiled images. A crafted TIFF can claim a huge tile dimension in the header, causing the decoder to allocate gigabytes of memory — unauthenticated remote DoS. **Fixed in `golang.org/x/image` v0.43.0** (released 2026-06-15). The prior v0.41.0 floor is **insufficient** for this CVE — must upgrade.

**CVE-2026-46604 / GO-2026-5066** (published 2026-06-26 20:04 UTC, Medium severity, CWE-787): The TIFF decoder reads strip offsets from the IFD without bounds checking. A crafted TIFF whose IFD declares a strip offset past the end of the underlying byte stream causes the decoder to index into the raw buffer and panic with `runtime error: index out of range`. Unauthenticated remote DoS — **fires on the first strip read**, so the `defer recover()` guard must wrap the entire `image.Decode` call (not a sub-function). **Fixed in the same `golang.org/x/image` v0.43.0** release as CVE-2026-46602 — both TIFF CVEs shipped together; same upgrade covers both.

**Mitigations:**
1. **Upgrade `golang.org/x/image` to v0.43.0+** — covers all three CVEs (BMP panic + TIFF tile OOM + TIFF strip panic) in one bump.
2. **Restrict accepted formats at the content sniff level.** Reject any upload that isn't JPEG/PNG/WebP before decoding (do **not** register `tiff` unless your domain needs it):
   ```go
   var allowedFormats = map[string]bool{
       "image/jpeg": true, "image/png": true, "image/webp": true,
   }

   func decodeAllowedImage(r io.Reader) (image.Image, string, error) {
       // Read first 512 bytes for MIME sniff
       buf := make([]byte, 512)
       n, _ := io.ReadFull(r, buf)
       contentType := http.DetectContentType(buf[:n])
       if !allowedFormats[contentType] {
           return nil, "", fmt.Errorf("unsupported image format: %s", contentType)
       }
       // Reset reader and decode
       if _, err := r.(io.ReadSeeker).Seek(0, io.SeekStart); err != nil {
           // Fall back to combining bytes
           r = io.MultiReader(bytes.NewReader(buf[:n]), r)
       }
       return image.Decode(r.(io.Reader))
   }
   ```
3. **Cap decoded dimensions before full decode** — block the TIFF tile-size bomb even if a decoder bug ships:
   ```go
   cfg, _, err := image.DecodeConfig(r)
   if err != nil { return nil, "", err }
   const maxPixels = 50_000_000 // 50M pixels — adjust per domain
   if int64(cfg.Width)*int64(cfg.Height) > maxPixels {
       return nil, "", fmt.Errorf("image too large: %dx%d", cfg.Width, cfg.Height)
   }
   r.Seek(0, io.SeekStart)
   img, format, err := image.Decode(r)
   ```
4. **Wrap decode in `recover()` as defense-in-depth** (same pattern shown in `security.md`).

## Path Traversal Prevention (CVE-2026-22786 Lesson)

The Jan 2026 CVE-2026-22786 against gin-vue-admin showed what happens when user-supplied filenames are concatenated to a directory path without sanitization. In raw Gin handlers, apply this rule:

```go
import "path/filepath"
import "strings"

// ALWAYS sanitize user-supplied filenames before using them in filesystem paths
func sanitizeUploadFilename(name string) (string, error) {
    base := filepath.Base(name) // strips any directory components
    if base == "." || base == "/" || base == "" {
        return "", errors.New("empty or invalid filename")
    }
    // Belt-and-braces: reject anything containing ".."
    if strings.Contains(base, "..") {
        return "", errors.New("filename contains '..'")
    }
    // Restrict to a conservative character class (no spaces, no unicode tricks)
    if matched, _ := regexp.MatchString(`^[a-zA-Z0-9._-]+$`, base); !matched {
        return "", errors.New("filename contains disallowed characters")
    }
    return base, nil
}

// In your upload handler:
func uploadFile(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(400, gin.H{"error": "file required"})
        return
    }
    safeName, err := sanitizeUploadFilename(file.Filename)
    if err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    finalName := fmt.Sprintf("%s-%s", uuid.New().String(), safeName)
    if err := c.SaveUploadedFile(file, "./uploads/"+finalName); err != nil {
        c.JSON(500, gin.H{"error": "save failed"})
        return
    }
    c.JSON(200, gin.H{"url": "/uploads/" + finalName})
}
```

**Why `filepath.Base` alone isn't enough:** `filepath.Base("../foo/bar")` returns `"bar"`, which is safe — but `filepath.Base("..\foo\bar")` on Windows returns `"bar"` too, and an attacker can use URL-encoded `%2e%2e%2f` before the request reaches the handler. Whitelisting a strict character class (alphanumeric + `.` + `_` + `-`) closes that loophole.

## Common Mistakes

1. **Using original filename** — `file.Filename` is user-controlled; generate UUID-based names
2. **Not checking file size** — large uploads can fill disk; validate `file.Size` before saving
3. **Trusting MIME from header** — always validate by reading file header bytes with `DetectContentType`
4. **Public S3 bucket** — keep bucket private, use presigned URLs or CloudFront signed URLs
5. **Not setting content type** — `Content-Type` header affects how browsers handle downloads
6. **No virus scanning** — scan uploaded files with ClamAV before serving
13. **Not restricting decoded image formats** — CVE-2026-42500 BMP panic + CVE-2026-46602 TIFF tile-size DoS + CVE-2026-46604 TIFF strip-offset panic; sniff content type before decoding, pin `golang.org/x/image` ≥ v0.43.0, wrap `image.Decode` in `defer recover()` (must wrap the full call, not a sub-function, due to CVE-2026-46604 firing on first IFD read)
14. **Trusting `file.Filename` for filesystem paths** — CVE-2026-22786 path traversal; always sanitize with `filepath.Base` + regex whitelist

## Updated from Research (2026-05)

### UUID Package (Go 1.27 Native)
- Go 1.27 adds `uuid` package to stdlib (not `crypto/uuid`)
- API: `uuid.New()` (v4), `uuid.NewRandomV7()` (v7, time-ordered), `uuid.Parse()`
- `uuid.UUID` is `[16]byte` — API-compatible with `github.com/google/uuid`
- Migration from `github.com/google/uuid` is trivial — same types, same method names
- UUIDv7 is recommended for database-primary-key use cases (time-ordered, better index locality than v4)

### AWS SDK v2
- AWS SDK v2 for Go is the recommended S3 client (v1 is in maintenance mode)
- Presigned URLs expire and are the standard way to serve private S3 files

Source: [Gin File Upload](https://gin-gonic.com/docs/examples/file-upload/) | [Go 1.27 UUID Proposal](https://rednafi.com/shards/2026/04/go-uuid/) | [AWS SDK v2 S3](https://aws.github.io/aws-sdk-go-v2/docs/configuring-sdk/)
