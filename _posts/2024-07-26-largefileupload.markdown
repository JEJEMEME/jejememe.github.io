---
layout: post
title: "Optimizing Large File Uploads in Swift: Comparing Original and Improved Approaches"
date: 2024-04-25
categories: iOS Swift Networking FileUpload MemoryManagement
author: raykim
author_url: https://github.com/raykim2414
---

# Optimizing Large File Uploads in Swift: Detailed Comparison of Approaches

In our journey to improve large file uploads in iOS applications, we've made significant changes to our implementation. Let's take a detailed look at our original approach, its limitations, and how we improved upon it.

## Original Approach: Single-Load Upload

Initially, our file upload method looked something like this:

```swift
private func s3Upload(data: Data, key: String, completion: @escaping (Bool) -> Void) {
    let s3 = S3()
    let payload = AWSPayload.data(data)
    let putObjectRequest = S3.PutObjectRequest(acl: .publicRead,
                                               body: payload,
                                               bucket: bucketName,
                                               key: key)
    let result = s3.putObject(putObjectRequest)
    result.whenSuccess { _ in
        completion(true)
    }
    
    result.whenFailure { error in
        completion(false)
    }
}
```

This method was typically called after loading the entire file into memory:

```swift
guard let fileData = try? Data(contentsOf: fileURL) else {
    print("Failed to load file data")
    return
}

s3Upload(data: fileData, key: "example-key") { success in
    if success {
        print("Upload successful")
    } else {
        print("Upload failed")
    }
}
```

### Issues with the Original Approach

1. **Memory Usage**: 
   - The entire file was loaded into memory at once with `Data(contentsOf: fileURL)`.
   - For large files (e.g., 1GB or more), this could easily lead to memory warnings or app crashes.

2. **Performance**: 
   - Loading large files into memory is time-consuming and can cause the app to become unresponsive.

3. **Scalability**: 
   - This approach doesn't scale well for very large files. As file sizes increase, the likelihood of crashes increases.

4. **User Experience**: 
   - No progress reporting, making it difficult to provide feedback to users during long uploads.

5. **Error Handling**: 
   - Limited error information. The completion handler only provides a boolean, making it difficult to diagnose issues.

6. **Network Efficiency**: 
   - If the upload fails, the entire process needs to be restarted, wasting bandwidth and time.

### Real-World Example

Let's say we tried to upload a 500MB video file using this method:

```swift
let videoURL = URL(fileURLWithPath: "/path/to/large/video.mp4")
guard let videoData = try? Data(contentsOf: videoURL) else {
    print("Failed to load video data")
    return
}

s3Upload(data: videoData, key: "large-video.mp4") { success in
    if success {
        print("Video upload successful")
    } else {
        print("Video upload failed")
    }
}
```

What would likely happen:

1. The app would freeze momentarily while loading the 500MB file into memory.
2. Memory usage would spike dramatically, potentially reaching hundreds of megabytes.
3. On devices with limited memory, this could trigger a memory warning or crash the app.
4. If the upload starts but fails midway (e.g., due to network issues), all progress is lost, and the entire 500MB needs to be uploaded again.
5. The user would have no idea of the upload progress, leading to a poor user experience for such a large file.

These issues led us to rethink our approach and develop the improved, chunked upload method.

## Improved Approach

Now, let's take a detailed look at our improved implementation:

