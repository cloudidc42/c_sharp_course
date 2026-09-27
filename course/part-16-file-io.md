# Part 16: File I/O และการจัดการไฟล์ (Steps 481-510)

## เป้าหมายการเรียนรู้
- File และ Directory operations
- StreamReader/StreamWriter, BinaryReader/BinaryWriter
- FileStream, MemoryStream, BufferedStream
- Async file operations
- Path และ Directory utilities
- File watching (FileSystemWatcher)
- Compression (ZipFile, GZipStream)

---

## Step 481: File and Directory Basics

```csharp
using System.IO;

// ============================================================
// File Operations
// ============================================================

// Write text to file
File.WriteAllText("output.txt", "Hello, World!\nSecond line");

// Append text
File.AppendAllText("output.txt", "\nAppended line");

// Read all text
string content = File.ReadAllText("output.txt");
Console.WriteLine(content);

// Read all lines
string[] lines = File.ReadAllLines("output.txt");
foreach (string line in lines)
    Console.WriteLine(line);

// Read/Write bytes
byte[] bytes = File.ReadAllBytes("image.png");
File.WriteAllBytes("copy.png", bytes);

// File.WriteAllLines — write multiple lines at once
File.WriteAllLines("list.txt", new[] { "Line 1", "Line 2", "Line 3" });

// File.ReadLines — lazy enumeration (good for large files)
foreach (string line in File.ReadLines("large.txt"))
{
    // Processes line by line without loading entire file
    if (line.Contains("ERROR")) Console.WriteLine(line);
}

// Check existence
bool exists = File.Exists("output.txt");
Console.WriteLine($"File exists: {exists}");

// Copy, Move, Delete
File.Copy("output.txt", "output_backup.txt", overwrite: true);
File.Move("output_backup.txt", "output_moved.txt");
File.Delete("output_moved.txt");

// FileInfo — OOP approach
var fileInfo = new FileInfo("output.txt");
Console.WriteLine($"Name:       {fileInfo.Name}");
Console.WriteLine($"FullName:   {fileInfo.FullName}");
Console.WriteLine($"Directory:  {fileInfo.DirectoryName}");
Console.WriteLine($"Extension:  {fileInfo.Extension}");
Console.WriteLine($"Size:       {fileInfo.Length} bytes");
Console.WriteLine($"Created:    {fileInfo.CreationTime}");
Console.WriteLine($"Modified:   {fileInfo.LastWriteTime}");
Console.WriteLine($"IsReadOnly: {fileInfo.IsReadOnly}");
Console.WriteLine($"Exists:     {fileInfo.Exists}");

// ============================================================
// Directory Operations
// ============================================================

// Create directory
Directory.CreateDirectory("mydir/subdir"); // Creates all missing dirs

// Get files and directories
string[] files = Directory.GetFiles(".", "*.txt");
string[] dirs = Directory.GetDirectories(".");
string[] all = Directory.GetFileSystemEntries(".");

// Recursive enumeration
IEnumerable<string> allCsFiles = Directory.EnumerateFiles(".", "*.cs", SearchOption.AllDirectories);

// Directory info
var dirInfo = new DirectoryInfo(".");
Console.WriteLine($"Full path: {dirInfo.FullName}");
Console.WriteLine($"Files: {dirInfo.GetFiles().Length}");
Console.WriteLine($"Subdirs: {dirInfo.GetDirectories().Length}");

// DirectoryInfo tree
static void PrintDirectoryTree(DirectoryInfo dir, int indent = 0)
{
    Console.WriteLine($"{new string(' ', indent * 2)}{dir.Name}/");
    foreach (var subDir in dir.GetDirectories())
        PrintDirectoryTree(subDir, indent + 1);
    foreach (var file in dir.GetFiles())
        Console.WriteLine($"{new string(' ', (indent + 1) * 2)}{file.Name} ({file.Length:N0} bytes)");
}

// Path utilities
string path1 = Path.Combine("folder", "subfolder", "file.txt"); // cross-platform
string ext = Path.GetExtension("document.pdf");         // ".pdf"
string name = Path.GetFileNameWithoutExtension("doc.pdf"); // "doc"
string fullName = Path.GetFileName("C:\\folder\\file.txt"); // "file.txt"
string dir2 = Path.GetDirectoryName("C:\\folder\\file.txt"); // "C:\\folder"
string absolute = Path.GetFullPath("relative/path");     // full absolute path
string temp = Path.GetTempFileName();                    // create temp file
string tempDir = Path.GetTempPath();                    // temp directory
bool rooted = Path.IsPathRooted("C:\\folder");           // true

// Special folders
string desktop = Environment.GetFolderPath(Environment.SpecialFolder.Desktop);
string docs = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
string appData = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
```

