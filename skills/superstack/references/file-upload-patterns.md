# File Upload Patterns — Supabase Storage + Next.js

Production-ready patterns for file uploads with Supabase Storage, drag-and-drop, compression, and security.

---

## 1. Supabase Storage Setup

### Bucket Creation (Supabase Dashboard or Migration)

```sql
-- Create storage buckets
INSERT INTO storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
VALUES
  ('avatars', 'avatars', true, 2097152, ARRAY['image/jpeg', 'image/png', 'image/webp']),
  ('documents', 'documents', false, 10485760, ARRAY['application/pdf', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document']),
  ('uploads', 'uploads', false, 52428800, NULL);

-- RLS: Authenticated users can upload to their own folder
CREATE POLICY "Users can upload own files"
ON storage.objects FOR INSERT
TO authenticated
WITH CHECK (
  bucket_id = 'uploads'
  AND (storage.foldername(name))[1] = auth.uid()::text
);

-- RLS: Users can read their own files
CREATE POLICY "Users can read own files"
ON storage.objects FOR SELECT
TO authenticated
USING (
  bucket_id = 'uploads'
  AND (storage.foldername(name))[1] = auth.uid()::text
);

-- RLS: Users can delete their own files
CREATE POLICY "Users can delete own files"
ON storage.objects FOR DELETE
TO authenticated
USING (
  bucket_id = 'uploads'
  AND (storage.foldername(name))[1] = auth.uid()::text
);

-- RLS: Public bucket — anyone can read
CREATE POLICY "Public read access"
ON storage.objects FOR SELECT
TO public
USING (bucket_id = 'avatars');

-- RLS: Authenticated users can upload avatars to their folder
CREATE POLICY "Users can upload avatars"
ON storage.objects FOR INSERT
TO authenticated
WITH CHECK (
  bucket_id = 'avatars'
  AND (storage.foldername(name))[1] = auth.uid()::text
);
```

### Client Upload Helper

```typescript
// lib/storage.ts
import { createClient } from "@/lib/supabase/client";

export async function uploadFile(
  bucket: string,
  path: string,
  file: File,
  options?: { upsert?: boolean; contentType?: string }
) {
  const supabase = createClient();

  const { data, error } = await supabase.storage
    .from(bucket)
    .upload(path, file, {
      cacheControl: "3600",
      upsert: options?.upsert ?? false,
      contentType: options?.contentType,
    });

  if (error) throw error;
  return data;
}

export function getPublicUrl(bucket: string, path: string) {
  const supabase = createClient();
  const { data } = supabase.storage.from(bucket).getPublicUrl(path);
  return data.publicUrl;
}

export async function getSignedUrl(
  bucket: string,
  path: string,
  expiresIn = 3600
) {
  const supabase = createClient();
  const { data, error } = await supabase.storage
    .from(bucket)
    .createSignedUrl(path, expiresIn);

  if (error) throw error;
  return data.signedUrl;
}

export async function deleteFile(bucket: string, paths: string[]) {
  const supabase = createClient();
  const { error } = await supabase.storage.from(bucket).remove(paths);
  if (error) throw error;
}
```

---

## 2. Presigned URLs (Server-Generated Upload)

### Server Action

```typescript
// app/actions/upload.ts
"use server";

import { createClient } from "@/lib/supabase/server";
import { auth } from "@/lib/auth";
import crypto from "crypto";

export async function createUploadUrl(
  bucket: string,
  filename: string,
  contentType: string
) {
  const session = await auth();
  if (!session?.user?.id) throw new Error("Unauthorized");

  const supabase = await createClient();
  const ext = filename.split(".").pop();
  const uniqueName = `${session.user.id}/${crypto.randomUUID()}.${ext}`;

  const { data, error } = await supabase.storage
    .from(bucket)
    .createSignedUploadUrl(uniqueName);

  if (error) throw error;

  return {
    signedUrl: data.signedUrl,
    token: data.token,
    path: uniqueName,
  };
}
```

### Client Direct Upload

