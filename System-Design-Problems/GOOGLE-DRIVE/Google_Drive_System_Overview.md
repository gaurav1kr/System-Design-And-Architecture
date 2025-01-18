
# System Overview: Google Drive

Google Drive is a cloud-based file storage and synchronization service developed by Google. It allows users to store and share files online and synchronize files across multiple devices. It is tightly integrated with other Google services like Google Docs, Sheets, and Slides.

## High-Level Design (HLD)

### Key Components of Google Drive:
- **Storage System**: Google Drive stores files on distributed storage systems.
- **Metadata Management**: Metadata related to files (e.g., name, size, owner) is handled by a database.
- **File Synchronization**: Google Drive synchronizes files across devices using algorithms to detect changes and efficiently sync files.
- **Collaboration**: Google Drive enables real-time collaboration on documents.
- **APIs**: Google Drive provides APIs for file management and storage.

## Low-Level Design (LLD)

### Detailed Components
- **File Storage**: Divided into block storage and object storage for scalability.
- **Metadata Storage**: Stored in databases like Bigtable or Spanner.
- **Sync Service**: Detects changes in files and synchronizes them across devices.
- **Collaboration Layer**: Allows real-time edits and updates to shared files.

## Data Flow:
1. **File Upload**: Files are uploaded from clients and stored in the cloud after being split into chunks.
2. **Sync**: Files are synchronized between devices using checksums and diff algorithms.
3. **Collaboration**: Changes to files are synced in real time using protocols like WebSocket.

## Class Diagram

```plaintext
+---------------------+
|      User           |
+---------------------+
| - userID: string    |
| - userName: string  |
+---------------------+
| + login()           |
| + logout()          |
+---------------------+
          |
          v
+---------------------+
|      File           |
+---------------------+
| - fileID: string    |
| - fileName: string  |
| - fileSize: long    |
| - owner: User*      |
+---------------------+
| + upload()          |
| + download()        |
| + share()           |
+---------------------+
          |
          v
+---------------------+
|    StorageService   |
+---------------------+
| - storageID: string |
| - storageSize: long |
+---------------------+
| + uploadFile()      |
| + downloadFile()    |
| + syncFile()        |
+---------------------+
```

## Schema Design

**User Table**:
```plaintext
UserTable
- userID (PK)
- userName
- email
- passwordHash
```

**File Table**:
```plaintext
FileTable
- fileID (PK)
- userID (FK)
- fileName
- fileSize
- fileVersion
- timestamp
```

**Permission Table**:
```plaintext
PermissionTable
- permissionID (PK)
- fileID (FK)
- userID (FK)
- accessLevel (read/write)
```

## APIs

- Upload File: POST /upload
- Download File: GET /download/{fileID}
- Share File: POST /share/{fileID}
- List Files: GET /files
- Create Folder: POST /folders

## Services
Explanation of Services Used
- Authentication Service:
	Logs in and logs out users using the authenticate() and logout() methods.
	Simulates OAuth-like functionality for managing user sessions.
	
- File Storage Service:
	Handles file uploads and downloads with uploadFile() and downloadFile() methods.

- Sync Service:
	Synchronizes multiple files across devices using syncFiles().

- Collaboration Service:
	Provides methods to share files (shareFile()) and notify other users about file changes (notifyChanges()).

- Versioning Service:
	Tracks changes to files (trackChanges()) and supports restoring files to previous versions (restoreVersion()).


## Sequence Diagram 
sequenceDiagram
    participant User
    participant AuthenticationService
    participant FileStorageService
    participant SyncService
    participant CollaborationService
    participant VersioningService

    User ->> AuthenticationService: authenticate()
    AuthenticationService -->> User: success/failure
    User ->> FileStorageService: uploadFile(file)
    FileStorageService -->> User: upload successful
    User ->> SyncService: syncFiles([file1, file2])
    SyncService -->> User: files synchronized
    User ->> CollaborationService: shareFile(file, user)
    CollaborationService -->> User: sharing successful
    CollaborationService ->> User: notifyChanges(file)
    User ->> VersioningService: trackChanges(file)
    VersioningService -->> User: version updated
    User ->> VersioningService: restoreVersion(file, version)
    VersioningService -->> User: version restored
    User ->> AuthenticationService: logout()
    AuthenticationService -->> User: logout successful

Explanation of Sequence Diagram :-
- Authentication Service: Authenticates the user at the start of the interaction.
- File Storage Service: Handles file uploads.
- Sync Service: Synchronizes multiple files across devices.
- Collaboration Service: Shares files and sends notifications about updates.
- Versioning Service: Tracks changes and restores previous versions.
- Authentication Service: Logs the user out at the end.