---

## Step 482: StreamReader and StreamWriter

```csharp
using System.IO;
using System.Text;

// StreamWriter — text writing with encoding control
static void WriteWithStreamWriter(string path)
{
    using var writer = new StreamWriter(path, append: false, encoding: Encoding.UTF8);
    
    writer.WriteLine("Line 1");
    writer.WriteLine("Line 2");
    writer.Write("No newline here");
    writer.WriteLine(); // just newline
    writer.WriteLine($"Date: {DateTime.Now:yyyy-MM-dd}");
    
    // Auto-flush when disposed
}

// StreamReader — text reading with encoding detection
static void ReadWithStreamReader(string path)
{
    using var reader = new StreamReader(path, Encoding.UTF8, detectEncodingFromByteOrderMarks: true);
    
    // Read all at once
    string all = reader.ReadToEnd();
    
    // Or line by line
    reader.BaseStream.Seek(0, SeekOrigin.Begin);
    reader.DiscardBufferedData();
    
    string? line;
    int lineNum = 0;
    while ((line = reader.ReadLine()) != null)
    {
        lineNum++;
        Console.WriteLine($"{lineNum:D4}: {line}");
    }
    
    Console.WriteLine($"Detected encoding: {reader.CurrentEncoding.EncodingName}");
}

// StreamWriter with custom settings
static void WriteFormattedCsv(string path, IEnumerable<(string Name, int Age, decimal Salary)> records)
{
    using var writer = new StreamWriter(path, false, Encoding.UTF8)
    {
        AutoFlush = false,     // Buffer for performance
        NewLine = "\r\n"       // Windows line ending
    };
    
    writer.WriteLine("Name,Age,Salary"); // header
    
    foreach (var (name, age, salary) in records)
    {
        string safeName = name.Contains(',') ? $"\"{name}\"" : name;
        writer.WriteLine($"{safeName},{age},{salary:F2}");
    }
    // Flush on dispose
}

// Reading CSV line by line
static IEnumerable<(string Name, int Age, decimal Salary)> ReadCsv(string path)
{
    using var reader = new StreamReader(path, Encoding.UTF8);
    
    string? header = reader.ReadLine(); // skip header
    if (header == null) yield break;
    
    string? line;
    while ((line = reader.ReadLine()) != null)
    {
        if (string.IsNullOrWhiteSpace(line)) continue;
        
        string[] parts = line.Split(',');
        if (parts.Length < 3) continue;
        
        if (int.TryParse(parts[1].Trim(), out int age) &&
            decimal.TryParse(parts[2].Trim(), out decimal salary))
        {
            yield return (parts[0].Trim(), age, salary);
        }
    }
}

// Demo
var data = new[]
{
    ("Alice Smith", 30, 75000m),
    ("Bob, Jr.", 25, 55000m),   // comma in name — needs quoting
    ("Carol White", 35, 65000m),
};

WriteFormattedCsv("employees.csv", data);

foreach (var (name, age, salary) in ReadCsv("employees.csv"))
    Console.WriteLine($"{name}, {age}, {salary:C}");
```

---

## Step 483: BinaryReader and BinaryWriter