```typescript
"use client";

import { createUploadUrl } from "@/app/actions/upload";

export async function directUpload(file: File, bucket: string) {
  // 1. Get presigned URL from server
  const { signedUrl, path } = await createUploadUrl(
    bucket,
    file.name,
    file.type
  );

  // 2. Upload directly to storage (bypasses your server)
  const response = await fetch(signedUrl, {
    method: "PUT",
    headers: {
      "Content-Type": file.type,
    },
    body: file,
  });

  if (!response.ok) {
    throw new Error(`Upload failed: ${response.statusText}`);
  }

  return { path };
}
```

---

## 3. Drag-and-Drop

### Install

```bash
bun add react-dropzone
```

### Dropzone Component

```typescript
"use client";

import { useCallback, useState } from "react";
import { useDropzone, type FileRejection } from "react-dropzone";
import { Upload, X, FileIcon } from "lucide-react";
import { cn } from "@/lib/utils";

interface FileUploadProps {
  onFilesAccepted: (files: File[]) => void;
  maxFiles?: number;
  maxSize?: number; // bytes
  accept?: Record<string, string[]>;
  className?: string;
}

export function FileUpload({
  onFilesAccepted,
  maxFiles = 5,
  maxSize = 10 * 1024 * 1024, // 10MB
  accept = {
    "image/*": [".png", ".jpg", ".jpeg", ".webp"],
    "application/pdf": [".pdf"],
  },
  className,
}: FileUploadProps) {
  const [errors, setErrors] = useState<string[]>([]);

  const onDrop = useCallback(
    (acceptedFiles: File[], rejections: FileRejection[]) => {
      setErrors([]);

      if (rejections.length > 0) {
        const errorMessages = rejections.map((r) => {
          const name = r.file.name;
          const reasons = r.errors.map((e) => e.message).join(", ");
          return `${name}: ${reasons}`;
        });
        setErrors(errorMessages);
      }

      if (acceptedFiles.length > 0) {
        onFilesAccepted(acceptedFiles);
      }
    },
    [onFilesAccepted]
  );

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    maxFiles,
    maxSize,
    accept,
  });

  return (
    <div className={className}>
      <div
        {...getRootProps()}
        className={cn(
          "flex cursor-pointer flex-col items-center justify-center rounded-lg border-2 border-dashed p-8 transition-colors",
          isDragActive
            ? "border-primary bg-primary/5"
            : "border-muted-foreground/25 hover:border-primary/50"
        )}
      >
        <input {...getInputProps()} />
        <Upload className="mb-2 h-8 w-8 text-muted-foreground" />
        {isDragActive ? (
          <p className="text-sm text-primary">Drop files here...</p>
        ) : (
          <div className="text-center">
            <p className="text-sm font-medium">
              Drag & drop files here, or click to select
            </p>
            <p className="mt-1 text-xs text-muted-foreground">
              Max {maxFiles} files, up to {Math.round(maxSize / 1024 / 1024)}MB
              each
            </p>
          </div>
        )}
      </div>

      {errors.length > 0 && (
        <div className="mt-2 space-y-1">
          {errors.map((err, i) => (
            <p key={i} className="text-sm text-destructive">
              {err}
            </p>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## 4. Image Compression

### Install

```bash
bun add browser-image-compression
```

### Compression Utility

```typescript
// lib/image-compression.ts
import imageCompression from "browser-image-compression";

export interface CompressionOptions {
  maxSizeMB?: number;
  maxWidthOrHeight?: number;
  quality?: number;
  useWebWorker?: boolean;
}

const defaultOptions: CompressionOptions = {
  maxSizeMB: 1,
  maxWidthOrHeight: 1920,
  quality: 0.8,
  useWebWorker: true,
};

export async function compressImage(
  file: File,
  options: CompressionOptions = {}
): Promise<File> {
  // Skip non-images
  if (!file.type.startsWith("image/")) return file;

  // Skip small files (< 100KB)
  if (file.size < 100 * 1024) return file;

  // Skip GIFs (compression breaks animation)
  if (file.type === "image/gif") return file;

  const opts = { ...defaultOptions, ...options };

  const compressed = await imageCompression(file, {
    maxSizeMB: opts.maxSizeMB!,
    maxWidthOrHeight: opts.maxWidthOrHeight!,
    useWebWorker: opts.useWebWorker!,
    fileType: "image/webp", // Convert to WebP for smaller size
  });

  console.log(
    `Compressed ${file.name}: ${(file.size / 1024).toFixed(0)}KB -> ${(compressed.size / 1024).toFixed(0)}KB`
  );

  return compressed;
}