## C++ Code Example:

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <mutex>
#include <map>
#include <memory>
#include <chrono>
#include <atomic>

// Mutex for thread safety
std::mutex global_mutex;

// User Class
class User {
public:
    std::string userID;
    std::string userName;
    std::string email;

    User(const std::string& id, const std::string& name, const std::string& email)
        : userID(id), userName(name), email(email) {}

    void login() {
        std::cout << "[Authentication Service] User " << userName << " logged in successfully." << std::endl;
    }

    void logout() {
        std::cout << "[Authentication Service] User " << userName << " logged out." << std::endl;
    }
};

// File Class
class File {
public:
    std::string fileID;
    std::string fileName;
    size_t fileSize;
    std::shared_ptr<User> owner;
    std::string version;

    File(const std::string& id, const std::string& name, size_t size, std::shared_ptr<User> user)
        : fileID(id), fileName(name), fileSize(size), owner(user), version("1.0") {}

    void updateVersion() {
        version = std::to_string(std::stod(version) + 0.1); // Increment version
        std::cout << "[Versioning Service] File " << fileName << " updated to version " << version << std::endl;
    }
};

// Authentication Service
class AuthenticationService {
public:
    void authenticate(std::shared_ptr<User> user) {
        user->login();
    }

    void logout(std::shared_ptr<User> user) {
        user->logout();
    }
};

// File Storage Service
class FileStorageService {
public:
    void uploadFile(const std::shared_ptr<File>& file) {
        std::lock_guard<std::mutex> lock(global_mutex);
        std::cout << "[File Storage Service] Uploading file: " << file->fileName << ", Size: " << file->fileSize << " bytes" << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(2)); // Simulate upload
        std::cout << "[File Storage Service] File " << file->fileName << " uploaded successfully!" << std::endl;
    }

    void downloadFile(const std::shared_ptr<File>& file) {
        std::cout << "[File Storage Service] Downloading file: " << file->fileName << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1)); // Simulate download
        std::cout << "[File Storage Service] File " << file->fileName << " downloaded successfully!" << std::endl;
    }
};

// Sync Service
class SyncService {
public:
    void syncFiles(const std::vector<std::shared_ptr<File>>& files) {
        for (const auto& file : files) {
            std::cout << "[Sync Service] Synchronizing file: " << file->fileName << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(500)); // Simulate sync
            std::cout << "[Sync Service] File " << file->fileName << " synchronized successfully!" << std::endl;
        }
    }
};

// Collaboration Service
class CollaborationService {
public:
    void shareFile(const std::shared_ptr<File>& file, const std::shared_ptr<User>& user) {
        std::cout << "[Collaboration Service] Sharing file: " << file->fileName << " with user: " << user->userName << std::endl;
    }

    void notifyChanges(const std::shared_ptr<File>& file) {
        std::cout << "[Collaboration Service] Notifying users about changes to file: " << file->fileName << std::endl;
    }
};

// Versioning Service
class VersioningService {
public:
    void trackChanges(const std::shared_ptr<File>& file) {
        file->updateVersion();
        std::cout << "[Versioning Service] Tracking changes to file: " << file->fileName << ", Current Version: " << file->version << std::endl;
    }

    void restoreVersion(const std::shared_ptr<File>& file, const std::string& version) {
        std::cout << "[Versioning Service] Restoring file: " << file->fileName << " to version " << version << std::endl;
    }
};

// Main function
int main() {
    // Create Users
    auto user1 = std::make_shared<User>("1", "Alice", "alice@example.com");
    auto user2 = std::make_shared<User>("2", "Bob", "bob@example.com");

    // Authentication Service
    AuthenticationService authService;
    authService.authenticate(user1);

    // Create Files
    auto file1 = std::make_shared<File>("F1", "document1.txt", 1024, user1);
    auto file2 = std::make_shared<File>("F2", "image2.jpg", 2048, user2);

    // File Storage Service
    FileStorageService storageService;
    storageService.uploadFile(file1);
    storageService.uploadFile(file2);

    // Sync Service
    SyncService syncService;
    syncService.syncFiles({file1, file2});

    // Collaboration Service
    CollaborationService collabService;
    collabService.shareFile(file1, user2);
    collabService.notifyChanges(file1);

    // Versioning Service
    VersioningService versionService;
    versionService.trackChanges(file1);
    versionService.restoreVersion(file1, "1.0");

    // Logout user after task completion
    authService.logout(user1);

    return 0;
}


```