```csharp
using System.IO;

// Binary files — more efficient for structured data
// Stores data in binary format (not human-readable)

record Player(string Name, int Level, float Health, double Score, bool IsAlive);

static void SaveGameState(string path, List<Player> players)
{
    using var stream = new FileStream(path, FileMode.Create, FileAccess.Write);
    using var writer = new BinaryWriter(stream, Encoding.UTF8);
    
    // Write header
    writer.Write("SAVEGAME"); // magic bytes
    writer.Write(1);           // version
    writer.Write(DateTime.UtcNow.ToBinary()); // save timestamp
    
    // Write player count
    writer.Write(players.Count);
    
    // Write each player
    foreach (var p in players)
    {
        writer.Write(p.Name);     // string (length-prefixed)
        writer.Write(p.Level);    // int32
        writer.Write(p.Health);   // float
        writer.Write(p.Score);    // double
        writer.Write(p.IsAlive);  // bool
    }
}

static List<Player> LoadGameState(string path)
{
    using var stream = new FileStream(path, FileMode.Open, FileAccess.Read);
    using var reader = new BinaryReader(stream, Encoding.UTF8);
    
    // Read and verify header
    string magic = reader.ReadString();
    if (magic != "SAVEGAME") throw new InvalidDataException("Not a save file");
    
    int version = reader.ReadInt32();
    DateTime savedAt = DateTime.FromBinary(reader.ReadInt64());
    Console.WriteLine($"Save file version {version}, saved at {savedAt:G}");
    
    // Read players
    int count = reader.ReadInt32();
    var players = new List<Player>(count);
    
    for (int i = 0; i < count; i++)
    {
        string name    = reader.ReadString();
        int level      = reader.ReadInt32();
        float health   = reader.ReadSingle();
        double score   = reader.ReadDouble();
        bool isAlive   = reader.ReadBoolean();
        
        players.Add(new Player(name, level, health, score, isAlive));
    }
    
    return players;
}

// Binary file with fixed-width records
struct FixedRecord
{
    public const int NameMaxLength = 20; // chars
    public const int RecordSize = NameMaxLength * 2 + sizeof(int) + sizeof(double); // bytes
    
    public string Name;
    public int Id;
    public double Value;
}

static void WriteFixedRecords(string path, IEnumerable<FixedRecord> records)
{
    using var writer = new BinaryWriter(File.Create(path));
    
    foreach (var r in records)
    {
        // Pad/truncate name to fixed length
        string paddedName = r.Name.PadRight(FixedRecord.NameMaxLength)[..FixedRecord.NameMaxLength];
        writer.Write(paddedName.ToCharArray());
        writer.Write(r.Id);
        writer.Write(r.Value);
    }
}

// Demo
var players = new List<Player>
{
    new("Alice",  30, 100.0f, 9500.5, true),
    new("Bob",    25,  75.5f, 7200.0, true),
    new("Carol",  15,   0.0f,  500.0, false),
};

SaveGameState("save.bin", players);
var loaded = LoadGameState("save.bin");

foreach (var p in loaded)
    Console.WriteLine($"{p.Name}: Level {p.Level}, Health {p.Health:F1}, Score {p.Score:N0}");

// Reading binary with Span (high performance)
static int ReadInt32BigEndian(ReadOnlySpan<byte> bytes)
{
    return (bytes[0] << 24) | (bytes[1] << 16) | (bytes[2] << 8) | bytes[3];
}
```

---

## Step 484: Async File Operations

```csharp
using System.IO;

// Async versions of file operations
static async Task AsyncFileDemo()
{
    // Write
    await File.WriteAllTextAsync("async_output.txt", "Async content");
    await File.AppendAllTextAsync("async_output.txt", "\nAppended async");
    
    string[] lines = { "Line 1", "Line 2", "Line 3" };
    await File.WriteAllLinesAsync("lines.txt", lines);
    
    // Read
    string content = await File.ReadAllTextAsync("async_output.txt");
    string[] readLines = await File.ReadAllLinesAsync("lines.txt");
    byte[] bytes = await File.ReadAllBytesAsync("async_output.txt");
    
    Console.WriteLine(content);
    
    // Async file stream
    await using var fs = new FileStream(
        "data.bin",
        FileMode.OpenOrCreate,
        FileAccess.ReadWrite,
        FileShare.None,
        bufferSize: 4096,
        useAsync: true);
    
    // Write to stream
    byte[] writeBuffer = new byte[1024];
    Random.Shared.NextBytes(writeBuffer);
    await fs.WriteAsync(writeBuffer);
    
    // Read from stream
    fs.Seek(0, SeekOrigin.Begin);
    byte[] readBuffer = new byte[1024];
    int bytesRead = await fs.ReadAsync(readBuffer);
    Console.WriteLine($"Read {bytesRead} bytes");
    
    // Async read large file line by line
    await using var stream = File.OpenRead("large_file.txt");
    using var reader = new StreamReader(stream);
    
    // ReadLineAsync
    string? line;
    int lineCount = 0;
    while ((line = await reader.ReadLineAsync()) != null)
    {
        lineCount++;
        // Process each line
    }
    Console.WriteLine($"Total lines: {lineCount}");
    
    // Copy file async with progress
    await CopyFileWithProgressAsync(
        "source.bin", "dest.bin",
        progress: new Progress<int>(pct => Console.Write($"\rCopying: {pct}%")));
}

static async Task CopyFileWithProgressAsync(
    string source, string destination,
    IProgress<int>? progress = null,
    CancellationToken cancellationToken = default)
{
    const int bufferSize = 64 * 1024; // 64KB buffer
    
    await using var sourceStream = new FileStream(
        source, FileMode.Open, FileAccess.Read, FileShare.Read, bufferSize, useAsync: true);
    await using var destStream = new FileStream(
        destination, FileMode.Create, FileAccess.Write, FileShare.None, bufferSize, useAsync: true);
    
    long totalBytes = sourceStream.Length;
    long bytesRead = 0;
    byte[] buffer = new byte[bufferSize];
    
    int read;
    while ((read = await sourceStream.ReadAsync(buffer, cancellationToken)) > 0)
    {
        await destStream.WriteAsync(buffer.AsMemory(0, read), cancellationToken);
        bytesRead += read;
        progress?.Report((int)(bytesRead * 100 / totalBytes));
    }
    
    Console.WriteLine();
}
```

