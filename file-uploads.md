# File Uploads — Multipart Forms, Cloud Storage

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

    // Save file
    filename := fmt.Sprintf("%s-%s", uuid.New().String(), file.Filename)
    if err := c.SaveUploadedFile(file, "./uploads/"+filename); err != nil {
        c.JSON(500, gin.H{"error": "save failed"})
        return
    }

    c.JSON(200, gin.H{"url": "/uploads/" + filename})
}
```

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
        "image/jpeg": true,
        "image/png":  true,
        "image/gif":  true,
        "application/pdf": true,
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

## Presigned URLs (Private S3)

```go
import (
    "github.com/aws/aws-sdk-go-v2/service/s3/presigned"
    "github.com/aws/aws-sdk-go-v2/service/s3"
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

## Common Mistakes

1. **Using original filename** — `file.Filename` is user-controlled; generate UUID-based names
2. **Not checking file size** — large uploads can fill disk; validate `file.Size` before saving
3. **Trusting MIME from header** — always validate by reading file header bytes with `DetectContentType`
4. **Public S3 bucket** — keep bucket private, use presigned URLs or CloudFront signed URLs
5. **Not setting content type** — `Content-Type` header affects how browsers handle downloads
6. **No virus scanning** — scan uploaded files with ClamAV before serving

## Updated from Research (2026-05)
- AWS SDK v2 for Go is the recommended S3 client (v1 is in maintenance mode)
- Presigned URLs expire and are the standard way to serve private S3 files

Source: [Gin File Upload](https://gin-gonic.com/docs/examples/file-upload/) | [AWS SDK v2 S3](https://aws.github.io/aws-sdk-go-v2/docs/configuring-sdk/)