export async function compressImages(
  files: File[],
  options?: CompressionOptions
): Promise<File[]> {
  return Promise.all(files.map((f) => compressImage(f, options)));
}
```

### Usage with Upload

```typescript
"use client";

import { compressImage } from "@/lib/image-compression";
import { uploadFile } from "@/lib/storage";

async function handleUpload(file: File) {
  const compressed = await compressImage(file, {
    maxSizeMB: 0.5,
    maxWidthOrHeight: 1200,
  });

  await uploadFile("uploads", `images/${file.name}`, compressed);
}
```

---

## 5. Upload Progress

### Progress Hook

```typescript
// hooks/use-upload.ts
"use client";

import { useState, useCallback } from "react";

interface UploadState {
  progress: number;
  isUploading: boolean;
  error: string | null;
}

export function useUploadWithProgress() {
  const [state, setState] = useState<UploadState>({
    progress: 0,
    isUploading: false,
    error: null,
  });

  const upload = useCallback(
    async (url: string, file: File, headers?: Record<string, string>) => {
      setState({ progress: 0, isUploading: true, error: null });

      return new Promise<void>((resolve, reject) => {
        const xhr = new XMLHttpRequest();

        xhr.upload.addEventListener("progress", (e) => {
          if (e.lengthComputable) {
            const progress = Math.round((e.loaded / e.total) * 100);
            setState((s) => ({ ...s, progress }));
          }
        });

        xhr.addEventListener("load", () => {
          if (xhr.status >= 200 && xhr.status < 300) {
            setState({ progress: 100, isUploading: false, error: null });
            resolve();
          } else {
            const error = `Upload failed: ${xhr.statusText}`;
            setState({ progress: 0, isUploading: false, error });
            reject(new Error(error));
          }
        });

        xhr.addEventListener("error", () => {
          const error = "Upload failed: network error";
          setState({ progress: 0, isUploading: false, error });
          reject(new Error(error));
        });

        xhr.open("PUT", url);
        if (headers) {
          Object.entries(headers).forEach(([k, v]) =>
            xhr.setRequestHeader(k, v)
          );
        }
        xhr.send(file);
      });
    },
    []
  );

  const reset = useCallback(() => {
    setState({ progress: 0, isUploading: false, error: null });
  }, []);

  return { ...state, upload, reset };
}
```

### Progress Bar Component

```typescript
"use client";

import { cn } from "@/lib/utils";

interface ProgressBarProps {
  progress: number;
  className?: string;
  showLabel?: boolean;
}

export function UploadProgressBar({
  progress,
  className,
  showLabel = true,
}: ProgressBarProps) {
  return (
    <div className={cn("w-full", className)}>
      <div className="h-2 w-full overflow-hidden rounded-full bg-secondary">
        <div
          className="h-full rounded-full bg-primary transition-all duration-300 ease-out"
          style={{ width: `${progress}%` }}
        />
      </div>
      {showLabel && (
        <p className="mt-1 text-xs text-muted-foreground text-end">
          {progress}%
        </p>
      )}
    </div>
  );
}
```

---

## 6. Image Preview

### Preview with FileReader

```typescript
"use client";

import { useState, useCallback } from "react";
import { X } from "lucide-react";

interface FilePreview {
  file: File;
  url: string;
}

export function useFilePreview() {
  const [previews, setPreviews] = useState<FilePreview[]>([]);

  const addFiles = useCallback((files: File[]) => {
    const newPreviews = files
      .filter((f) => f.type.startsWith("image/"))
      .map((file) => ({
        file,
        url: URL.createObjectURL(file),
      }));

    setPreviews((prev) => [...prev, ...newPreviews]);
  }, []);

  const removeFile = useCallback((index: number) => {
    setPreviews((prev) => {
      const removed = prev[index];
      if (removed) URL.revokeObjectURL(removed.url);
      return prev.filter((_, i) => i !== index);
    });
  }, []);

  const clearAll = useCallback(() => {
    previews.forEach((p) => URL.revokeObjectURL(p.url));
    setPreviews([]);
  }, [previews]);

  return { previews, addFiles, removeFile, clearAll };
}
```

### Gallery Grid

```typescript
"use client";