---

## Step 485: MemoryStream and BufferedStream

```csharp
using System.IO;

// MemoryStream — in-memory stream
static byte[] SerializeToBytes<T>(T data) where T : class
{
    using var ms = new MemoryStream();
    using var writer = new BinaryWriter(ms);
    
    // Write some data
    writer.Write(data.ToString() ?? "");
    
    return ms.ToArray(); // get bytes
}

// Process binary data in memory
static void MemoryStreamDemo()
{
    // Create image thumbnail in memory (without temp files)
    using var inputStream = File.OpenRead("image.jpg");
    using var outputMs = new MemoryStream();
    
    // Copy with transformation (simplified)
    inputStream.CopyTo(outputMs);
    outputMs.Position = 0; // rewind for reading
    
    byte[] thumbnail = outputMs.ToArray();
    Console.WriteLine($"Data size: {thumbnail.Length} bytes");
    
    // Working with GetBuffer vs ToArray
    var ms = new MemoryStream();
    ms.Write(new byte[] { 1, 2, 3, 4, 5 });
    
    byte[] buffer = ms.GetBuffer();  // may have extra zeros (capacity-sized)
    byte[] exact = ms.ToArray();     // exactly ms.Length bytes
    Console.WriteLine($"Buffer: {buffer.Length}, Exact: {exact.Length}"); // 256, 5
    
    // MemoryStream from existing bytes (read-only)
    byte[] data = { 1, 2, 3, 4, 5, 6, 7, 8 };
    using var readMs = new MemoryStream(data);
    int b = readMs.ReadByte(); // 1
    
    // Pipe streams together
    using var source = new MemoryStream(System.Text.Encoding.UTF8.GetBytes("Hello, World!"));
    using var dest = new MemoryStream();
    source.CopyTo(dest);
    string result = System.Text.Encoding.UTF8.GetString(dest.ToArray());
    Console.WriteLine(result); // Hello, World!
}

// BufferedStream — wrap unbuffered stream for performance
static void BufferedStreamDemo()
{
    // Network or file streams without buffering are slow for many small reads
    using var fileStream = new FileStream("data.bin", FileMode.Create);
    using var buffered = new BufferedStream(fileStream, bufferSize: 65536);
    using var writer = new BinaryWriter(buffered);
    
    // Many small writes are buffered
    for (int i = 0; i < 10000; i++)
        writer.Write(i); // 4 bytes each — buffered, not 10000 disk writes
    
    // Buffer is flushed on Dispose/Flush
}

// Pipe-based IO (.NET 5+)
using System.IO.Pipelines;

static async Task PipelineDemo()
{
    var pipe = new Pipe();
    
    async Task Producer()
    {
        PipeWriter writer = pipe.Writer;
        for (int i = 0; i < 10; i++)
        {
            Memory<byte> buffer = writer.GetMemory(10);
            int written = System.Text.Encoding.UTF8.GetBytes($"Item {i}\n", buffer.Span);
            writer.Advance(written);
            await writer.FlushAsync();
        }
        await writer.CompleteAsync();
    }
    
    async Task Consumer()
    {
        PipeReader reader = pipe.Reader;
        while (true)
        {
            ReadResult result = await reader.ReadAsync();
            ReadOnlySequence<byte> buffer = result.Buffer;
            
            foreach (var segment in buffer)
            {
                string text = System.Text.Encoding.UTF8.GetString(segment.Span);
                Console.Write(text);
            }
            
            reader.AdvanceTo(buffer.End);
            if (result.IsCompleted) break;
        }
        await reader.CompleteAsync();
    }
    
    await Task.WhenAll(Producer(), Consumer());
}
```