```swift
static func chunkedFileUpload(fileURL: URL, bucket: String, key: String) async -> Result<String, Error> {
    return await withCheckedContinuation({ continuation in
        print("Attempting to upload file at path: \(fileURL.path)")
        
        let fileManager = FileManager.default
        
        // 파일 존재 여부 및 속성 확인
        guard let attributes = try? fileManager.attributesOfItem(atPath: fileURL.path),
              let fileSize = attributes[.size] as? Int64 else {
            let error = NSError(domain: "FileError", code: 0, userInfo: [NSLocalizedDescriptionKey: "Unable to get file attributes: \(fileURL.path)"])
            print(error.localizedDescription)
            continuation.resume(returning: .failure(error))
            return
        }
        
        print("File size: \(fileSize) bytes")
        if let filePermissions = attributes[.posixPermissions] as? Int {
            print("File permissions: \(String(format:"%o", filePermissions))")
        }
        
        // S3 설정
        let s3 = S3()
        
        // 멀티파트 업로드 생성
        let multipartUpload = s3.createMultipartUpload(.init(bucket: bucket, key: key))
        
        multipartUpload.whenSuccess { createMultipartUploadOutput in
            guard let uploadId = createMultipartUploadOutput.uploadId else {
                let error = NSError(domain: "UploadError", code: 0, userInfo: [NSLocalizedDescriptionKey: "Failed to get uploadId"])
                print(error.localizedDescription)
                continuation.resume(returning: .failure(error))
                return
            }
            
            // 청크 사이즈 설정 (예: 5MB)
            let chunkSize = 5 * 1024 * 1024
            var partNumber = 1
            var uploadedParts: [S3.CompletedPart] = []
            
            // 청크 단위로 파일 읽기 및 업로드
            func uploadNextChunk() {
                autoreleasepool {
                    do {
                        // 메모리 매핑을 사용하여 파일 데이터에 접근
                        let data = try Data(contentsOf: fileURL, options: .mappedIfSafe)
                        let startIndex = (partNumber - 1) * chunkSize
                        let endIndex = min(startIndex + chunkSize, data.count)
                        
                        if startIndex >= data.count {
                            // 모든 파트 업로드 완료, 멀티파트 업로드 완료 요청
                            print("All parts uploaded. Completing multipart upload.")
                            let completeMultipartUpload = s3.completeMultipartUpload(.init(
                                bucket: bucket,
                                key: key,
                                multipartUpload: .init(parts: uploadedParts),
                                uploadId: uploadId
                            ))
                            
                            completeMultipartUpload.whenSuccess { _ in
                                let path = endpoint + bucket + "/" + key
                                print("Upload completed successfully. Path: \(path)")
                                continuation.resume(returning: .success(path))
                            }
                            
                            completeMultipartUpload.whenFailure { error in
                                print("Failed to complete multipart upload: \(error.localizedDescription)")
                                continuation.resume(returning: .failure(error))
                            }
                        } else {
                            let chunkData = data.subdata(in: startIndex..<endIndex)
                            print("Uploading part \(partNumber) (size: \(chunkData.count) bytes)")
                            
                            // 파트 업로드
                            let upload = s3.uploadPart(.init(
                                body: .data(chunkData),
                                bucket: bucket,
                                key: key,
                                partNumber: Int(partNumber),
                                uploadId: uploadId
                            ))
                            
                            upload.whenSuccess { response in
                                if let eTag = response.eTag {
                                    uploadedParts.append(.init(eTag: eTag, partNumber: Int(partNumber)))
                                    partNumber += 1
                                    uploadNextChunk()
                                } else {
                                    let error = NSError(domain: "UploadError", code: 0, userInfo: [NSLocalizedDescriptionKey: "Failed to get eTag"])
                                    print(error.localizedDescription)
                                    continuation.resume(returning: .failure(error))
                                }
                            }
                            
                            upload.whenFailure { error in
                                print("Failed to upload part \(partNumber): \(error.localizedDescription)")
                                continuation.resume(returning: .failure(error))
                            }
                        }
                    } catch {
                        print("Error reading file: \(error.localizedDescription)")
                        continuation.resume(returning: .failure(error))
                    }
                }
            }
            
            uploadNextChunk()
        }
        
        multipartUpload.whenFailure { error in
            print("Failed to create multipart upload: \(error.localizedDescription)")
            continuation.resume(returning: .failure(error))
        }
    })
}
```

Let's break down the key improvements and their significance:

1. **Memory Mapping with `.mappedIfSafe`**:
   ```swift
   let data = try Data(contentsOf: fileURL, options: .mappedIfSafe)
   ```
   - This is the most crucial improvement. It uses memory mapping to efficiently handle large files.
   - Benefits:
     - Reduced memory overhead: The entire file isn't loaded into memory at once.
     - Improved performance: The OS can efficiently manage file access.
     - Better scalability: Can handle very large files more easily.

2. **Detailed Error Handling and Logging**:
   - The improved version includes more detailed error messages and logging throughout the process.
   - This aids in debugging and provides better insight into the upload process.

3. **Chunked Upload Process**:
   - The file is read and uploaded in chunks of 5MB.
   - This allows for efficient handling of large files and provides opportunities for progress tracking.

4. **Completion Handling**:
   - The function uses `Result<String, Error>` to clearly communicate the outcome of the upload.
   - It provides the final URL of the uploaded file on success.

5. **Resource Management with `autoreleasepool`**:
   - Each chunk processing is wrapped in an `autoreleasepool` block.
   - This ensures efficient memory management by releasing temporary objects after each chunk is processed.

6. **Flexible S3 Configuration**:
   - While the S3 client is still configured within the function, it's more parameterized (using `bucket` and `key` as arguments).
   - This makes the function more reusable across different S3 bucket configurations.

7. **Comprehensive File Attribute Checking**:
   - The function checks for file existence, size, and even permissions before starting the upload.
   - This proactive checking can prevent issues later in the upload process.

8. **Recursive Chunk Uploading**:
   - The `uploadNextChunk` function calls itself after successfully uploading each part.
   - This creates a clean, recursive approach to handling the multi-part upload.

9. **Detailed Progress Logging**:
   - The function logs the size of each chunk being uploaded, which can be easily extended to provide user-facing progress updates.

These improvements collectively result in a more robust, efficient, and scalable file upload function. The use of memory mapping and chunked uploading allows this function to handle files of virtually any size, while the detailed error handling and logging make it easier to monitor and debug the upload process.