import Image from "next/image";
import { X } from "lucide-react";
import { Button } from "@/components/ui/button";

interface PreviewGalleryProps {
  previews: { file: File; url: string }[];
  onRemove: (index: number) => void;
}

export function PreviewGallery({ previews, onRemove }: PreviewGalleryProps) {
  if (previews.length === 0) return null;

  return (
    <div className="grid grid-cols-2 gap-4 sm:grid-cols-3 md:grid-cols-4">
      {previews.map((preview, index) => (
        <div
          key={preview.url}
          className="group relative aspect-square overflow-hidden rounded-lg border"
        >
          <Image
            src={preview.url}
            alt={preview.file.name}
            fill
            className="object-cover"
          />
          <div className="absolute inset-0 bg-black/40 opacity-0 transition-opacity group-hover:opacity-100" />
          <Button
            size="icon"
            variant="destructive"
            className="absolute right-1 top-1 h-6 w-6 opacity-0 transition-opacity group-hover:opacity-100"
            onClick={() => onRemove(index)}
          >
            <X className="h-3 w-3" />
          </Button>
          <div className="absolute bottom-0 left-0 right-0 bg-black/60 px-2 py-1 opacity-0 group-hover:opacity-100">
            <p className="truncate text-xs text-white">{preview.file.name}</p>
            <p className="text-xs text-white/70">
              {(preview.file.size / 1024).toFixed(0)} KB
            </p>
          </div>
        </div>
      ))}
    </div>
  );
}
```

---

## 7. Avatar Upload

### Install

```bash
bun add react-image-crop
```

### Avatar Upload Component

```typescript
"use client";

import { useState, useRef, useCallback } from "react";
import ReactCrop, { type Crop, centerCrop, makeAspectCrop } from "react-image-crop";
import "react-image-crop/dist/ReactCrop.css";
import { compressImage } from "@/lib/image-compression";
import { uploadFile, getPublicUrl, deleteFile } from "@/lib/storage";
import { Button } from "@/components/ui/button";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import Image from "next/image";
import { Camera } from "lucide-react";

interface AvatarUploadProps {
  currentUrl?: string | null;
  userId: string;
  onUploadComplete: (url: string) => void;
}