---

## Step 486: File Compression

```csharp
using System.IO.Compression;

// GZip compression
static async Task GZipDemo()
{
    string original = string.Join("\n", Enumerable.Range(1, 1000).Select(i => $"Line {i}: Some repeated content here"));
    byte[] originalBytes = Encoding.UTF8.GetBytes(original);
    
    // Compress
    using var compressedStream = new MemoryStream();
    await using (var gzip = new GZipStream(compressedStream, CompressionLevel.Optimal, leaveOpen: true))
    {
        await gzip.WriteAsync(originalBytes);
    }
    
    byte[] compressed = compressedStream.ToArray();
    Console.WriteLine($"Original: {originalBytes.Length:N0} bytes");
    Console.WriteLine($"Compressed: {compressed.Length:N0} bytes");
    Console.WriteLine($"Ratio: {(double)compressed.Length / originalBytes.Length:P1}");
    
    // Decompress
    using var decompressInput = new MemoryStream(compressed);
    await using var decompressGzip = new GZipStream(decompressInput, CompressionMode.Decompress);
    using var output = new MemoryStream();
    await decompressGzip.CopyToAsync(output);
    string decompressed = Encoding.UTF8.GetString(output.ToArray());
    
    Console.WriteLine($"Decompressed matches original: {decompressed == original}");
    
    // Compress file
    await using var sourceFile = File.OpenRead("large.txt");
    await using var destFile = File.Create("large.txt.gz");
    await using var gzipFile = new GZipStream(destFile, CompressionLevel.Optimal);
    await sourceFile.CopyToAsync(gzipFile);
}

// ZipFile — work with .zip archives
static void ZipFileDemo()
{
    // Create zip from directory
    ZipFile.CreateFromDirectory("mydir", "archive.zip");
    
    // Extract zip
    ZipFile.ExtractToDirectory("archive.zip", "extracted", overwriteFiles: true);
    
    // Open and manipulate zip
    using var zip = ZipFile.Open("archive.zip", ZipArchiveMode.Update);
    
    // Add entry
    ZipArchiveEntry entry = zip.CreateEntry("readme.txt");
    using (var entryStream = entry.Open())
    using (var writer = new StreamWriter(entryStream))
    {
        writer.WriteLine("This is the readme");
    }
    
    // Read entry
    var existingEntry = zip.GetEntry("readme.txt");
    if (existingEntry != null)
    {
        using var stream = existingEntry.Open();
        using var reader = new StreamReader(stream);
        Console.WriteLine(reader.ReadToEnd());
    }
    
    // List entries
    foreach (var e in zip.Entries)
        Console.WriteLine($"{e.FullName} ({e.CompressedLength:N0}/{e.Length:N0} bytes)");
    
    // Delete entry
    zip.GetEntry("readme.txt")?.Delete();
}

// Brotli compression (.NET 6+, better compression than GZip)
static async Task BrotliDemo()
{
    byte[] data = Encoding.UTF8.GetBytes("Sample data to compress...");
    
    using var compressed = new MemoryStream();
    await using (var brotli = new BrotliStream(compressed, CompressionLevel.Optimal, leaveOpen: true))
    {
        await brotli.WriteAsync(data);
    }
    
    Console.WriteLine($"Brotli: {data.Length} → {compressed.Length} bytes");
}
```

---

## Step 487: FileSystemWatcher

