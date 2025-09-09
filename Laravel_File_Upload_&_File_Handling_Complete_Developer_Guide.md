# Laravel File Upload & File Handling: Complete Developer Guide

## Table of Contents

1. [File Upload Basics](#file-upload-basics)
2. [File Storage Fundamentals](#file-storage-fundamentals)
3. [File Handling & Retrieval](#file-handling--retrieval)
4. [Storage Disks & Drivers](#storage-disks--drivers)
5. [Validation & Security](#validation--security)
6. [Real-World Examples](#real-world-examples)
7. [When NOT to Use Laravel's File Handling](#when-not-to-use-laravels-file-handling)
8. [Senior Developer Best Practices](#senior-developer-best-practices)
9. [Testing File Uploads](#testing-file-uploads)
10. [Summary & Key Takeaways](#summary--key-takeaways)

---

## File Upload Basics

### Understanding Laravel's File Upload Flow

Laravel provides a robust file upload system built on top of Symfony's file handling components. The flow involves receiving uploads through HTTP requests, validating files, storing them using filesystem abstractions, and managing access through various retrieval methods.

**Core Components:**

- `Illuminate\Http\UploadedFile` - Represents uploaded files
- `Illuminate\Support\Facades\Storage` - Filesystem abstraction layer
- `Illuminate\Filesystem\FilesystemManager` - Manages multiple storage disks
- Built-in validation rules for file constraints

### Basic File Upload Handling

**Problem**: Handle a simple file upload from a form input.

**Before** (manual PHP file handling):

```php
<?php

class DocumentController extends Controller
{
    public function store(Request $request)
    {
        // Manual PHP file handling - error-prone and insecure
        if (isset($_FILES['document'])) {
            $file = $_FILES['document'];

            if ($file['error'] === UPLOAD_ERR_OK) {
                $filename = $file['name'];
                $destination = public_path('uploads/' . $filename);
                move_uploaded_file($file['tmp_name'], $destination);
            }
        }

        return redirect()->back();
    }
}
```

**After** (Laravel's approach):

```php
<?php

class DocumentController extends Controller
{
    public function store(Request $request)
    {
        // Laravel provides clean, secure file handling
        if ($request->hasFile('document')) {
            $file = $request->file('document');

            // Validate the file
            $request->validate([
                'document' => 'required|file|mimes:pdf,doc,docx|max:10240', // 10MB
            ]);

            // Store the file securely
            $path = $file->store('documents', 'private');

            // Save file info to database
            Document::create([
                'name' => $file->getClientOriginalName(),
                'path' => $path,
                'size' => $file->getSize(),
                'mime_type' => $file->getMimeType(),
            ]);
        }

        return redirect()->back()->with('success', 'Document uploaded successfully');
    }
}
```

**Key Improvements:**

- `$request->hasFile()` safely checks for file presence
- `$request->file()` returns an `UploadedFile` instance with helpful methods
- Built-in validation rules handle size, MIME type, and extension checks
- `store()` method handles secure file storage with automatic path generation
- Files stored outside public directory by default for security
- Proper error handling and user feedback

### Working with UploadedFile Methods

The `UploadedFile` class provides numerous methods for file inspection and handling:

```php
<?php

public function store(Request $request)
{
    $file = $request->file('upload');

    // File information methods
    $originalName = $file->getClientOriginalName();     // "document.pdf"
    $extension = $file->getClientOriginalExtension();   // "pdf"
    $mimeType = $file->getMimeType();                   // "application/pdf"
    $size = $file->getSize();                           // File size in bytes
    $error = $file->getError();                         // Upload error code

    // File validation helpers
    $isValid = $file->isValid();                        // No upload errors
    $path = $file->getRealPath();                       // Temporary file path

    // Generate secure filename
    $filename = time() . '_' . Str::random(10) . '.' . $extension;

    // Store with custom name
    $storedPath = $file->storeAs('documents', $filename, 'private');

    return response()->json(['path' => $storedPath]);
}
```

---

## File Storage Fundamentals

### Storage Methods and Options

Laravel offers multiple storage methods to accommodate different scenarios and security requirements.

**Basic Storage Methods:**

```php
<?php

class FileUploadService
{
    public function demonstrateStorageMethods(UploadedFile $file): array
    {
        $results = [];

        // 1. Basic store - auto-generates filename
        $results['auto_path'] = $file->store('uploads');
        // Returns: "uploads/XYZ123.pdf"

        // 2. Store with custom filename
        $customName = 'document_' . now()->timestamp . '.pdf';
        $results['custom_path'] = $file->storeAs('uploads', $customName);
        // Returns: "uploads/document_1634567890.pdf"

        // 3. Store on specific disk
        $results['s3_path'] = $file->store('uploads', 's3');
        // Stores on S3 bucket

        // 4. Store with visibility control
        $results['public_path'] = $file->store('uploads', ['disk' => 'public']);
        // Publicly accessible files

        return $results;
    }
}
```

### Public vs Private Storage

Understanding when to use public versus private storage is crucial for security and access control.

**Public Storage** (`storage/app/public/`):

- Files accessible via web URL
- Suitable for: user avatars, product images, public documents
- Requires symbolic link: `php artisan storage:link`
- URL: `Storage::url($path)` generates public URLs

**Private Storage** (`storage/app/`):

- Files not directly accessible via web
- Suitable for: user documents, invoices, private media
- Access controlled through application logic
- Served via controllers with authentication/authorization

```php
<?php

class AvatarController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'avatar' => 'required|image|max:2048',
        ]);

        $user = auth()->user();

        // Delete old avatar if exists
        if ($user->avatar_path) {
            Storage::disk('public')->delete($user->avatar_path);
        }

        // Store new avatar in public disk
        $path = $request->file('avatar')->store('avatars', 'public');

        // Update user record
        $user->update(['avatar_path' => $path]);

        return response()->json([
            'message' => 'Avatar updated successfully',
            'avatar_url' => Storage::url($path), // Generate public URL
        ]);
    }
}

class DocumentController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'document' => 'required|file|mimes:pdf,doc,docx|max:10240',
        ]);

        // Store in private disk - requires authentication to access
        $path = $request->file('document')->store('documents', 'private');

        Document::create([
            'user_id' => auth()->id(),
            'path' => $path,
            'name' => $request->file('document')->getClientOriginalName(),
        ]);

        return redirect()->back()->with('success', 'Document uploaded securely');
    }
}
```

### Using Storage Facade

The Storage facade provides a consistent API across different storage drivers and enables easy switching between local, S3, FTP, and other storage systems.

```php
<?php

use Illuminate\Support\Facades\Storage;

class FileManagerService
{
    public function moveFileToArchive(string $filePath): string
    {
        // Read file content
        $content = Storage::disk('private')->get($filePath);

        // Generate archive path
        $archivePath = 'archive/' . date('Y/m/d/') . basename($filePath);

        // Write to archive location
        Storage::disk('archive')->put($archivePath, $content);

        // Delete original
        Storage::disk('private')->delete($filePath);

        return $archivePath;
    }

    public function copyBetweenDisks(string $sourcePath, string $destinationPath): bool
    {
        // Get file from source disk
        $fileContent = Storage::disk('local')->get($sourcePath);

        // Put on destination disk
        return Storage::disk('s3')->put($destinationPath, $fileContent);
    }

    public function getFileInfo(string $path, string $disk = 'private'): array
    {
        $storage = Storage::disk($disk);

        return [
            'exists' => $storage->exists($path),
            'size' => $storage->size($path),
            'last_modified' => $storage->lastModified($path),
            'mime_type' => $storage->mimeType($path),
            'visibility' => $storage->getVisibility($path),
        ];
    }
}
```

---

## File Handling & Retrieval

### Serving Files Securely

**Problem**: Serve private files while maintaining access control and security.

**Before** (insecure direct access):

```php
<?php

// BAD: Direct file serving without access control
class DocumentController extends Controller
{
    public function download($filename)
    {
        // Insecure - allows arbitrary file access
        $path = storage_path('app/documents/' . $filename);

        if (file_exists($path)) {
            return response()->download($path);
        }

        abort(404);
    }
}
```

**After** (secure controlled access):

```php
<?php

class DocumentController extends Controller
{
    public function download(Document $document)
    {
        // Verify user has access to this document
        $this->authorize('download', $document);

        // Check if file exists
        if (!Storage::disk('private')->exists($document->path)) {
            abort(404, 'File not found');
        }

        // Serve file securely
        return Storage::disk('private')->download(
            $document->path,
            $document->original_name,
            ['Content-Type' => $document->mime_type]
        );
    }

    public function stream(Document $document)
    {
        $this->authorize('view', $document);

        // Stream file for large files or inline viewing
        return Storage::disk('private')->response(
            $document->path,
            $document->original_name,
            ['Content-Type' => $document->mime_type]
        );
    }

    public function generateSignedUrl(Document $document)
    {
        $this->authorize('share', $document);

        // Create temporary signed URL (expires in 1 hour)
        return URL::temporarySignedRoute(
            'documents.download-signed',
            now()->addHour(),
            ['document' => $document->id]
        );
    }
}
```

**Security Improvements:**

- Authorization policies prevent unauthorized access
- File paths are validated against database records
- No direct filesystem path exposure
- Support for signed URLs with expiration
- Proper MIME type headers for security

### File Operations

Common file operations using Laravel's Storage facade:

```php
<?php

class FileOperationService
{
    protected string $disk = 'private';

    public function createDirectory(string $path): bool
    {
        return Storage::disk($this->disk)->makeDirectory($path);
    }

    public function moveFile(string $from, string $to): bool
    {
        return Storage::disk($this->disk)->move($from, $to);
    }

    public function copyFile(string $from, string $to): bool
    {
        return Storage::disk($this->disk)->copy($from, $to);
    }

    public function deleteFile(string $path): bool
    {
        return Storage::disk($this->disk)->delete($path);
    }

    public function deleteMultiple(array $paths): bool
    {
        return Storage::disk($this->disk)->delete($paths);
    }

    public function listFiles(string $directory = ''): array
    {
        return Storage::disk($this->disk)->files($directory);
    }

    public function listDirectories(string $directory = ''): array
    {
        return Storage::disk($this->disk)->directories($directory);
    }

    public function getFileSize(string $path): int
    {
        return Storage::disk($this->disk)->size($path);
    }

    public function getLastModified(string $path): int
    {
        return Storage::disk($this->disk)->lastModified($path);
    }

    public function setVisibility(string $path, string $visibility): bool
    {
        // $visibility: 'public' or 'private'
        return Storage::disk($this->disk)->setVisibility($path, $visibility);
    }
}
```

---

## Storage Disks & Drivers

### Filesystem Configuration

Laravel's filesystem configuration in `config/filesystems.php` defines available storage disks and their drivers.

```php
<?php

// config/filesystems.php
return [
    'default' => env('FILESYSTEM_DISK', 'local'),

    'disks' => [
        'local' => [
            'driver' => 'local',
            'root' => storage_path('app'),
            'throw' => false,
        ],

        'public' => [
            'driver' => 'local',
            'root' => storage_path('app/public'),
            'url' => env('APP_URL').'/storage',
            'visibility' => 'public',
            'throw' => false,
        ],

        's3' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BUCKET'),
            'url' => env('AWS_URL'),
            'endpoint' => env('AWS_ENDPOINT'),
            'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
            'throw' => false,
        ],

        'ftp' => [
            'driver' => 'ftp',
            'host' => env('FTP_HOST'),
            'username' => env('FTP_USERNAME'),
            'password' => env('FTP_PASSWORD'),
            'port' => env('FTP_PORT', 21),
            'root' => env('FTP_ROOT'),
            'passive' => env('FTP_PASSIVE', true),
            'ssl' => env('FTP_SSL', false),
            'timeout' => 30,
        ],

        // Custom disk for user-specific storage
        'user_documents' => [
            'driver' => 'local',
            'root' => storage_path('app/users'),
            'visibility' => 'private',
        ],

        // Archive disk with different retention policies
        'archive' => [
            'driver' => 's3',
            'key' => env('ARCHIVE_AWS_KEY'),
            'secret' => env('ARCHIVE_AWS_SECRET'),
            'region' => env('ARCHIVE_AWS_REGION'),
            'bucket' => env('ARCHIVE_AWS_BUCKET'),
            'options' => [
                'StorageClass' => 'GLACIER', // Long-term archival
            ],
        ],
    ],
];
```

### Choosing the Right Disk

**Local Disk (`local`):**

- **Use for**: Development, testing, temporary files
- **Pros**: Fast, no external dependencies, simple setup
- **Cons**: Not scalable, server-dependent, no CDN support

**Public Disk (`public`):**

- **Use for**: Static assets, user avatars, public images
- **Pros**: Direct web access, fast serving, simple URLs
- **Cons**: Security risk for sensitive files, server storage limits

**S3 Disk (`s3`):**

- **Use for**: Production file storage, large files, scalable applications
- **Pros**: Unlimited storage, CDN integration, durability, global availability
- **Cons**: Network latency, costs for frequent access, requires AWS setup

**FTP Disk (`ftp`):**

- **Use for**: Legacy systems integration, existing FTP infrastructure
- **Pros**: Integration with existing systems
- **Cons**: Limited features, security concerns, slower than modern alternatives

### Working with S3 Storage

**Problem**: Store files on S3 with proper configuration for production use.

**Before** (basic S3 setup):

```php
<?php

class DocumentController extends Controller
{
    public function store(Request $request)
    {
        // Basic S3 storage without optimization
        $path = $request->file('document')->store('documents', 's3');

        return response()->json(['path' => $path]);
    }
}
```

**After** (production-ready S3 configuration):

```php
<?php

class DocumentController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'document' => 'required|file|mimes:pdf,doc,docx,jpg,png|max:51200', // 50MB
        ]);

        $file = $request->file('document');
        $user = auth()->user();

        // Generate organized path structure
        $directory = sprintf(
            'documents/%s/%s',
            $user->id,
            date('Y/m')
        );

        // Generate unique filename to prevent conflicts
        $filename = sprintf(
            '%s_%s.%s',
            time(),
            Str::random(8),
            $file->getClientOriginalExtension()
        );

        // Store on S3 with custom options
        $path = $file->storeAs($directory, $filename, [
            'disk' => 's3',
            'visibility' => 'private',
        ]);

        // Store metadata in database
        $document = Document::create([
            'user_id' => $user->id,
            'filename' => $file->getClientOriginalName(),
            'path' => $path,
            'size' => $file->getSize(),
            'mime_type' => $file->getMimeType(),
        ]);

        return response()->json([
            'id' => $document->id,
            'message' => 'Document uploaded successfully',
            'download_url' => route('documents.download', $document),
        ]);
    }

    public function generateTemporaryUrl(Document $document, int $minutes = 60): string
    {
        // Generate temporary S3 URL (bypasses application for performance)
        return Storage::disk('s3')->temporaryUrl($document->path, now()->addMinutes($minutes));
    }

    public function getPublicUrl(Document $document): string
    {
        // For public files, get permanent URL
        return Storage::disk('s3')->url($document->path);
    }
}
```

**Production Considerations:**

- Organized directory structure prevents listing issues
- Unique filenames prevent conflicts and overwrites
- Private visibility ensures access control
- Temporary URLs reduce server load for file serving
- Database metadata enables search and management
- Proper validation prevents malicious uploads

---

## Validation & Security

### Comprehensive File Validation

**Problem**: Validate uploaded files against multiple security criteria including size, type, and content.

**Before** (basic validation):

```php
<?php

class FileController extends Controller
{
    public function store(Request $request)
    {
        // Basic validation - insufficient for security
        $request->validate([
            'file' => 'required|file',
        ]);

        $path = $request->file('file')->store('uploads');

        return response()->json(['path' => $path]);
    }
}
```

**After** (comprehensive validation):

```php
<?php

class FileController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'file' => [
                'required',
                'file',
                'max:10240',                    // 10MB max size
                'mimes:pdf,doc,docx,jpg,png',  // Allowed MIME types
                'extensions:pdf,doc,docx,jpg,png', // Double-check extensions
                function ($attribute, $value, $fail) {
                    // Custom validation for file content
                    if (!$this->isValidFileContent($value)) {
                        $fail('The uploaded file contains invalid content.');
                    }
                },
            ],
        ]);

        $file = $request->file('file');

        // Additional security checks
        $this->performSecurityChecks($file);

        // Sanitize filename
        $safeFilename = $this->sanitizeFilename($file->getClientOriginalName());

        // Store with sanitized name
        $path = $file->storeAs(
            'uploads/' . auth()->id(),
            $safeFilename,
            'private'
        );

        return response()->json(['path' => $path]);
    }

    protected function isValidFileContent(UploadedFile $file): bool
    {
        // Check file signature/magic numbers
        $finfo = finfo_open(FILEINFO_MIME_TYPE);
        $detectedMimeType = finfo_file($finfo, $file->getRealPath());
        finfo_close($finfo);

        // Verify MIME type matches extension
        $allowedTypes = [
            'pdf' => 'application/pdf',
            'doc' => 'application/msword',
            'docx' => 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            'jpg' => 'image/jpeg',
            'png' => 'image/png',
        ];

        $extension = strtolower($file->getClientOriginalExtension());

        return isset($allowedTypes[$extension]) &&
               $detectedMimeType === $allowedTypes[$extension];
    }

    protected function performSecurityChecks(UploadedFile $file): void
    {
        // Check for embedded scripts in images
        if (in_array($file->getMimeType(), ['image/jpeg', 'image/png', 'image/gif'])) {
            $content = file_get_contents($file->getRealPath());

            // Look for script tags or PHP code
            if (preg_match('/<script|<\?php|\x00/i', $content)) {
                throw new \InvalidArgumentException('Malicious content detected in image file.');
            }
        }

        // Check file size against PHP limits
        $maxSize = min(
            ini_get('upload_max_filesize'),
            ini_get('post_max_size'),
            ini_get('memory_limit')
        );

        if ($file->getSize() > $this->parseSize($maxSize)) {
            throw new \InvalidArgumentException('File exceeds server limits.');
        }
    }

    protected function sanitizeFilename(string $filename): string
    {
        // Remove path traversal attempts
        $filename = basename($filename);

        // Remove special characters and spaces
        $filename = preg_replace('/[^a-zA-Z0-9.-]/', '_', $filename);

        // Ensure reasonable length
        if (strlen($filename) > 100) {
            $info = pathinfo($filename);
            $name = substr($info['filename'], 0, 90);
            $filename = $name . '.' . $info['extension'];
        }

        // Add timestamp to prevent conflicts
        $info = pathinfo($filename);

        return sprintf(
            '%s_%s.%s',
            $info['filename'],
            time(),
            $info['extension']
        );
    }

    protected function parseSize(string $size): int
    {
        $size = trim($size);
        $last = strtolower($size[strlen($size) - 1]);
        $size = (int) $size;

        return match ($last) {
            'g' => $size * 1073741824,
            'm' => $size * 1048576,
            'k' => $size * 1024,
            default => $size,
        };
    }
}
```

### Multi-Tenant Security

**Problem**: Ensure file isolation in multi-tenant applications where users should only access their own files.

```php
<?php

class TenantAwareFileController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'file' => 'required|file|max:10240',
            'folder' => 'nullable|string|regex:/^[a-zA-Z0-9_-]+$/', // Sanitize folder names
        ]);

        $tenant = auth()->user()->tenant;
        $folder = $request->input('folder', 'general');

        // Create tenant-specific directory structure
        $directory = sprintf(
            'tenants/%s/users/%s/%s',
            $tenant->id,
            auth()->id(),
            $folder
        );

        $path = $request->file('file')->store($directory, 'private');

        // Store with tenant and user association
        File::create([
            'tenant_id' => $tenant->id,
            'user_id' => auth()->id(),
            'path' => $path,
            'folder' => $folder,
            'filename' => $request->file('file')->getClientOriginalName(),
            'size' => $request->file('file')->getSize(),
        ]);

        return response()->json(['message' => 'File uploaded successfully']);
    }

    public function download(File $file)
    {
        // Ensure user can only access their tenant's files
        if ($file->tenant_id !== auth()->user()->tenant_id ||
            $file->user_id !== auth()->id()) {
            abort(403);
        }

        return Storage::disk('private')->download($file->path, $file->filename);
    }

    public function list(Request $request)
    {
        $tenant = auth()->user()->tenant;
        $folder = $request->input('folder', 'general');

        // Only show files from current tenant and user
        $files = File::where('tenant_id', $tenant->id)
                    ->where('user_id', auth()->id())
                    ->where('folder', $folder)
                    ->get();

        return response()->json($files);
    }
}
```

---

## Real-World Examples

### 1. User Avatar Upload with Image Resizing

**Problem**: Allow users to upload avatars with automatic resizing and old avatar cleanup.

**Before** (basic avatar upload):

```php
<?php

class ProfileController extends Controller
{
    public function updateAvatar(Request $request)
    {
        // Basic upload without optimization or cleanup
        $path = $request->file('avatar')->store('avatars', 'public');

        auth()->user()->update(['avatar' => $path]);

        return back();
    }
}
```

**After** (production-ready avatar system):

```php
<?php

use Intervention\Image\Facades\Image;

class AvatarUploadService
{
    protected array $sizes = [
        'thumb' => 50,
        'small' => 100,
        'medium' => 200,
        'large' => 400,
    ];

    public function upload(UploadedFile $file, User $user): array
    {
        // Validate image
        $this->validateImage($file);

        // Clean up old avatars
        $this->cleanupOldAvatars($user);

        // Generate unique filename
        $filename = $this->generateFilename($file);

        // Create multiple sizes
        $paths = [];
        foreach ($this->sizes as $size => $dimensions) {
            $paths[$size] = $this->createResizedImage($file, $filename, $size, $dimensions);
        }

        // Update user record
        $user->update(['avatar_paths' => $paths]);

        return $paths;
    }

    protected function validateImage(UploadedFile $file): void
    {
        $rules = [
            'mimes:jpeg,png,jpg,gif',
            'max:5120', // 5MB
            'dimensions:min_width=50,min_height=50,max_width=2000,max_height=2000',
        ];

        validator(['avatar' => $file], ['avatar' => $rules])->validate();
    }

    protected function cleanupOldAvatars(User $user): void
    {
        if ($user->avatar_paths) {
            foreach ($user->avatar_paths as $path) {
                Storage::disk('public')->delete($path);
            }
        }
    }

    protected function generateFilename(UploadedFile $file): string
    {
        return sprintf(
            'avatars/%s/%s.%s',
            auth()->id(),
            Str::random(20),
            $file->getClientOriginalExtension()
        );
    }

    protected function createResizedImage(UploadedFile $file, string $baseFilename, string $size, int $dimensions): string
    {
        // Create resized image using Intervention Image
        $image = Image::make($file->getRealPath())
                     ->fit($dimensions, $dimensions, function ($constraint) {
                         $constraint->upsize();
                     })
                     ->encode('jpg', 85); // Optimize quality

        // Generate size-specific filename
        $info = pathinfo($baseFilename);
        $filename = $info['dirname'] . '/' . $info['filename'] . '_' . $size . '.jpg';

        // Store optimized image
        Storage::disk('public')->put($filename, $image->stream());

        return $filename;
    }
}

class ProfileController extends Controller
{
    public function updateAvatar(Request $request, AvatarUploadService $avatarService)
    {
        if (!$request->hasFile('avatar')) {
            return back()->withErrors(['avatar' => 'Please select an image file.']);
        }

        try {
            $paths = $avatarService->upload($request->file('avatar'), auth()->user());

            return back()->with('success', 'Avatar updated successfully!');
        } catch (\Exception $e) {
            return back()->withErrors(['avatar' => 'Failed to upload avatar: ' . $e->getMessage()]);
        }
    }

    public function getAvatarUrl(string $size = 'medium'): string
    {
        $user = auth()->user();

        if ($user->avatar_paths && isset($user->avatar_paths[$size])) {
            return Storage::url($user->avatar_paths[$size]);
        }

        // Return default avatar
        return asset('images/default-avatar.png');
    }
}
```

### 2. Document Upload with Signed Download Links

**Problem**: Upload sensitive documents that require authentication and generate temporary access links.

**Before** (insecure document access):

```php
<?php

class DocumentController extends Controller
{
    public function download($filename)
    {
        // Insecure - anyone with filename can download
        return response()->download(storage_path('app/documents/' . $filename));
    }
}
```

**After** (secure document system with signed URLs):

```php
<?php

class DocumentUploadService
{
    public function upload(UploadedFile $file, User $user, array $metadata = []): Document
    {
        $this->validateDocument($file);

        // Generate secure path structure
        $directory = sprintf('documents/%s/%s', $user->id, date('Y/m'));
        $filename = $this->generateSecureFilename($file);

        // Store document
        $path = $file->storeAs($directory, $filename, 'private');

        // Create database record
        return Document::create([
            'user_id' => $user->id,
            'original_name' => $file->getClientOriginalName(),
            'filename' => $filename,
            'path' => $path,
            'size' => $file->getSize(),
            'mime_type' => $file->getMimeType(),
            'metadata' => $metadata,
            'upload_ip' => request()->ip(),
        ]);
    }

    public function generateSignedDownloadUrl(Document $document, int $expirationMinutes = 60): string
    {
        return URL::temporarySignedRoute(
            'documents.signed-download',
            now()->addMinutes($expirationMinutes),
            [
                'document' => $document->id,
                'hash' => hash('sha256', $document->path . $document->created_at)
            ]
        );
    }

    public function generateShareableLink(Document $document, int $expirationHours = 24): string
    {
        $token = Str::random(32);

        // Store temporary access token
        DocumentAccess::create([
            'document_id' => $document->id,
            'access_token' => $token,
            'expires_at' => now()->addHours($expirationHours),
            'created_by' => auth()->id(),
        ]);

        return route('documents.public-access', ['token' => $token]);
    }

    protected function validateDocument(UploadedFile $file): void
    {
        validator(['document' => $file], [
            'document' => [
                'required',
                'file',
                'max:51200', // 50MB
                'mimes:pdf,doc,docx,xls,xlsx,ppt,pptx,txt,jpg,png',
            ]
        ])->validate();
    }

    protected function generateSecureFilename(UploadedFile $file): string
    {
        return sprintf(
            '%s_%s.%s',
            time(),
            Str::random(16),
            $file->getClientOriginalExtension()
        );
    }
}

class DocumentController extends Controller
{
    public function __construct(protected DocumentUploadService $documentService) {}

    public function store(Request $request)
    {
        if (!$request->hasFile('document')) {
            return response()->json(['error' => 'No file uploaded'], 400);
        }

        $metadata = [
            'category' => $request->input('category'),
            'tags' => $request->input('tags', []),
            'description' => $request->input('description'),
        ];

        $document = $this->documentService->upload(
            $request->file('document'),
            auth()->user(),
            $metadata
        );

        return response()->json([
            'id' => $document->id,
            'name' => $document->original_name,
            'download_url' => $this->documentService->generateSignedDownloadUrl($document),
            'share_url' => $this->documentService->generateShareableLink($document),
        ]);
    }

    public function signedDownload(Request $request, Document $document)
    {
        // Verify signature
        if (!$request->hasValidSignature()) {
            abort(403, 'Invalid or expired download link');
        }

        // Verify hash
        $expectedHash = hash('sha256', $document->path . $document->created_at);
        if ($request->get('hash') !== $expectedHash) {
            abort(403, 'Invalid download hash');
        }

        // Log access
        DocumentDownload::create([
            'document_id' => $document->id,
            'ip_address' => $request->ip(),
            'user_agent' => $request->userAgent(),
        ]);

        return Storage::disk('private')->download($document->path, $document->original_name);
    }

    public function publicAccess(Request $request, string $token)
    {
        $access = DocumentAccess::where('access_token', $token)
                                ->where('expires_at', '>', now())
                                ->firstOrFail();

        $document = $access->document;

        // Log access
        DocumentAccess::where('id', $access->id)->increment('access_count');

        return Storage::disk('private')->download($document->path, $document->original_name);
    }
}
```

### 3. Multiple File Upload with Progress Tracking

**Problem**: Handle multiple file uploads with validation, progress tracking, and batch processing.

```php
<?php

class BatchUploadService
{
    public function uploadMultiple(array $files, User $user, string $category): array
    {
        $results = [];
        $batchId = Str::uuid();

        // Create batch record for tracking
        $batch = UploadBatch::create([
            'batch_id' => $batchId,
            'user_id' => $user->id,
            'total_files' => count($files),
            'category' => $category,
            'status' => 'processing',
        ]);

        foreach ($files as $index => $file) {
            try {
                $result = $this->uploadSingleFile($file, $user, $category, $batch);
                $results[] = $result;

                // Update progress
                $batch->increment('completed_files');

            } catch (\Exception $e) {
                $results[] = [
                    'success' => false,
                    'filename' => $file->getClientOriginalName(),
                    'error' => $e->getMessage(),
                ];

                $batch->increment('failed_files');
            }
        }

        // Update final status
        $batch->update([
            'status' => $batch->failed_files > 0 ? 'completed_with_errors' : 'completed',
            'completed_at' => now(),
        ]);

        return [
            'batch_id' => $batchId,
            'results' => $results,
            'summary' => [
                'total' => $batch->total_files,
                'completed' => $batch->completed_files,
                'failed' => $batch->failed_files,
            ],
        ];
    }

    protected function uploadSingleFile(UploadedFile $file, User $user, string $category, UploadBatch $batch): array
    {
        // Validate individual file
        $this->validateFile($file);

        // Generate path
        $directory = sprintf('uploads/%s/%s/%s', $user->id, $category, date('Y-m-d'));
        $filename = time() . '_' . Str::random(8) . '.' . $file->getClientOriginalExtension();

        // Store file
        $path = $file->storeAs($directory, $filename, 'private');

        // Create database record
        $document = Document::create([
            'batch_id' => $batch->batch_id,
            'user_id' => $user->id,
            'original_name' => $file->getClientOriginalName(),
            'path' => $path,
            'size' => $file->getSize(),
            'mime_type' => $file->getMimeType(),
            'category' => $category,
        ]);

        return [
            'success' => true,
            'id' => $document->id,
            'filename' => $file->getClientOriginalName(),
            'path' => $path,
        ];
    }

    public function getBatchProgress(string $batchId): array
    {
        $batch = UploadBatch::where('batch_id', $batchId)->firstOrFail();

        return [
            'batch_id' => $batchId,
            'status' => $batch->status,
            'progress' => [
                'total' => $batch->total_files,
                'completed' => $batch->completed_files,
                'failed' => $batch->failed_files,
                'percentage' => round(($batch->completed_files + $batch->failed_files) / $batch->total_files * 100, 2),
            ],
            'files' => $batch->documents()->select('id', 'original_name', 'size')->get(),
        ];
    }

    protected function validateFile(UploadedFile $file): void
    {
        validator(['file' => $file], [
            'file' => [
                'required',
                'file',
                'max:20480', // 20MB per file
                'mimes:pdf,doc,docx,jpg,png,xlsx,csv',
            ]
        ])->validate();
    }
}

class BatchUploadController extends Controller
{
    public function __construct(protected BatchUploadService $batchService) {}

    public function upload(Request $request)
    {
        $request->validate([
            'files' => 'required|array|min:1|max:10',
            'files.*' => 'file',
            'category' => 'required|string|in:documents,images,reports',
        ]);

        $results = $this->batchService->uploadMultiple(
            $request->file('files'),
            auth()->user(),
            $request->input('category')
        );

        return response()->json($results);
    }

    public function progress(string $batchId)
    {
        $progress = $this->batchService->getBatchProgress($batchId);

        return response()->json($progress);
    }
}
```

### 4. Organizing Uploads into User Directories

**Problem**: Organize file uploads into logical directory structures based on users, dates, and categories.

```php
<?php

class OrganizedFileService
{
    protected array $directoryStructures = [
        'user_date' => 'users/{user_id}/{year}/{month}',
        'category_user' => '{category}/users/{user_id}',
        'tenant_user_date' => 'tenants/{tenant_id}/users/{user_id}/{year}/{month}',
        'project_based' => 'projects/{project_id}/files/{category}',
    ];

    public function store(UploadedFile $file, string $structure = 'user_date', array $context = []): array
    {
        $directory = $this->buildDirectory($structure, $context);
        $filename = $this->generateFilename($file, $context);

        // Store file
        $path = $file->storeAs($directory, $filename, 'private');

        // Create metadata record
        $fileRecord = OrganizedFile::create([
            'user_id' => auth()->id(),
            'tenant_id' => $context['tenant_id'] ?? null,
            'project_id' => $context['project_id'] ?? null,
            'category' => $context['category'] ?? 'general',
            'original_name' => $file->getClientOriginalName(),
            'stored_path' => $path,
            'directory_structure' => $structure,
            'file_size' => $file->getSize(),
            'mime_type' => $file->getMimeType(),
            'metadata' => $context['metadata'] ?? [],
        ]);

        return [
            'id' => $fileRecord->id,
            'path' => $path,
            'directory' => $directory,
            'filename' => $filename,
            'url' => $this->generateAccessUrl($fileRecord),
        ];
    }

    protected function buildDirectory(string $structure, array $context): string
    {
        $template = $this->directoryStructures[$structure] ?? $structure;

        $replacements = [
            '{user_id}' => auth()->id(),
            '{tenant_id}' => $context['tenant_id'] ?? 'default',
            '{project_id}' => $context['project_id'] ?? 'general',
            '{category}' => $context['category'] ?? 'files',
            '{year}' => date('Y'),
            '{month}' => date('m'),
            '{day}' => date('d'),
            '{date}' => date('Y-m-d'),
        ];

        return str_replace(array_keys($replacements), array_values($replacements), $template);
    }

    protected function generateFilename(UploadedFile $file, array $context): string
    {
        $prefix = $context['prefix'] ?? '';
        $timestamp = now()->format('His'); // HHMMSS
        $random = Str::random(6);
        $extension = $file->getClientOriginalExtension();

        $filename = collect([
            $prefix,
            $timestamp,
            $random
        ])->filter()->implode('_');

        return $filename . '.' . $extension;
    }

    public function listUserFiles(string $category = null, string $structure = null): array
    {
        $query = OrganizedFile::where('user_id', auth()->id());

        if ($category) {
            $query->where('category', $category);
        }

        if ($structure) {
            $query->where('directory_structure', $structure);
        }

        return $query->orderBy('created_at', 'desc')
                    ->get()
                    ->groupBy('category')
                    ->toArray();
    }

    public function moveFileToCategory(OrganizedFile $file, string $newCategory): bool
    {
        // Build new directory path
        $context = [
            'category' => $newCategory,
            'tenant_id' => $file->tenant_id,
            'project_id' => $file->project_id,
        ];

        $newDirectory = $this->buildDirectory($file->directory_structure, $context);
        $newPath = $newDirectory . '/' . basename($file->stored_path);

        // Move file
        if (Storage::disk('private')->move($file->stored_path, $newPath)) {
            $file->update([
                'stored_path' => $newPath,
                'category' => $newCategory,
            ]);

            return true;
        }

        return false;
    }

    protected function generateAccessUrl(OrganizedFile $file): string
    {
        return route('files.download', [
            'file' => $file->id,
            'token' => $file->generateAccessToken(),
        ]);
    }
}

// Usage examples
class FileUploadController extends Controller
{
    public function __construct(protected OrganizedFileService $fileService) {}

    public function uploadUserDocument(Request $request)
    {
        $request->validate([
            'document' => 'required|file|max:10240',
            'category' => 'required|in:invoices,contracts,reports',
        ]);

        $result = $this->fileService->store(
            $request->file('document'),
            'user_date',
            [
                'category' => $request->input('category'),
                'metadata' => [
                    'uploaded_from' => 'web_interface',
                    'ip_address' => $request->ip(),
                ]
            ]
        );

        return response()->json($result);
    }

    public function uploadProjectFile(Request $request, Project $project)
    {
        $request->validate([
            'file' => 'required|file|max:51200',
            'category' => 'required|in:designs,documents,assets',
        ]);

        $result = $this->fileService->store(
            $request->file('file'),
            'project_based',
            [
                'project_id' => $project->id,
                'category' => $request->input('category'),
                'prefix' => Str::slug($project->name),
            ]
        );

        return response()->json($result);
    }
}
```

### 5. S3 Storage with Temporary URLs

**Problem**: Store files on S3 and generate temporary URLs for secure access without exposing permanent links.

```php
<?php

class S3FileService
{
    protected string $disk = 's3';
    protected int $defaultExpirationMinutes = 60;

    public function uploadToS3(UploadedFile $file, array $options = []): array
    {
        // Set S3-specific options
        $s3Options = [
            'visibility' => $options['visibility'] ?? 'private',
            'ServerSideEncryption' => 'AES256',
            'StorageClass' => $options['storage_class'] ?? 'STANDARD',
        ];

        if (isset($options['content_type'])) {
            $s3Options['ContentType'] = $options['content_type'];
        }

        // Generate S3 path
        $directory = $options['directory'] ?? $this->generateDirectory();
        $filename = $this->generateS3Filename($file, $options);

        // Upload to S3 with options
        $path = $file->storeAs($directory, $filename, [
            'disk' => $this->disk,
            'options' => $s3Options,
        ]);

        // Store metadata
        $s3File = S3File::create([
            'user_id' => auth()->id(),
            'original_name' => $file->getClientOriginalName(),
            's3_path' => $path,
            's3_bucket' => config('filesystems.disks.s3.bucket'),
            'file_size' => $file->getSize(),
            'mime_type' => $file->getMimeType(),
            'visibility' => $s3Options['visibility'],
            'storage_class' => $s3Options['StorageClass'],
            'metadata' => $options['metadata'] ?? [],
        ]);

        return [
            'id' => $s3File->id,
            'path' => $path,
            'bucket' => $s3File->s3_bucket,
            'temporary_url' => $this->generateTemporaryUrl($s3File),
        ];
    }

    public function generateTemporaryUrl(S3File $file, int $expirationMinutes = null): string
    {
        $expirationMinutes = $expirationMinutes ?? $this->defaultExpirationMinutes;

        return Storage::disk($this->disk)->temporaryUrl(
            $file->s3_path,
            now()->addMinutes($expirationMinutes),
            [
                'ResponseContentDisposition' => 'attachment; filename="' . $file->original_name . '"',
                'ResponseContentType' => $file->mime_type,
            ]
        );
    }

    public function generatePresignedUploadUrl(string $filename, string $contentType, int $expirationMinutes = 15): array
    {
        $s3Client = Storage::disk($this->disk)->getDriver()->getAdapter()->getClient();
        $bucket = config('filesystems.disks.s3.bucket');

        // Generate unique S3 key
        $key = $this->generateDirectory() . '/' . $this->sanitizeFilename($filename);

        // Create presigned POST data
        $formData = $s3Client->createPresignedRequest(
            $s3Client->getCommand('PutObject', [
                'Bucket' => $bucket,
                'Key' => $key,
                'ContentType' => $contentType,
                'ServerSideEncryption' => 'AES256',
            ]),
            '+' . $expirationMinutes . ' minutes'
        );

        return [
            'upload_url' => (string) $formData->getUri(),
            'headers' => $formData->getHeaders(),
            's3_key' => $key,
            'expires_at' => now()->addMinutes($expirationMinutes),
        ];
    }

    public function copyToArchive(S3File $file): bool
    {
        $archivePath = 'archive/' . date('Y/m/') . basename($file->s3_path);

        // Copy to archive storage class
        $success = Storage::disk($this->disk)->copy(
            $file->s3_path,
            $archivePath
        );

        if ($success) {
            // Set archive storage class
            $s3Client = Storage::disk($this->disk)->getDriver()->getAdapter()->getClient();

            $s3Client->copyObject([
                'Bucket' => config('filesystems.disks.s3.bucket'),
                'Key' => $archivePath,
                'CopySource' => config('filesystems.disks.s3.bucket') . '/' . $file->s3_path,
                'StorageClass' => 'GLACIER',
                'ServerSideEncryption' => 'AES256',
            ]);

            $file->update([
                'archive_path' => $archivePath,
                'archived_at' => now(),
            ]);

            return true;
        }

        return false;
    }

    public function deleteFromS3(S3File $file): bool
    {
        $deleted = Storage::disk($this->disk)->delete($file->s3_path);

        if ($deleted) {
            $file->update([
                'deleted_at' => now(),
                'deleted_by' => auth()->id(),
            ]);
        }

        return $deleted;
    }

    protected function generateDirectory(): string
    {
        return sprintf(
            'uploads/%s/%s',
            auth()->id(),
            date('Y/m/d')
        );
    }

    protected function generateS3Filename(UploadedFile $file, array $options): string
    {
        $prefix = $options['prefix'] ?? '';
        $suffix = $options['suffix'] ?? Str::random(8);
        $extension = $file->getClientOriginalExtension();

        return collect([
            $prefix,
            time(),
            $suffix
        ])->filter()->implode('_') . '.' . $extension;
    }

    protected function sanitizeFilename(string $filename): string
    {
        // Remove dangerous characters for S3
        return preg_replace('/[^a-zA-Z0-9._-]/', '_', $filename);
    }
}

class S3Controller extends Controller
{
    public function __construct(protected S3FileService $s3Service) {}

    public function upload(Request $request)
    {
        $request->validate([
            'file' => 'required|file|max:102400', // 100MB
            'visibility' => 'sometimes|in:public,private',
            'storage_class' => 'sometimes|in:STANDARD,REDUCED_REDUNDANCY,GLACIER',
        ]);

        $result = $this->s3Service->uploadToS3(
            $request->file('file'),
            [
                'visibility' => $request->input('visibility', 'private'),
                'storage_class' => $request->input('storage_class', 'STANDARD'),
                'metadata' => [
                    'uploaded_by' => auth()->user()->name,
                    'uploaded_at' => now()->toISOString(),
                ],
            ]
        );

        return response()->json($result);
    }

    public function getTemporaryUrl(S3File $file, Request $request)
    {
        // Verify user has access to this file
        $this->authorize('access', $file);

        $expirationMinutes = $request->input('expires_in', 60);
        $url = $this->s3Service->generateTemporaryUrl($file, $expirationMinutes);

        return response()->json([
            'temporary_url' => $url,
            'expires_at' => now()->addMinutes($expirationMinutes),
        ]);
    }

    public function getPresignedUploadUrl(Request $request)
    {
        $request->validate([
            'filename' => 'required|string|max:255',
            'content_type' => 'required|string',
        ]);

        $uploadData = $this->s3Service->generatePresignedUploadUrl(
            $request->input('filename'),
            $request->input('content_type')
        );

        return response()->json($uploadData);
    }
}
```

### 6. Safe File Replacement

**Problem**: Replace existing files safely without breaking concurrent access or losing data.

```php
<?php

class SafeFileReplacementService
{
    public function replaceFile(string $fileId, UploadedFile $newFile): array
    {
        $existingFile = File::findOrFail($fileId);

        // Verify user can replace this file
        $this->authorize('replace', $existingFile);

        // Validate new file
        $this->validateReplacement($newFile, $existingFile);

        return DB::transaction(function () use ($existingFile, $newFile) {
            // Create backup before replacement
            $backupPath = $this->createBackup($existingFile);

            try {
                // Store new file with temporary name
                $tempPath = $this->storeTempFile($newFile, $existingFile);

                // Atomic replacement
                $this->performAtomicReplacement($existingFile, $tempPath);

                // Update file metadata
                $updatedFile = $this->updateFileMetadata($existingFile, $newFile);

                // Clean up backup after successful replacement
                $this->scheduleBackupCleanup($backupPath);

                return [
                    'success' => true,
                    'file_id' => $updatedFile->id,
                    'message' => 'File replaced successfully',
                    'backup_path' => $backupPath,
                ];

            } catch (\Exception $e) {
                // Restore from backup on failure
                $this->restoreFromBackup($existingFile, $backupPath);
                throw $e;
            }
        });
    }

    protected function createBackup(File $file): string
    {
        $backupDirectory = 'backups/' . date('Y/m/d');
        $backupFilename = pathinfo($file->path, PATHINFO_FILENAME) .
                         '_backup_' . time() . '.' .
                         pathinfo($file->path, PATHINFO_EXTENSION);
        $backupPath = $backupDirectory . '/' . $backupFilename;

        // Copy original file to backup location
        Storage::disk('private')->copy($file->path, $backupPath);

        // Store backup reference
        FileBackup::create([
            'file_id' => $file->id,
            'backup_path' => $backupPath,
            'original_path' => $file->path,
            'created_by' => auth()->id(),
        ]);

        return $backupPath;
    }

    protected function storeTempFile(UploadedFile $newFile, File $existingFile): string
    {
        $tempDirectory = dirname($existingFile->path) . '/temp';
        $tempFilename = 'temp_' . time() . '_' . $newFile->getClientOriginalName();

        return $newFile->storeAs($tempDirectory, $tempFilename, 'private');
    }

    protected function performAtomicReplacement(File $existingFile, string $tempPath): void
    {
        $storage = Storage::disk('private');

        // Move temp file to final location (atomic operation)
        if (!$storage->move($tempPath, $existingFile->path)) {
            throw new \RuntimeException('Failed to replace file atomically');
        }
    }

    protected function updateFileMetadata(File $existingFile, UploadedFile $newFile): File
    {
        $existingFile->update([
            'original_name' => $newFile->getClientOriginalName(),
            'file_size' => $newFile->getSize(),
            'mime_type' => $newFile->getMimeType(),
            'updated_at' => now(),
            'updated_by' => auth()->id(),
            'version' => $existingFile->version + 1,
        ]);

        // Log file replacement
        FileActivity::create([
            'file_id' => $existingFile->id,
            'action' => 'replaced',
            'user_id' => auth()->id(),
            'metadata' => [
                'old_size' => $existingFile->getOriginal('file_size'),
                'new_size' => $newFile->getSize(),
                'version' => $existingFile->version,
            ],
        ]);

        return $existingFile->fresh();
    }

    protected function restoreFromBackup(File $file, string $backupPath): void
    {
        Storage::disk('private')->copy($backupPath, $file->path);

        \Log::warning('File replacement failed, restored from backup', [
            'file_id' => $file->id,
            'backup_path' => $backupPath,
            'user_id' => auth()->id(),
        ]);
    }

    protected function scheduleBackupCleanup(string $backupPath): void
    {
        // Schedule backup cleanup job for later
        CleanupBackupJob::dispatch($backupPath)->delay(now()->addDays(7));
    }

    protected function validateReplacement(UploadedFile $newFile, File $existingFile): void
    {
        // Ensure new file meets requirements
        validator(['file' => $newFile], [
            'file' => [
                'required',
                'file',
                'max:' . (config('app.max_file_size') ?: 10240),
                function ($attribute, $value, $fail) use ($existingFile) {
                    // Optional: Ensure file type consistency
                    if ($this->shouldEnforceTypeConsistency()) {
                        $existingType = pathinfo($existingFile->original_name, PATHINFO_EXTENSION);
                        $newType = $value->getClientOriginalExtension();

                        if (strtolower($existingType) !== strtolower($newType)) {
                            $fail('New file must be the same type as the existing file.');
                        }
                    }
                },
            ]
        ])->validate();
    }

    protected function shouldEnforceTypeConsistency(): bool
    {
        return config('files.enforce_type_consistency', false);
    }

    public function listFileVersions(File $file): array
    {
        $backups = FileBackup::where('file_id', $file->id)
                            ->orderBy('created_at', 'desc')
                            ->get();

        $versions = [];
        foreach ($backups as $backup) {
            $versions[] = [
                'version' => $backup->id,
                'created_at' => $backup->created_at,
                'created_by' => $backup->creator->name ?? 'Unknown',
                'restore_url' => route('files.restore-version', $backup->id),
            ];
        }

        return $versions;
    }
}

class FileReplacementController extends Controller
{
    public function __construct(protected SafeFileReplacementService $replacementService) {}

    public function replace(Request $request, File $file)
    {
        if (!$request->hasFile('replacement')) {
            return response()->json(['error' => 'No replacement file provided'], 400);
        }

        try {
            $result = $this->replacementService->replaceFile(
                $file->id,
                $request->file('replacement')
            );

            return response()->json($result);

        } catch (\Exception $e) {
            return response()->json([
                'error' => 'Failed to replace file: ' . $e->getMessage()
            ], 500);
        }
    }

    public function versions(File $file)
    {
        $this->authorize('view', $file);

        $versions = $this->replacementService->listFileVersions($file);

        return response()->json(['versions' => $versions]);
    }
}
```

---

## When NOT to Use Laravel's File Handling

### Large File Uploads and Streaming

**Problem**: Laravel's default file handling isn't suitable for very large files (>100MB) due to memory and timeout limitations.

**When to avoid Laravel's built-in handling:**

- Files larger than your server's memory limit
- Video files, large datasets, or backup files
- Applications requiring upload progress tracking
- Mobile applications with unreliable connections

**Better alternatives:**

```php
<?php

// BAD: Using Laravel for large file uploads
class LargeFileController extends Controller
{
    public function upload(Request $request)
    {
        // This will fail for large files (memory/timeout issues)
        $request->validate(['file' => 'required|file|max:2097152']); // 2GB - will fail

        $path = $request->file('file')->store('uploads', 's3');

        return response()->json(['path' => $path]);
    }
}

// GOOD: Use chunked uploads with external service
class ChunkedUploadService
{
    public function initializeUpload(string $filename, int $fileSize): array
    {
        $uploadId = Str::uuid();

        // Use AWS S3 Multipart Upload
        $s3Client = AWS::createClient('s3');

        $result = $s3Client->createMultipartUpload([
            'Bucket' => config('filesystems.disks.s3.bucket'),
            'Key' => 'large-uploads/' . $uploadId . '/' . $filename,
            'ContentType' => 'application/octet-stream',
        ]);

        ChunkedUpload::create([
            'upload_id' => $uploadId,
            'aws_upload_id' => $result['UploadId'],
            'filename' => $filename,
            'file_size' => $fileSize,
            'user_id' => auth()->id(),
            'status' => 'initialized',
        ]);

        return [
            'upload_id' => $uploadId,
            'chunk_size' => 5 * 1024 * 1024, // 5MB chunks
            'total_chunks' => ceil($fileSize / (5 * 1024 * 1024)),
        ];
    }

    public function uploadChunk(string $uploadId, int $chunkNumber, string $chunkData): array
    {
        $upload = ChunkedUpload::where('upload_id', $uploadId)->firstOrFail();

        $s3Client = AWS::createClient('s3');

        $result = $s3Client->uploadPart([
            'Bucket' => config('filesystems.disks.s3.bucket'),
            'Key' => 'large-uploads/' . $uploadId . '/' . $upload->filename,
            'PartNumber' => $chunkNumber,
            'UploadId' => $upload->aws_upload_id,
            'Body' => $chunkData,
        ]);

        // Store part ETag for completion
        ChunkedUploadPart::create([
            'upload_id' => $uploadId,
            'part_number' => $chunkNumber,
            'etag' => $result['ETag'],
        ]);

        return [
            'chunk_number' => $chunkNumber,
            'etag' => $result['ETag'],
            'status' => 'uploaded',
        ];
    }

    public function completeUpload(string $uploadId): array
    {
        $upload = ChunkedUpload::where('upload_id', $uploadId)->firstOrFail();
        $parts = $upload->parts()->orderBy('part_number')->get();

        $s3Client = AWS::createClient('s3');

        $result = $s3Client->completeMultipartUpload([
            'Bucket' => config('filesystems.disks.s3.bucket'),
            'Key' => 'large-uploads/' . $uploadId . '/' . $upload->filename,
            'UploadId' => $upload->aws_upload_id,
            'MultipartUpload' => [
                'Parts' => $parts->map(function ($part) {
                    return [
                        'ETag' => $part->etag,
                        'PartNumber' => $part->part_number,
                    ];
                })->toArray(),
            ],
        ]);

        $upload->update([
            'status' => 'completed',
            'completed_at' => now(),
            's3_location' => $result['Location'],
        ]);

        return [
            'upload_id' => $uploadId,
            'location' => $result['Location'],
            'status' => 'completed',
        ];
    }
}
```

### Heavy Processing in Controllers

**Problem**: Performing image/video processing or file transformations directly in controllers blocks request handling and can cause timeouts.

```php
<?php

// BAD: Heavy processing in controller
class ImageController extends Controller
{
    public function upload(Request $request)
    {
        $image = $request->file('image');

        // This blocks the request and can timeout
        $processed = Image::make($image)
            ->resize(1920, 1080)
            ->sharpen(10)
            ->encode('jpg', 85);

        $thumbnail = Image::make($image)
            ->fit(300, 300)
            ->encode('jpg', 90);

        // Multiple slow operations...

        return response()->json(['success' => true]);
    }
}

// GOOD: Use queued jobs for processing
class ImageUploadService
{
    public function queueImageProcessing(UploadedFile $image, User $user): array
    {
        // Store original image quickly
        $originalPath = $image->store('images/originals', 'private');

        // Create database record
        $imageRecord = Image::create([
            'user_id' => $user->id,
            'original_path' => $originalPath,
            'original_name' => $image->getClientOriginalName(),
            'processing_status' => 'queued',
        ]);

        // Queue processing jobs
        ProcessImageJob::dispatch($imageRecord->id)
                      ->onQueue('image-processing');

        GenerateThumbnailJob::dispatch($imageRecord->id)
                           ->onQueue('thumbnails');

        return [
            'id' => $imageRecord->id,
            'status' => 'processing',
            'estimated_completion' => now()->addMinutes(5),
        ];
    }
}

class ProcessImageJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(protected int $imageId) {}

    public function handle(): void
    {
        $image = Image::findOrFail($this->imageId);

        try {
            // Perform heavy processing
            $processedImage = InterventionImage::make(
                Storage::disk('private')->path($image->original_path)
            );

            // Multiple processing steps...
            $processedImage->resize(1920, 1080)
                          ->sharpen(10)
                          ->encode('jpg', 85);

            // Store processed version
            $processedPath = 'images/processed/' . basename($image->original_path);
            Storage::disk('private')->put($processedPath, $processedImage->stream());

            $image->update([
                'processed_path' => $processedPath,
                'processing_status' => 'completed',
                'processed_at' => now(),
            ]);

        } catch (\Exception $e) {
            $image->update(['processing_status' => 'failed']);
            throw $e;
        }
    }
}
```

### External File Service Integration

**When to use external services instead of Laravel's file handling:**

```php
<?php

// Use external services for:
class ExternalFileService
{
    // Cloudflare R2 for cost-effective storage
    public function uploadToCloudflareR2(UploadedFile $file): array
    {
        $s3Client = new S3Client([
            'version' => 'latest',
            'region' => 'auto',
            'endpoint' => 'https://your-account.r2.cloudflarestorage.com',
            'credentials' => [
                'key' => config('services.cloudflare.r2.key'),
                'secret' => config('services.cloudflare.r2.secret'),
            ],
        ]);

        // R2 is S3-compatible but much cheaper
        return $this->uploadToS3Compatible($s3Client, $file);
    }

    // Firebase Storage for real-time applications
    public function uploadToFirebase(UploadedFile $file): array
    {
        $firebase = Firebase::createWithServiceAccount(
            config('services.firebase.credentials')
        );

        $storage = $firebase->createStorage();
        $bucket = $storage->getBucket();

        $object = $bucket->upload(
            fopen($file->getRealPath(), 'r'),
            [
                'name' => 'uploads/' . Str::uuid() . '.' . $file->getClientOriginalExtension(),
                'metadata' => [
                    'contentType' => $file->getMimeType(),
                ]
            ]
        );

        return [
            'firebase_url' => $object->signedUrl(now()->addHour()),
            'path' => $object->name(),
        ];
    }

    // Wasabi for archive storage
    public function archiveToWasabi(string $filePath): bool
    {
        $wasabiClient = new S3Client([
            'version' => 'latest',
            'region' => 'us-east-1',
            'endpoint' => 'https://s3.wasabisys.com',
            'credentials' => [
                'key' => config('services.wasabi.key'),
                'secret' => config('services.wasabi.secret'),
            ],
        ]);

        // Move to cheaper long-term storage
        return $this->transferToArchive($wasabiClient, $filePath);
    }
}
```

---

## Senior Developer Best Practices

### Architecture and Code Organization

**Keep Controllers Thin**

```php
<?php

// BAD: Fat controller with mixed responsibilities
class DocumentController extends Controller
{
    public function store(Request $request)
    {
        // Validation logic
        $request->validate([
            'document' => 'required|file|mimes:pdf,doc|max:10240',
            'category' => 'required|in:contract,invoice,report',
        ]);

        // File processing logic
        $file = $request->file('document');
        $filename = time() . '_' . $file->getClientOriginalName();
        $path = $file->storeAs('documents', $filename, 'private');

        // Business logic
        $document = Document::create([
            'user_id' => auth()->id(),
            'filename' => $filename,
            'path' => $path,
            'category' => $request->category,
        ]);

        // Notification logic
        Mail::to(auth()->user())->send(new DocumentUploaded($document));

        // Logging logic
        Log::info('Document uploaded', ['document_id' => $document->id]);

        return response()->json(['document' => $document]);
    }
}

// GOOD: Separated concerns with service classes
class DocumentController extends Controller
{
    public function __construct(
        protected DocumentUploadService $uploadService,
        protected DocumentNotificationService $notificationService
    ) {}

    public function store(DocumentUploadRequest $request)
    {
        $document = $this->uploadService->upload(
            $request->file('document'),
            $request->validated()
        );

        $this->notificationService->notifyUploadComplete($document);

        return new DocumentResource($document);
    }
}

class DocumentUploadService
{
    public function upload(UploadedFile $file, array $metadata): Document
    {
        $path = $this->storeFile($file);
        $document = $this->createDocumentRecord($file, $path, $metadata);
        $this->logUpload($document);

        return $document;
    }

    protected function storeFile(UploadedFile $file): string
    {
        $filename = $this->generateSecureFilename($file);
        return $file->storeAs(
            $this->getUploadDirectory(),
            $filename,
            'private'
        );
    }

    // Additional methods...
}
```

**Use Storage Disk Abstraction**

```php
<?php

// BAD: Hardcoded storage paths and methods
class FileService
{
    public function saveFile(UploadedFile $file): string
    {
        // Hardcoded to local storage
        return $file->store('uploads', 'local');
    }

    public function getFileContent(string $path): string
    {
        // Directly accessing filesystem
        return file_get_contents(storage_path('app/' . $path));
    }
}

// GOOD: Storage abstraction with configurable disks
class FileService
{
    public function __construct(protected string $defaultDisk = null)
    {
        $this->defaultDisk = $defaultDisk ?? config('filesystems.default');
    }

    public function saveFile(UploadedFile $file, string $disk = null): string
    {
        $disk = $disk ?? $this->defaultDisk;
        return $file->store('uploads', $disk);
    }

    public function getFileContent(string $path, string $disk = null): string
    {
        $disk = $disk ?? $this->defaultDisk;
        return Storage::disk($disk)->get($path);
    }

    public function moveFileBetweenDisks(string $path, string $fromDisk, string $toDisk): string
    {
        $content = Storage::disk($fromDisk)->get($path);
        $newPath = Storage::disk($toDisk)->put($path, $content);
        Storage::disk($fromDisk)->delete($path);

        return $newPath;
    }
}
```

### Security Best Practices

**Never Store User Files in Public Directory**

```php
<?php

// BAD: Storing sensitive files in public directory
class BadFileController extends Controller
{
    public function store(Request $request)
    {
        // Anyone can access this file directly via URL!
        $path = $request->file('document')->store('documents', 'public');

        return response()->json(['url' => Storage::url($path)]);
    }
}

// GOOD: Store sensitive files privately with controlled access
class SecureFileController extends Controller
{
    public function store(Request $request)
    {
        // Store privately - requires authentication to access
        $path = $request->file('document')->store('documents', 'private');

        $document = Document::create([
            'user_id' => auth()->id(),
            'path' => $path,
            'access_level' => $request->input('access_level', 'private'),
        ]);

        return response()->json([
            'id' => $document->id,
            'download_url' => route('documents.download', $document),
        ]);
    }

    public function download(Document $document)
    {
        // Verify access permissions
        $this->authorize('download', $document);

        return Storage::disk('private')->download(
            $document->path,
            $document->original_filename
        );
    }
}
```

**Input Validation and Sanitization**

```php
<?php

class SecureUploadRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'file' => [
                'required',
                'file',
                'max:10240', // 10MB
                'mimes:pdf,doc,docx,jpg,png',
                function ($attribute, $value, $fail) {
                    // Validate file content, not just extension
                    if (!$this->isValidFileType($value)) {
                        $fail('Invalid file content detected.');
                    }

                    // Check for malicious content
                    if ($this->containsMaliciousContent($value)) {
                        $fail('Potentially malicious file detected.');
                    }
                },
            ],
            'filename' => [
                'sometimes',
                'string',
                'max:255',
                'regex:/^[a-zA-Z0-9._-]+$/', // Only safe characters
            ],
        ];
    }

    protected function isValidFileType(UploadedFile $file): bool
    {
        // Use finfo to check actual file content
        $finfo = finfo_open(FILEINFO_MIME_TYPE);
        $mimeType = finfo_file($finfo, $file->getRealPath());
        finfo_close($finfo);

        $allowedMimes = [
            'application/pdf',
            'application/msword',
            'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            'image/jpeg',
            'image/png',
        ];

        return in_array($mimeType, $allowedMimes);
    }

    protected function containsMaliciousContent(UploadedFile $file): bool
    {
        $content = file_get_contents($file->getRealPath());

        // Check for script injection attempts
        $maliciousPatterns = [
            '/<script/i',
            '/<\?php/i',
            '/javascript:/i',
            '/vbscript:/i',
            '/onload=/i',
            '/onerror=/i',
        ];

        foreach ($maliciousPatterns as $pattern) {
            if (preg_match($pattern, $content)) {
                return true;
            }
        }

        return false;
    }
}
```

### Performance Optimization

**Optimize File Access Patterns**

```php
<?php

class OptimizedFileService
{
    // Cache file metadata to avoid repeated disk operations
    public function getFileInfo(string $path, string $disk = 'private'): array
    {
        return Cache::remember(
            "file_info_{$disk}_{$path}",
            now()->addMinutes(30),
            function () use ($path, $disk) {
                $storage = Storage::disk($disk);

                return [
                    'exists' => $storage->exists($path),
                    'size' => $storage->size($path),
                    'last_modified' => $storage->lastModified($path),
                    'mime_type' => $storage->mimeType($path),
                ];
            }
        );
    }

    // Use streaming for large files
    public function streamLargeFile(string $path, string $disk = 'private'): StreamedResponse
    {
        return Storage::disk($disk)->response($path, null, [
            'Cache-Control' => 'no-cache, no-store, must-revalidate',
            'Pragma' => 'no-cache',
            'Expires' => '0',
        ]);
    }

    // Batch file operations for efficiency
    public function deleteMultipleFiles(array $paths, string $disk = 'private'): array
    {
        $storage = Storage::disk($disk);
        $results = [];

        // Group operations to reduce overhead
        $validPaths = array_filter($paths, fn($path) => $storage->exists($path));

        if (!empty($validPaths)) {
            $storage->delete($validPaths);
            $results['deleted'] = count($validPaths);
        }

        $results['not_found'] = count($paths) - count($validPaths);

        return $results;
    }
}
```

**Use Signed URLs for Performance**

```php
<?php

class PerformantFileService
{
    public function getOptimizedFileUrl(File $file, int $expirationMinutes = 60): string
    {
        // For S3/CDN files, return direct temporary URL to bypass application
        if ($file->stored_on_s3) {
            return Storage::disk('s3')->temporaryUrl(
                $file->path,
                now()->addMinutes($expirationMinutes)
            );
        }

        // For local files, use signed routes
        return URL::temporarySignedRoute(
            'files.download',
            now()->addMinutes($expirationMinutes),
            ['file' => $file->id]
        );
    }

    public function generateBulkDownloadUrls(Collection $files): array
    {
        // Generate all URLs in batch to reduce overhead
        return $files->mapWithKeys(function (File $file) {
            return [
                $file->id => $this->getOptimizedFileUrl($file, 120) // 2 hour expiry
            ];
        })->toArray();
    }
}
```

### Error Handling and Monitoring

```php
<?php

class RobustFileService
{
    public function uploadWithRetry(UploadedFile $file, int $maxRetries = 3): array
    {
        $attempt = 0;
        $lastException = null;

        while ($attempt < $maxRetries) {
            try {
                $path = $file->store('uploads', 'private');

                // Verify upload succeeded
                if (!Storage::disk('private')->exists($path)) {
                    throw new \RuntimeException('File upload verification failed');
                }

                return [
                    'success' => true,
                    'path' => $path,
                    'attempts' => $attempt + 1,
                ];

            } catch (\Exception $e) {
                $attempt++;
                $lastException = $e;

                Log::warning('File upload attempt failed', [
                    'attempt' => $attempt,
                    'max_retries' => $maxRetries,
                    'error' => $e->getMessage(),
                    'file_name' => $file->getClientOriginalName(),
                ]);

                if ($attempt < $maxRetries) {
                    sleep(1); // Brief delay before retry
                }
            }
        }

        // All retries failed
        Log::error('File upload failed after all retries', [
            'file_name' => $file->getClientOriginalName(),
            'final_error' => $lastException->getMessage(),
        ]);

        throw new FileUploadException(
            'Upload failed after ' . $maxRetries . ' attempts: ' . $lastException->getMessage(),
            0,
            $lastException
        );
    }

    public function monitorDiskUsage(): void
    {
        $disks = ['local', 'private', 's3'];

        foreach ($disks as $diskName) {
            try {
                $disk = Storage::disk($diskName);

                // Monitor local disk space
                if (in_array($diskName, ['local', 'private'])) {
                    $freeSpace = disk_free_space($disk->path(''));
                    $totalSpace = disk_total_space($disk->path(''));
                    $usedPercentage = (($totalSpace - $freeSpace) / $totalSpace) * 100;

                    if ($usedPercentage > 85) {
                        Log::warning('High disk usage detected', [
                            'disk' => $diskName,
                            'used_percentage' => round($usedPercentage, 2),
                        ]);
                    }
                }

            } catch (\Exception $e) {
                Log::error('Disk monitoring failed', [
                    'disk' => $diskName,
                    'error' => $e->getMessage(),
                ]);
            }
        }
    }
}
```

---

## Testing File Uploads

### Using Storage Fake for Testing

Laravel provides `Storage::fake()` to mock filesystem operations during testing:

```php
<?php

use Illuminate\Http\Testing\File;
use Illuminate\Support\Facades\Storage;

class FileUploadTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        // Fake the storage disk
        Storage::fake('private');
        Storage::fake('public');
    }

    /** @test */
    public function it_uploads_file_successfully()
    {
        // Create a fake file
        $file = File::create('test-document.pdf', 1000); // 1000 kilobytes

        $response = $this->actingAs($this->user)
                         ->postJson('/api/documents', [
                             'document' => $file,
                             'category' => 'reports',
                         ]);

        $response->assertSuccessful();

        // Assert file was stored
        Storage::disk('private')->assertExists(
            'documents/' . $this->user->id . '/' . $file->hashName()
        );

        // Assert database record created
        $this->assertDatabaseHas('documents', [
            'user_id' => $this->user->id,
            'original_name' => 'test-document.pdf',
            'category' => 'reports',
        ]);
    }

    /** @test */
    public function it_validates_file_type()
    {
        $invalidFile = File::create('malicious.exe', 100);

        $response = $this->actingAs($this->user)
                         ->postJson('/api/documents', [
                             'document' => $invalidFile,
                         ]);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['document']);

        // Assert file was not stored
        Storage::disk('private')->assertMissing($invalidFile->hashName());
    }

    /** @test */
    public function it_handles_file_size_limits()
    {
        $largeFile = File::create('large-file.pdf', 50000); // 50MB

        $response = $this->actingAs($this->user)
                         ->postJson('/api/documents', [
                             'document' => $largeFile,
                         ]);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['document']);
    }

    /** @test */
    public function it_cleans_up_old_files()
    {
        $oldFile = File::create('old-document.pdf', 100);
        $newFile = File::create('new-document.pdf', 100);

        // Upload old file first
        $this->user->documents()->create([
            'path' => $oldFile->store('documents', 'private'),
            'original_name' => 'old-document.pdf',
        ]);

        // Upload new file (should replace old one)
        $this->actingAs($this->user)
             ->postJson('/api/documents/replace/' . $this->user->documents->first()->id, [
                 'document' => $newFile,
             ]);

        // Assert old file was deleted
        Storage::disk('private')->assertMissing($oldFile->hashName());
        // Assert new file exists
        Storage::disk('private')->assertExists($newFile->hashName());
    }
}
```

### Testing File Download and Access

```php
<?php

class FileDownloadTest extends TestCase
{
    /** @test */
    public function it_allows_authorized_user_to_download_file()
    {
        Storage::fake('private');

        $file = File::create('test.pdf', 100);
        $storedPath = $file->store('documents', 'private');

        $document = Document::factory()->create([
            'user_id' => $this->user->id,
            'path' => $storedPath,
            'original_name' => 'test.pdf',
        ]);

        $response = $this->actingAs($this->user)
                         ->get(route('documents.download', $document));

        $response->assertSuccessful()
                 ->assertHeader('content-type', 'application/pdf')
                 ->assertHeader('content-disposition', 'attachment; filename="test.pdf"');
    }

    /** @test */
    public function it_denies_unauthorized_access()
    {
        $otherUser = User::factory()->create();
        $document = Document::factory()->create(['user_id' => $otherUser->id]);

        $response = $this->actingAs($this->user)
                         ->get(route('documents.download', $document));

        $response->assertForbidden();
    }

    /** @test */
    public function it_generates_valid_signed_urls()
    {
        Storage::fake('private');

        $document = Document::factory()->create(['user_id' => $this->user->id]);

        $signedUrl = URL::temporarySignedRoute(
            'documents.signed-download',
            now()->addHour(),
            ['document' => $document->id]
        );

        $response = $this->get($signedUrl);

        $response->assertSuccessful();
    }
}
```

### Testing S3 Integration

```php
<?php

class S3FileTest extends TestCase
{
    /** @test */
    public function it_uploads_to_s3_successfully()
    {
        // Mock
```