export function AvatarUpload({
  currentUrl,
  userId,
  onUploadComplete,
}: AvatarUploadProps) {
  const [selectedFile, setSelectedFile] = useState<string | null>(null);
  const [crop, setCrop] = useState<Crop>();
  const [isOpen, setIsOpen] = useState(false);
  const [isUploading, setIsUploading] = useState(false);
  const imgRef = useRef<HTMLImageElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  function onSelectFile(e: React.ChangeEvent<HTMLInputElement>) {
    if (e.target.files && e.target.files.length > 0) {
      const reader = new FileReader();
      reader.addEventListener("load", () => {
        setSelectedFile(reader.result as string);
        setIsOpen(true);
      });
      reader.readAsDataURL(e.target.files[0]);
    }
  }

  function onImageLoad(e: React.SyntheticEvent<HTMLImageElement>) {
    const { width, height } = e.currentTarget;
    const crop = centerCrop(
      makeAspectCrop({ unit: "%", width: 90 }, 1, width, height),
      width,
      height
    );
    setCrop(crop);
  }

  async function getCroppedImage(): Promise<File | null> {
    const image = imgRef.current;
    if (!image || !crop) return null;

    const canvas = document.createElement("canvas");
    const scaleX = image.naturalWidth / image.width;
    const scaleY = image.naturalHeight / image.height;
    const size = 256; // Output size

    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext("2d")!;

    ctx.drawImage(
      image,
      (crop.x / 100) * image.width * scaleX,
      (crop.y / 100) * image.height * scaleY,
      (crop.width / 100) * image.width * scaleX,
      (crop.height / 100) * image.height * scaleY,
      0,
      0,
      size,
      size
    );

    return new Promise((resolve) => {
      canvas.toBlob(
        (blob) => {
          if (blob) {
            resolve(new File([blob], "avatar.webp", { type: "image/webp" }));
          } else {
            resolve(null);
          }
        },
        "image/webp",
        0.85
      );
    });
  }

  async function handleUpload() {
    setIsUploading(true);
    try {
      const croppedFile = await getCroppedImage();
      if (!croppedFile) return;

      const path = `${userId}/avatar.webp`;

      // Delete existing avatar
      try {
        await deleteFile("avatars", [path]);
      } catch {
        // Ignore if not exists
      }

      await uploadFile("avatars", path, croppedFile, { upsert: true });
      const url = getPublicUrl("avatars", path);
      onUploadComplete(`${url}?t=${Date.now()}`); // Cache bust

      setIsOpen(false);
      setSelectedFile(null);
    } catch (error) {
      console.error("Avatar upload failed:", error);
    } finally {
      setIsUploading(false);
    }
  }

  return (
    <div>
      {/* Avatar Display */}
      <div
        className="group relative h-24 w-24 cursor-pointer overflow-hidden rounded-full border-2"
        onClick={() => inputRef.current?.click()}
      >
        {currentUrl ? (
          <Image src={currentUrl} alt="Avatar" fill className="object-cover" />
        ) : (
          <div className="flex h-full w-full items-center justify-center bg-muted text-2xl font-bold text-muted-foreground">
            ?
          </div>
        )}
        <div className="absolute inset-0 flex items-center justify-center bg-black/40 opacity-0 transition-opacity group-hover:opacity-100">
          <Camera className="h-6 w-6 text-white" />
        </div>
      </div>

      <input
        ref={inputRef}
        type="file"
        accept="image/jpeg,image/png,image/webp"
        onChange={onSelectFile}
        className="hidden"
      />

      {/* Crop Dialog */}
      <Dialog open={isOpen} onOpenChange={setIsOpen}>
        <DialogContent className="max-w-md">
          <DialogHeader>
            <DialogTitle>Crop Avatar</DialogTitle>
          </DialogHeader>
          {selectedFile && (
            <div className="space-y-4">
              <ReactCrop
                crop={crop}
                onChange={(_, pc) => setCrop(pc)}
                aspect={1}
                circularCrop
              >
                <img
                  ref={imgRef}
                  src={selectedFile}
                  alt="Crop preview"
                  onLoad={onImageLoad}
                  className="max-h-[400px] w-full object-contain"
                />
              </ReactCrop>
              <div className="flex justify-end gap-2">
                <Button variant="outline" onClick={() => setIsOpen(false)}>
                  Cancel
                </Button>
                <Button onClick={handleUpload} disabled={isUploading}>
                  {isUploading ? "Uploading..." : "Save"}
                </Button>
              </div>
            </div>
          )}
        </DialogContent>
      </Dialog>
    </div>
  );
}
```

---

## 8. Document Upload

### Document List Component

```typescript
"use client";

import { FileText, FileSpreadsheet, File, Download, Trash2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import { getSignedUrl, deleteFile } from "@/lib/storage";

interface Document {
  id: string;
  name: string;
  path: string;
  size: number;
  type: string;
  uploadedAt: string;
}

const fileIcons: Record<string, typeof FileText> = {
  "application/pdf": FileText,
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document": FileText,
  "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet": FileSpreadsheet,
};

function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / 1024 / 1024).toFixed(1)} MB`;
}

export function DocumentList({
  documents,
  onDelete,
}: {
  documents: Document[];
  onDelete: (doc: Document) => void;
}) {
  async function handleDownload(doc: Document) {
    const url = await getSignedUrl("documents", doc.path, 60);
    window.open(url, "_blank");
  }

  return (
    <div className="divide-y rounded-lg border">
      {documents.map((doc) => {
        const Icon = fileIcons[doc.type] || File;

        return (
          <div
            key={doc.id}
            className="flex items-center gap-3 px-4 py-3 hover:bg-muted/50"
          >
            <Icon className="h-8 w-8 shrink-0 text-muted-foreground" />
            <div className="min-w-0 flex-1">
              <p className="truncate text-sm font-medium">{doc.name}</p>
              <p className="text-xs text-muted-foreground">
                {formatFileSize(doc.size)} — {new Date(doc.uploadedAt).toLocaleDateString()}
              </p>
            </div>
            <div className="flex gap-1">
              <Button
                size="icon"
                variant="ghost"
                onClick={() => handleDownload(doc)}
              >
                <Download className="h-4 w-4" />
              </Button>
              <Button
                size="icon"
                variant="ghost"
                onClick={() => onDelete(doc)}
              >
                <Trash2 className="h-4 w-4 text-destructive" />
              </Button>
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

---

## 9. Bulk Upload with Queue

```typescript
// lib/upload-queue.ts
"use client";