```csharp
using System.IO;

// FileSystemWatcher — monitor file system changes
static async Task FileWatcherDemo()
{
    string watchPath = Path.GetTempPath();
    
    using var watcher = new FileSystemWatcher(watchPath)
    {
        Filter = "*.txt",                      // only .txt files
        NotifyFilter = NotifyFilters.FileName  // what to watch
                     | NotifyFilters.LastWrite
                     | NotifyFilters.Size,
        IncludeSubdirectories = true,
        EnableRaisingEvents = true             // start watching
    };
    
    // Subscribe to events
    watcher.Created += (sender, e) =>
        Console.WriteLine($"[CREATED] {e.FullPath}");
    
    watcher.Changed += (sender, e) =>
        Console.WriteLine($"[CHANGED] {e.FullPath} ({e.ChangeType})");
    
    watcher.Deleted += (sender, e) =>
        Console.WriteLine($"[DELETED] {e.FullPath}");
    
    watcher.Renamed += (sender, e) =>
        Console.WriteLine($"[RENAMED] {e.OldFullPath} → {e.FullPath}");
    
    watcher.Error += (sender, e) =>
        Console.WriteLine($"[ERROR] {e.GetException().Message}");
    
    Console.WriteLine($"Watching {watchPath} for .txt changes. Press any key to stop...");
    
    // Create/modify files to trigger events
    string testFile = Path.Combine(watchPath, "test.txt");
    await File.WriteAllTextAsync(testFile, "Initial content");     // Created
    await Task.Delay(100);
    await File.AppendAllTextAsync(testFile, "\nAppended");         // Changed
    await Task.Delay(100);
    File.Move(testFile, testFile + ".bak");                       // Renamed
    await Task.Delay(100);
    File.Delete(testFile + ".bak");                               // Deleted
    
    await Task.Delay(500); // wait for events
    
    // Thread-safe file watcher pattern
    await SafeFileWatcherDemo();
}

static async Task SafeFileWatcherDemo()
{
    var channel = System.Threading.Channels.Channel.CreateUnbounded<FileSystemEventArgs>();
    
    using var watcher = new FileSystemWatcher(Path.GetTempPath())
    {
        Filter = "*.log",
        EnableRaisingEvents = true
    };
    
    // Queue events instead of processing inline
    watcher.Created += (s, e) => channel.Writer.TryWrite(e);
    watcher.Changed += (s, e) => channel.Writer.TryWrite(e);
    
    // Process events in background
    _ = Task.Run(async () =>
    {
        await foreach (var evt in channel.Reader.ReadAllAsync())
        {
            Console.WriteLine($"Processing: {evt.FullPath}");
            await Task.Delay(100); // simulate async processing
        }
    });
    
    // Simulate log file creation
    string logPath = Path.Combine(Path.GetTempPath(), "app.log");
    for (int i = 0; i < 5; i++)
    {
        await File.AppendAllTextAsync(logPath, $"[{DateTime.Now:T}] Log entry {i}\n");
        await Task.Delay(200);
    }
    
    await Task.Delay(500);
    channel.Writer.Complete();
    File.Delete(logPath);
}
```

---

## Step 488: โปรแกรมตัวอย่าง — File Manager System

```csharp
using System;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using System.Collections.Generic;

// ============================================================
// File Manager System — ระบบจัดการไฟล์
// ============================================================

class FileStats
{
    public string Path { get; init; } = "";
    public string Name { get; init; } = "";
    public long Size { get; init; }
    public DateTime LastModified { get; init; }
    public string Extension { get; init; } = "";
}

class FileManager
{
    public static IEnumerable<FileStats> GetFileStats(
        string directory,
        string pattern = "*",
        bool recursive = false)
    {
        var option = recursive ? SearchOption.AllDirectories : SearchOption.TopDirectoryOnly;
        
        return Directory.EnumerateFiles(directory, pattern, option)
            .Select(path =>
            {
                var info = new FileInfo(path);
                return new FileStats
                {
                    Path = path,
                    Name = info.Name,
                    Size = info.Length,
                    LastModified = info.LastWriteTime,
                    Extension = info.Extension.ToLower()
                };
            });
    }
    
    // Organize files by extension
    public static async Task OrganizeByTypeAsync(string sourceDir, string targetDir)
    {
        Console.WriteLine($"Organizing files from {sourceDir} to {targetDir}...");
        int moved = 0;
        
        foreach (string filePath in Directory.EnumerateFiles(sourceDir))
        {
            string ext = Path.GetExtension(filePath).TrimStart('.').ToLower();
            if (string.IsNullOrEmpty(ext)) ext = "other";
            
            string typeDir = Path.Combine(targetDir, ext);
            Directory.CreateDirectory(typeDir);
            
            string destPath = Path.Combine(typeDir, Path.GetFileName(filePath));
            if (File.Exists(destPath))
                destPath = Path.Combine(typeDir, $"{Path.GetFileNameWithoutExtension(filePath)}_{DateTime.Now:yyyyMMddHHmmss}{Path.GetExtension(filePath)}");
            
            File.Copy(filePath, destPath, overwrite: false);
            moved++;
        }
        
        Console.WriteLine($"Organized {moved} files");
    }
    
    // Find duplicate files by content hash
    public static async Task<Dictionary<string, List<string>>> FindDuplicatesAsync(string directory)
    {
        Console.WriteLine("Scanning for duplicates...");
        
        var hashGroups = new Dictionary<string, List<string>>();
        
        foreach (string file in Directory.EnumerateFiles(directory, "*", SearchOption.AllDirectories))
        {
            string hash = await GetFileHashAsync(file);
            if (!hashGroups.ContainsKey(hash))
                hashGroups[hash] = new List<string>();
            hashGroups[hash].Add(file);
        }
        
        return hashGroups
            .Where(g => g.Value.Count > 1)
            .ToDictionary(g => g.Key, g => g.Value);
    }
    
    private static async Task<string> GetFileHashAsync(string filePath)
    {
        using var sha256 = System.Security.Cryptography.SHA256.Create();
        await using var stream = File.OpenRead(filePath);
        byte[] hash = await sha256.ComputeHashAsync(stream);
        return Convert.ToHexString(hash);
    }
    
    // Safe file copy with progress
    public static async Task<bool> SafeCopyAsync(
        string source,
        string destination,
        IProgress<(long current, long total)>? progress = null,
        bool overwrite = false,
        CancellationToken ct = default)
    {
        if (!File.Exists(source)) return false;
        if (!overwrite && File.Exists(destination)) return false;
        
        Directory.CreateDirectory(Path.GetDirectoryName(destination)!);
        
        const int bufferSize = 64 * 1024;
        long totalBytes = new FileInfo(source).Length;
        long bytesWritten = 0;
        
        try
        {
            await using var src = new FileStream(source, FileMode.Open, FileAccess.Read, FileShare.Read, bufferSize, useAsync: true);
            await using var dst = new FileStream(destination, FileMode.Create, FileAccess.Write, FileShare.None, bufferSize, useAsync: true);
            
            byte[] buffer = new byte[bufferSize];
            int bytesRead;
            
            while ((bytesRead = await src.ReadAsync(buffer, ct)) > 0)
            {
                await dst.WriteAsync(buffer.AsMemory(0, bytesRead), ct);
                bytesWritten += bytesRead;
                progress?.Report((bytesWritten, totalBytes));
            }
            
            return true;
        }
        catch when (!ct.IsCancellationRequested)
        {
            File.Delete(destination); // cleanup partial file
            throw;
        }
    }
    
    // Directory size
    public static long GetDirectorySize(string directory)
        => Directory.EnumerateFiles(directory, "*", SearchOption.AllDirectories)
            .Sum(f => new FileInfo(f).Length);
    
    // Print disk usage tree
    public static void PrintDiskUsage(string directory, int depth = 0, int maxDepth = 3)
    {
        if (depth > maxDepth) return;
        
        try
        {
            long size = GetDirectorySize(directory);
            string indent = new string(' ', depth * 2);
            string name = Path.GetFileName(directory) ?? directory;
            Console.WriteLine($"{indent}{name}/ {FormatSize(size)}");
            
            foreach (var subDir in Directory.GetDirectories(directory).Take(10))
                PrintDiskUsage(subDir, depth + 1, maxDepth);
        }
        catch (UnauthorizedAccessException)
        {
            Console.WriteLine($"{new string(' ', depth * 2)}[Access denied]");
        }
    }
    
    public static string FormatSize(long bytes) => bytes switch
    {
        < 1024 => $"{bytes} B",
        < 1024 * 1024 => $"{bytes / 1024.0:F1} KB",
        < 1024 * 1024 * 1024 => $"{bytes / 1024.0 / 1024.0:F1} MB",
        _ => $"{bytes / 1024.0 / 1024.0 / 1024.0:F2} GB"
    };
    
    // Report
    public static void PrintReport(string directory)
    {
        Console.WriteLine($"\n=== File Report: {directory} ===");
        
        var files = GetFileStats(directory, recursive: true).ToList();
        
        Console.WriteLine($"Total files:   {files.Count:N0}");
        Console.WriteLine($"Total size:    {FormatSize(files.Sum(f => f.Size))}");
        Console.WriteLine($"Largest file:  {files.MaxBy(f => f.Size)?.Name ?? "none"}");
        Console.WriteLine($"Smallest file: {files.MinBy(f => f.Size)?.Name ?? "none"}");
        Console.WriteLine($"Newest file:   {files.MaxBy(f => f.LastModified)?.Name ?? "none"}");
        
        Console.WriteLine("\nBy Extension:");
        foreach (var g in files.GroupBy(f => f.Extension).OrderByDescending(g => g.Sum(f => f.Size)))
        {
            string ext = string.IsNullOrEmpty(g.Key) ? "(no ext)" : g.Key;
            Console.WriteLine($"  {ext,-12} {g.Count(),5} files  {FormatSize(g.Sum(f => f.Size)),10}");
        }
    }
}

// Demo
string tempDir = Path.Combine(Path.GetTempPath(), "fm_demo");
Directory.CreateDirectory(tempDir);

// Create test files
await File.WriteAllTextAsync(Path.Combine(tempDir, "doc1.txt"), "Text content 1");
await File.WriteAllTextAsync(Path.Combine(tempDir, "doc2.txt"), "Text content 2");
await File.WriteAllBytesAsync(Path.Combine(tempDir, "data.csv"), new byte[1024]);
await File.WriteAllBytesAsync(Path.Combine(tempDir, "image.png"), new byte[2048]);
await File.WriteAllBytesAsync(Path.Combine(tempDir, "backup.zip"), new byte[512]);

// Show report
FileManager.PrintReport(tempDir);

// Copy with progress
var progress = new Progress<(long current, long total)>(p =>
    Console.Write($"\rCopying: {p.current * 100 / p.total}%"));

await FileManager.SafeCopyAsync(
    Path.Combine(tempDir, "data.csv"),
    Path.Combine(tempDir, "data_backup.csv"),
    progress, overwrite: true);

Console.WriteLine("\nCopy complete!");

// Cleanup
Directory.Delete(tempDir, recursive: true);
```