export interface QueueItem {
  id: string;
  file: File;
  status: "pending" | "uploading" | "success" | "error";
  progress: number;
  error?: string;
}

type QueueListener = (items: QueueItem[]) => void;

export class UploadQueue {
  private items: QueueItem[] = [];
  private listeners: Set<QueueListener> = new Set();
  private concurrency: number;
  private activeCount = 0;
  private uploadFn: (file: File, onProgress: (p: number) => void) => Promise<string>;

  constructor(
    uploadFn: (file: File, onProgress: (p: number) => void) => Promise<string>,
    concurrency = 3
  ) {
    this.uploadFn = uploadFn;
    this.concurrency = concurrency;
  }

  subscribe(listener: QueueListener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  private notify() {
    const snapshot = [...this.items];
    this.listeners.forEach((l) => l(snapshot));
  }

  private updateItem(id: string, update: Partial<QueueItem>) {
    this.items = this.items.map((item) =>
      item.id === id ? { ...item, ...update } : item
    );
    this.notify();
  }

  addFiles(files: File[]) {
    const newItems: QueueItem[] = files.map((file) => ({
      id: crypto.randomUUID(),
      file,
      status: "pending" as const,
      progress: 0,
    }));

    this.items = [...this.items, ...newItems];
    this.notify();
    this.processQueue();
  }

  retryFailed() {
    this.items = this.items.map((item) =>
      item.status === "error"
        ? { ...item, status: "pending" as const, progress: 0, error: undefined }
        : item
    );
    this.notify();
    this.processQueue();
  }

  private async processQueue() {
    const pending = this.items.filter((i) => i.status === "pending");

    for (const item of pending) {
      if (this.activeCount >= this.concurrency) break;

      this.activeCount++;
      this.updateItem(item.id, { status: "uploading" });

      try {
        await this.uploadFn(item.file, (progress) => {
          this.updateItem(item.id, { progress });
        });
        this.updateItem(item.id, { status: "success", progress: 100 });
      } catch (error) {
        this.updateItem(item.id, {
          status: "error",
          error: error instanceof Error ? error.message : "Upload failed",
        });
      } finally {
        this.activeCount--;
        this.processQueue(); // Process next in queue
      }
    }
  }

  get stats() {
    return {
      total: this.items.length,
      pending: this.items.filter((i) => i.status === "pending").length,
      uploading: this.items.filter((i) => i.status === "uploading").length,
      success: this.items.filter((i) => i.status === "success").length,
      error: this.items.filter((i) => i.status === "error").length,
    };
  }
}
```

### Bulk Upload Hook

```typescript
// hooks/use-bulk-upload.ts
"use client";

import { useState, useEffect, useMemo } from "react";
import { UploadQueue, type QueueItem } from "@/lib/upload-queue";
import { uploadFile } from "@/lib/storage";

export function useBulkUpload(bucket: string, folder: string) {
  const [items, setItems] = useState<QueueItem[]>([]);

  const queue = useMemo(
    () =>
      new UploadQueue(async (file, onProgress) => {
        const path = `${folder}/${crypto.randomUUID()}-${file.name}`;

        // Use XMLHttpRequest for progress tracking
        return new Promise((resolve, reject) => {
          const xhr = new XMLHttpRequest();
          xhr.upload.onprogress = (e) => {
            if (e.lengthComputable) onProgress(Math.round((e.loaded / e.total) * 100));
          };
          xhr.onload = () =>
            xhr.status < 300 ? resolve(path) : reject(new Error(xhr.statusText));
          xhr.onerror = () => reject(new Error("Network error"));

          // For Supabase, use the storage API directly
          uploadFile(bucket, path, file, { upsert: true })
            .then(() => {
              onProgress(100);
              resolve(path);
            })
            .catch(reject);
        });
      }, 3),
    [bucket, folder]
  );

  useEffect(() => {
    return queue.subscribe(setItems);
  }, [queue]);

  return {
    items,
    stats: queue.stats,
    addFiles: (files: File[]) => queue.addFiles(files),
    retryFailed: () => queue.retryFailed(),
  };
}
```

### Bulk Upload UI

```typescript
"use client";

import { FileUpload } from "@/components/file-upload";
import { UploadProgressBar } from "@/components/upload-progress-bar";
import { useBulkUpload } from "@/hooks/use-bulk-upload";
import { Button } from "@/components/ui/button";
import { CheckCircle, AlertCircle, Loader2 } from "lucide-react";

export function BulkUploadPanel({ userId }: { userId: string }) {
  const { items, stats, addFiles, retryFailed } = useBulkUpload(
    "uploads",
    userId
  );

  return (
    <div className="space-y-4">
      <FileUpload onFilesAccepted={addFiles} maxFiles={20} />

      {items.length > 0 && (
        <div className="space-y-2">
          <div className="flex items-center justify-between text-sm">
            <span>
              {stats.success}/{stats.total} uploaded
            </span>
            {stats.error > 0 && (
              <Button size="sm" variant="outline" onClick={retryFailed}>
                Retry {stats.error} failed
              </Button>
            )}
          </div>

          <div className="max-h-64 space-y-1 overflow-y-auto">
            {items.map((item) => (
              <div
                key={item.id}
                className="flex items-center gap-2 rounded px-2 py-1 text-sm"
              >
                {item.status === "success" && (
                  <CheckCircle className="h-4 w-4 shrink-0 text-green-500" />
                )}
                {item.status === "error" && (
                  <AlertCircle className="h-4 w-4 shrink-0 text-destructive" />
                )}
                {item.status === "uploading" && (
                  <Loader2 className="h-4 w-4 shrink-0 animate-spin" />
                )}
                {item.status === "pending" && (
                  <div className="h-4 w-4 shrink-0 rounded-full border-2" />
                )}
                <span className="min-w-0 flex-1 truncate">
                  {item.file.name}
                </span>
                {item.status === "uploading" && (
                  <UploadProgressBar
                    progress={item.progress}
                    showLabel={false}
                    className="w-20"
                  />
                )}
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
```

---

## 10. Security

### File Type Validation (Magic Bytes)

```typescript
// lib/file-validation.ts

// Magic byte signatures for common file types
const FILE_SIGNATURES: Record<string, number[][]> = {
  "image/jpeg": [[0xff, 0xd8, 0xff]],
  "image/png": [[0x89, 0x50, 0x4e, 0x47]],
  "image/webp": [[0x52, 0x49, 0x46, 0x46]], // RIFF header
  "image/gif": [
    [0x47, 0x49, 0x46, 0x38, 0x37], // GIF87a
    [0x47, 0x49, 0x46, 0x38, 0x39], // GIF89a
  ],
  "application/pdf": [[0x25, 0x50, 0x44, 0x46]], // %PDF
  "application/zip": [[0x50, 0x4b, 0x03, 0x04]],
};

export async function validateFileType(
  file: File,
  allowedTypes: string[]
): Promise<{ valid: boolean; detectedType: string | null }> {
  const buffer = await file.slice(0, 8).arrayBuffer();
  const bytes = new Uint8Array(buffer);

  for (const [mimeType, signatures] of Object.entries(FILE_SIGNATURES)) {
    if (!allowedTypes.includes(mimeType)) continue;

    for (const sig of signatures) {
      const matches = sig.every((byte, i) => bytes[i] === byte);
      if (matches) {
        return { valid: true, detectedType: mimeType };
      }
    }
  }

  return { valid: false, detectedType: null };
}

// Comprehensive validation
export interface ValidationResult {
  valid: boolean;
  errors: string[];
}

export async function validateFile(
  file: File,
  options: {
    maxSize?: number; // bytes
    allowedTypes?: string[];
    allowedExtensions?: string[];
    checkMagicBytes?: boolean;
  }
): Promise<ValidationResult> {
  const errors: string[] = [];

  // Size check
  if (options.maxSize && file.size > options.maxSize) {
    const maxMB = (options.maxSize / 1024 / 1024).toFixed(1);
    errors.push(`File exceeds maximum size of ${maxMB}MB`);
  }

  // Extension check
  if (options.allowedExtensions) {
    const ext = file.name.split(".").pop()?.toLowerCase();
    if (!ext || !options.allowedExtensions.includes(`.${ext}`)) {
      errors.push(
        `File extension .${ext} not allowed. Allowed: ${options.allowedExtensions.join(", ")}`
      );
    }
  }

  // MIME type check (from browser — can be spoofed)
  if (options.allowedTypes && !options.allowedTypes.includes(file.type)) {
    errors.push(`File type ${file.type} not allowed`);
  }

  // Magic bytes check (reliable)
  if (options.checkMagicBytes && options.allowedTypes) {
    const { valid, detectedType } = await validateFileType(
      file,
      options.allowedTypes
    );
    if (!valid) {
      errors.push(
        `File content does not match any allowed type (detected: ${detectedType || "unknown"})`
      );
    }
  }

  return { valid: errors.length === 0, errors };
}
```

### Server-Side Validation (API Route)

```typescript
// app/api/upload/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
const ALLOWED_TYPES = [
  "image/jpeg",
  "image/png",
  "image/webp",
  "application/pdf",
];

export async function POST(req: NextRequest) {
  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const formData = await req.formData();
  const file = formData.get("file") as File | null;

  if (!file) {
    return NextResponse.json({ error: "No file provided" }, { status: 400 });
  }

  // Server-side validation
  if (file.size > MAX_FILE_SIZE) {
    return NextResponse.json({ error: "File too large" }, { status: 413 });
  }

  if (!ALLOWED_TYPES.includes(file.type)) {
    return NextResponse.json({ error: "File type not allowed" }, { status: 415 });
  }

  // Sanitize filename — strip path traversal, special characters
  const sanitizedName = file.name
    .replace(/[^a-zA-Z0-9._-]/g, "_")
    .replace(/\.{2,}/g, ".");

  const path = `${user.id}/${crypto.randomUUID()}-${sanitizedName}`;

  const { data, error } = await supabase.storage
    .from("uploads")
    .upload(path, file, {
      contentType: file.type,
      cacheControl: "3600",
    });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ path: data.path });
}
```

### Security Checklist

```typescript
/*
 * FILE UPLOAD SECURITY CHECKLIST
 *
 * Client-side (UX, not security):
 * - [ ] File extension validation
 * - [ ] File size limit
 * - [ ] MIME type check
 * - [ ] Magic bytes validation
 *
 * Server-side (actual security):
 * - [ ] Re-validate file size
 * - [ ] Re-validate MIME type
 * - [ ] Sanitize filename (no path traversal ../)
 * - [ ] Generate UUID filenames (prevent overwrites)
 * - [ ] Enforce storage bucket RLS policies
 * - [ ] Rate limit upload endpoints
 * - [ ] Set appropriate bucket file_size_limit
 * - [ ] Set allowed_mime_types on bucket
 * - [ ] Use signed upload URLs (short expiry)
 * - [ ] Scan for viruses (ClamAV or cloud service like VirusTotal API)
 * - [ ] Strip EXIF data from images (privacy)
 * - [ ] Serve uploads from separate domain/CDN (prevent XSS via uploaded HTML)
 * - [ ] Set Content-Disposition: attachment for downloads
 * - [ ] Never trust client-provided Content-Type on the server
 */
```