---

## สรุป Part 16

### File/Directory Basics
- ✅ `File.ReadAll*/WriteAll*` convenience methods
- ✅ `File.ReadLines` — lazy enumeration for large files
- ✅ `FileInfo`, `DirectoryInfo` — OOP approach
- ✅ `Path` utilities: Combine, GetExtension, GetTempPath

### Streams
- ✅ `StreamReader`/`StreamWriter` — text with encoding
- ✅ `BinaryReader`/`BinaryWriter` — binary data
- ✅ `FileStream` — low-level, async capable
- ✅ `MemoryStream` — in-memory operations
- ✅ `BufferedStream` — performance wrapper

### Async File IO
- ✅ `File.ReadAllTextAsync`, `WriteAllTextAsync`
- ✅ `FileStream` with `useAsync: true`
- ✅ `reader.ReadLineAsync()`
- ✅ Async copy with progress reporting

### Compression
- ✅ `GZipStream`, `DeflateStream`, `BrotliStream`
- ✅ `ZipFile` — create/extract archives
- ✅ `ZipArchive` — low-level zip manipulation

### File Watching
- ✅ `FileSystemWatcher` — monitor file changes
- ✅ Thread-safe event processing with `Channel<T>`

---

## แบบฝึกหัด

1. สร้าง `LogRotator` ที่ monitor log file, rotate เมื่อ size > 10MB, เก็บ 5 backup copies
2. เขียน `FastCsvReader` ที่ใช้ `Span<char>` และ async reading สำหรับ files > 1GB
3. สร้าง `FileSyncTool` ที่ sync 2 directories แบบ one-way (copy new/modified, delete extra)
4. Implement `SafeDelete` ที่ move files ไปยัง "recycle bin" folder แทนการลบทิ้ง
5. สร้าง `DirectoryCompressor` ที่ compress directory เป็น zip พร้อม progress และ cancellation
