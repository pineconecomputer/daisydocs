```
 ============================================================================
  S d F a t   ( A d a f r u i t   F o r k )
  P R O G R A M M E R ' S   R E F E R E N C E   M A N U A L

  Filesystem Services for the Adafruit Metro ESP32-S3
  Volume I:  Devices, Volumes, and Files

  Target:    Metro ESP32-S3, rev B
  Bus:       SPI (onboard MicroSD slot)
  Framework: Arduino / PlatformIO

  Edition 3
 ============================================================================
```

## HOW TO READ THIS MANUAL

**Chapter 0** is a complete working sketch. Paste it, build it, run it.

**Chapters 1 through 6** are the working manual: mounting a card, reading its
properties, opening files, reading and writing them, and walking directories.

**Chapter 7** is a flat member reference for all four objects.

**Chapter 8** covers high-rate logging, buffered output, and timestamps.

**Appendix A** is the gotcha list. Every trap, edge case, and platform quirk
lives there, numbered `[G1]`, `[G2]`, and so on. Those markers appear throughout
the manual and mean "there is a known trap here." Look them up when you hit
trouble; they are not required reading up front.

**Code style.** SdFat is a C++ library, so its objects are unavoidable, but the
examples here use C idiom wherever it exists: `char` buffers rather than
`String`, `snprintf` rather than stream operators, plain `Serial.print` rather
than the library's `ArduinoOutStream`. Most official examples include
`sdios.h` and write `cout << F("...")`. That helper is optional and is not used
here.

---

# PART ONE — GETTING RUNNING

## CHAPTER 0 — TEN MINUTES

Paste this, build it, run it. It mounts the card, reports what it found, writes
a file, reads it back, appends to it, and lists the card. If this runs, you
have a working foundation and can skip to whichever chapter covers what you
need next.

```cpp
#ifndef DISABLE_FS_H_WARNING
#define DISABLE_FS_H_WARNING            /* see [G1] */
#endif
#include "SdFat.h"

#ifndef SDCARD_SS_PIN
const uint8_t SD_CS_PIN = SS;           /* see [G2] */
#else
const uint8_t SD_CS_PIN = SDCARD_SS_PIN;
#endif

#define SD_CONFIG SdSpiConfig(SD_CS_PIN, SHARED_SPI, SD_SCK_MHZ(25))

SdFs   sd;                              /* the card + filesystem  */
FsFile file;                            /* one open file          */

void setup(void) {
  char  buf[64];
  int   n;

  Serial.begin(115200);
  uint32_t t0 = millis();
  while (!Serial && millis() - t0 < 3000) { /* wait */ }   /* see [G3] */

  /* ---- 1. Mount ---------------------------------------------------- */
  if (!sd.begin(SD_CONFIG)) {
    Serial.println("Mount failed.");
    return;
  }
  Serial.println("Mounted.");

  /* ---- 2. Report what was found ------------------------------------ */
  Serial.print("FAT type      : ");
  Serial.println(sd.fatType());                    /* 16, 32, or 64 */
  Serial.print("Card size (MB): ");
  Serial.println((uint32_t)(sd.card()->sectorCount() / 1953));
  Serial.print("Cluster bytes : ");
  Serial.println(sd.vol()->bytesPerCluster());

  /* ---- 3. Write a file (fresh every run) --------------------------- */
  if (!file.open("hello.txt", O_WRONLY | O_CREAT | O_TRUNC)) {
    Serial.println("open for write failed");
    return;
  }
  file.println("line one");
  file.println("line two");
  file.close();

  /* ---- 4. Append to it --------------------------------------------- */
  if (!file.open("hello.txt", O_WRONLY | O_APPEND)) {
    Serial.println("open for append failed");
    return;
  }
  file.println("line three (appended)");
  file.close();

  /* ---- 5. Read it back --------------------------------------------- */
  if (!file.open("hello.txt", O_RDONLY)) {
    Serial.println("open for read failed");
    return;
  }
  while ((n = file.fgets(buf, sizeof(buf))) > 0) {
    Serial.print(buf);                      /* fgets keeps the newline */
  }

  /* ---- 6. Ask the file about itself -------------------------------- */
  char name[13];                            /* 13 minimum - see [G4]  */
  file.getName(name, sizeof(name));
  Serial.print("Name : ");
  Serial.println(name);
  Serial.print("Size : ");
  Serial.println((uint32_t)file.fileSize());
  file.close();

  /* ---- 7. List the card -------------------------------------------- */
  Serial.println("--- card contents ---");
  sd.ls(LS_A | LS_R | LS_SIZE | LS_DATE);   /* see [G5] */
}

void loop(void) {}
```

If step 1 fails, go straight to Appendix C. If it succeeds and something later
fails, the relevant chapter below covers it.

---

## CHAPTER 1 — THE FOUR OBJECTS

You deal with four things. Almost every "why doesn't this method exist"
question is a method being looked up on the wrong one.

### Figure 1-1: What lives where

```
   sd               the whole card + filesystem
   |                begin, open, ls, mkdir, remove, rename, exists
   |
   +-- sd.vol()     the volume (the formatted partition)
   |                fatType, bytesPerCluster, clusterCount, freeClusterCount
   |
   +-- sd.card()    the raw card hardware
   |                sectorCount, errorCode, errorData, readCID
   |
   file             one open file or directory
                    open, close, read, write, print, seek, getName, fileSize
```

Plain English:

```
  I want to know...                        Ask...
  ---------------------------------------  --------------------
  ...how big the card is                   sd.card()
  ...whether it's FAT32 or exFAT           sd.vol()  (or sd.fatType())
  ...how much space is free                sd.vol()
  ...whether /logs/today.csv exists        sd
  ...to create, delete, or rename a path   sd
  ...anything about a file I have open     file
  ---------------------------------------  --------------------
```

`sd.fatType()` is offered on `sd` directly as a convenience; everything else
volume-related goes through `sd.vol()`.

### 1.1 Which classes to declare

Use **`SdFs`** and **`FsFile`**. Always, on this board.

```cpp
SdFs   sd;
FsFile file;
```

There are other pairs — `SdFat32`/`File32` for FAT-only, `SdExFat`/`ExFile` for
exFAT-only — but they only save flash, which you have 16 MB of. There is also a
generic `SdFat`/`File` pair which **does not work on ESP32** `[G1]`.

> **Do not write `SdFat` or `File` in a real project.** They are typedefs that
> resolve differently depending on `SDFAT_FILE_TYPE`, which a stray build flag
> can change without warning. Mixing a resolved `SdFat` with an `FsFile`
> produces a silent, error-free failure that is genuinely hard to diagnose
> `[G28]`. `SdFs` and `FsFile` are unambiguous. Use them.

The three pairs must never be mixed:

```
  SdFs     sd;   FsFile dir, file;      <- use this
  SdFat32  sd;   File32 dir, file;
  SdExFat  sd;   ExFile dir, file;
```

---

## CHAPTER 2 — MOUNTING THE CARD

### 2.1 The one line that matters

```cpp
if (!sd.begin(SD_CONFIG)) {
  /* handle failure */
}
```

`begin()` returns `bool`. That is your entire error check for mounting.

### 2.2 Building SD_CONFIG

```cpp
#define SD_CONFIG SdSpiConfig(SD_CS_PIN, SHARED_SPI, SD_SCK_MHZ(25))
                              |           |           |
                              |           |           +-- bus speed
                              |           +-------------- bus sharing mode
                              +-------------------------- chip select pin
```

Verified values:

```
  SHARED_SPI     == 0    other devices may use SCK/MOSI/MISO
  DEDICATED_SPI  == 1    the card owns the bus; much faster
  SD_SCK_MHZ(n)  == n * 1000000          [G6]
```

Pick `SHARED_SPI` if anything else is on the SPI bus — a display, a sensor, an
Ethernet controller. Pick `DEDICATED_SPI` if the card is alone on it. The
speed difference is large: the library's README measures roughly 294 KB/sec
shared against 3965 KB/sec dedicated on a Due. `[G7]`

Start at `SD_SCK_MHZ(25)`. If you get intermittent read or write failures, drop
to `SD_SCK_MHZ(4)` and work back up. That is the standard first move.

### 2.3 The chip select pin

On the Metro ESP32-S3 the slot is wired SCK 39 / MISO 21 / MOSI 42 / CS 45. The
idiom below lets the board definition supply the pin `[G2]`:

```cpp
#ifndef SDCARD_SS_PIN
const uint8_t SD_CS_PIN = SS;
#else
const uint8_t SD_CS_PIN = SDCARD_SS_PIN;
#endif
```

Print it once to confirm:

```cpp
Serial.print("CS pin: ");
Serial.println(SD_CS_PIN);
```

To take explicit control of the bus rather than relying on the default `SPI`
instance, create your own and pass it as the fourth argument to `SdSpiConfig`:

```cpp
SPIClass spi(HSPI);

spi.begin(39, 21, 42, SD_CS_PIN);     /* SCK, MISO, MOSI, CS */
sd.begin(SdSpiConfig(SD_CS_PIN, SHARED_SPI, SD_SCK_MHZ(25), &spi));
```

Declaring an `SPIClass` and then calling `sd.begin(SD_CS_PIN)` does **not**
connect the two. The one-argument form expands to
`begin(SdSpiConfig(csPin, SHARED_SPI))`, which uses the default `SPI` object
and ignores yours entirely.

### 2.4 Shorter forms of begin()

```cpp
bool begin(SdCsPin_t csPin = SS);              /* pin only          */
bool begin(SdCsPin_t csPin, uint32_t maxSck);  /* pin + speed       */
bool begin(SdSpiConfig spiConfig);             /* full control      */
```

`sd.begin(45)` works and is fine for a quick test. Use the config form in real
code so the sharing mode is explicit.

### 2.5 Finding out why it failed

```cpp
if (!sd.begin(SD_CONFIG)) {
  Serial.print("error code 0x");
  Serial.println(sd.sdErrorCode(), HEX);
  Serial.print("error data 0x");
  Serial.println(sd.sdErrorData(), HEX);
  return;
}
```

`sd.sdErrorCode()` and `sd.sdErrorData()` are convenience accessors on `sd`
itself; `sd.card()->errorCode()` and `->errorData()` are the same values from
the card object.

To turn a code into words:

```cpp
printSdErrorSymbol(&Serial, code);   /* e.g. SD_CARD_ERROR_CMD0 */
printSdErrorText(&Serial, code);     /* human-readable text     */
```

There are also halt-on-error helpers used throughout the official examples:

```cpp
sd.initErrorHalt(&Serial);        /* diagnose mount failure, then hang */
sd.errorHalt(&Serial, "message"); /* your message + card state, hang   */
```

These are bench tools, not error handlers `[G8]`.

### 2.6 Unmounting

```cpp
sd.end();
```

Releases the card. Needed if you want to re-`begin()` with different settings,
or power the card down.

---

## CHAPTER 3 — WHAT IS ON THE CARD

### 3.1 The numbers you actually want

```cpp
uint8_t  type      = sd.fatType();               /* 16, 32, or 64      */
uint32_t sectors   = sd.card()->sectorCount();   /* 512 bytes each     */
uint32_t clustByte = sd.vol()->bytesPerCluster();
uint32_t clusters  = sd.vol()->clusterCount();
int32_t  freeClust = sd.vol()->freeClusterCount();   /* slow! [G9]     */
```

Converting to something human:

```cpp
/* Card size. 1 MB = 1,000,000 bytes, so 1 MB = 1953.125 sectors. */
uint32_t cardMB  = sectors / 1953;

/* Free space, in bytes. Watch the overflow on large cards. */
uint64_t freeBytes = (uint64_t)freeClust * clustByte;
```

Printing a 64-bit value on Arduino needs a cast or `snprintf`:

```cpp
char line[48];
snprintf(line, sizeof(line), "Free: %lu MB",
         (unsigned long)(freeBytes / 1000000ULL));
Serial.println(line);
```

Testing the filesystem type:

```cpp
if (sd.fatType() <= 32) {
  Serial.print("FAT");
  Serial.println(sd.fatType());     /* FAT16 or FAT32 */
} else {
  Serial.println("exFAT");
}
```

`[G10]` explains why you test the range rather than compare to a constant.

### 3.2 The volume name — honest answer

**SdFat does not expose the volume label.** There is no `getVolumeLabel()`, no
`volumeName()`. The label exists in the boot sector structure (`volumeLabel[11]`
in `common/FsStructs.h`) but no public method returns it. If you need the label
you must read sector zero yourself and parse it, or walk the root directory
looking for the entry with `FAT_ATTRIB_LABEL` (0x08) set.

What you *can* get easily is the **card's** identity, which is usually what
people actually want when they ask:

```cpp
cid_t cid;

if (sd.card()->readCID(&cid)) {
  char product[6];
  memcpy(product, cid.pnm, 5);       /* pnm is 5 chars, NOT terminated */
  product[5] = '\0';

  Serial.print("Product      : ");
  Serial.println(product);
  Serial.print("Manufacturer : 0x");
  Serial.println(cid.mid, HEX);
  Serial.print("OEM          : ");
  Serial.print(cid.oid[0]);
  Serial.println(cid.oid[1]);
  Serial.print("Serial       : 0x");
  Serial.println(cid.psn(), HEX);
  Serial.print("Made         : ");
  Serial.print(cid.mdtMonth());
  Serial.print('/');
  Serial.println(cid.mdtYear());
}
```

`cid.pnm` is a five-character product name that is **not null-terminated**;
copying it into a six-byte buffer and terminating it yourself is required
`[G11]`. `psn()`, `mdtMonth()`, and `mdtYear()` are accessor functions, not
fields.

---

# PART TWO — FILES

## CHAPTER 4 — WRITING, READING, APPENDING

### 4.1 The three things you will do 95% of the time

```cpp
/* WRITE - fresh file every time, discarding old contents */
file.open("data.txt", O_WRONLY | O_CREAT | O_TRUNC);

/* APPEND - add to the end, create if missing */
file.open("log.csv", O_WRONLY | O_CREAT | O_APPEND);

/* READ */
file.open("data.txt", O_RDONLY);
```

Always test the return:

```cpp
if (!file.open("data.txt", O_WRONLY | O_CREAT | O_TRUNC)) {
  Serial.println("open failed");
  return;
}
```

### 4.2 The flags, plainly

```
  You must pick exactly one access mode:
    O_RDONLY    read only
    O_WRONLY    write only
    O_RDWR      read and write

  Then add modifiers as needed:
    O_CREAT     make the file if it doesn't exist
    O_TRUNC     empty the file on open
    O_APPEND    every write goes to the end, no matter where you seek
    O_AT_END    start at the end, but you may seek away afterward
    O_EXCL      fail if the file already exists
    O_SYNC      flush on every write (slow, safe)
```

Never use the numeric values of these flags — they differ by platform. `[G12]`

### Figure 4-1: Flag recipes

```
  What you want                         What to pass
  ------------------------------------  ---------------------------------
  Read an existing file                 O_RDONLY
  Overwrite from scratch                O_WRONLY | O_CREAT | O_TRUNC
  Append to a log                       O_WRONLY | O_CREAT | O_APPEND
  Create only if new, else fail         O_WRONLY | O_CREAT | O_EXCL
  Read and write, positioned at end     O_RDWR   | O_CREAT | O_AT_END
  ------------------------------------  ---------------------------------
```

There are two shorthand macros:

```
  FILE_READ   ==  O_RDONLY
  FILE_WRITE  ==  O_RDWR | O_CREAT | O_AT_END
```

`FILE_WRITE` does **not** give you an empty file `[G13]`. Adafruit's own demo
sketch uses it, which is why re-running that demo grows its test file.

### 4.3 Writing

`FsFile` inherits from `Print`, so everything you know from `Serial` works:

```cpp
file.print("temperature=");
file.println(23.4);
file.write(buf, len);              /* raw bytes  */
file.write('\n');                  /* one byte   */
```

C-style formatting, which is usually cleaner than chained `print` calls:

```cpp
char line[64];
int  n = snprintf(line, sizeof(line), "%lu,%d,%.2f\n",
                  (unsigned long)millis(), reading, voltage);
file.write(line, n);
```

Print methods do not return errors. Check once at the end:

```cpp
if (file.getWriteError()) {
  Serial.println("write failed");
}
```

### 4.4 Reading

Byte at a time:

```cpp
int c;
while ((c = file.read()) >= 0) {     /* read() returns -1 at EOF */
  Serial.write(c);
}
```

Block at a time — much faster:

```cpp
uint8_t buf[256];
int     n;

while ((n = file.read(buf, sizeof(buf))) > 0) {
  Serial.write(buf, n);
}
```

A line at a time, which is what you want for text:

```cpp
char line[64];
int  n;

while ((n = file.fgets(line, sizeof(line))) > 0) {
  /* line still contains its trailing '\n' if there was room */
  if (n > 0 && line[n - 1] == '\n') {
    line[n - 1] = '\0';              /* strip it */
  }
  Serial.println(line);
}
```

`fgets` is the C function, not Arduino's `readStringUntil`. It returns the
number of characters placed in the buffer, 0 at end of file, or negative on
error. Long lines need a guard `[G14]`.

Signature: `int fgets(char *str, int num, char *delim = nullptr)`. The third
argument sets a custom delimiter set; omit it for newline behaviour.

### 4.5 Closing — do not skip this

```cpp
file.sync();     /* flush to card, keep the file open  */
file.close();    /* sync, then release the file object */
```

Data you wrote but never synced is gone on reset or power loss, and the
directory entry is not updated either, so the file may appear empty or
zero-length `[G15]`.

### 4.6 Moving around inside a file

```cpp
void     rewind(void);                  /* back to byte 0 */
bool     seekSet(uint64_t pos);         /* absolute       */
bool     seekCur(int64_t offset);       /* relative       */
bool     seekEnd(int64_t offset = 0);   /* from the end   */
uint64_t curPosition(void) const;
uint64_t fileSize(void) const;
```

Backward `seekCur` near the start of a file has an unsigned-arithmetic trap
`[G16]`.

---

## CHAPTER 5 — ASKING A FILE ABOUT ITSELF

### 5.1 Getting the name — the direct answer

```cpp
size_t getName(char *name, size_t len);
```

You pass it **a `char` array you own and its size**. It fills the array with
the filename plus a terminating zero, and returns the string length.

```cpp
char name[64];

if (file.getName(name, sizeof(name)) > 0) {
  Serial.print("Name: ");
  Serial.println(name);
}
```

Three rules:

1. **The buffer must be at least 13 bytes.** That is the documented minimum
   (8.3 name plus dot plus terminator). Anything smaller is not supported.
2. **Long names get truncated, not rejected.** If the real name is longer than
   your buffer, you silently get a shortened version. For long filenames use
   64 or 256 bytes.
3. **It returns the length, or 0 on failure.** `[G4]`

Two variants exist on the underlying FAT file class if you need explicit
encoding control:

```
  getName7(char *name, size_t size)    ASCII only
  getName8(char *name, size_t size)    UTF-8
```

If you only want to print the name and never manipulate it, there is a shortcut
that needs no buffer at all:

```cpp
file.printName(&Serial);
```

### 5.2 Size, position, and dates

```cpp
uint64_t size = file.fileSize();
uint64_t pos  = file.curPosition();
int      left = file.available();       /* int - see [G17] */
```

Printing a `uint64_t`:

```cpp
Serial.println((uint32_t)file.fileSize());          /* if under 4 GB */
```

Dates, print-only (there is no plain getter that returns a struct):

```cpp
file.printModifyDateTime(&Serial);
file.printAccessDate(&Serial);
file.printCreateDateTime(&Serial);
file.printFileSize(&Serial);
```

All four return `size_t` — the number of characters printed.

### 5.3 What kind of thing is it

```cpp
bool isDir(void)          const;   /* a directory              */
bool isFile(void)         const;   /* a normal file            */
bool isFileOrSubDir(void) const;
bool isHidden(void)       const;
bool isReadOnly(void)     const;
bool isSystem(void)       const;
bool isOpen(void)         const;
bool isBusy(void);                 /* card busy with this file */
```

### 5.4 Renaming an open file

```cpp
file.rename("newname.txt");             /* stays open, keeps writing */
file.rename("subdir/newname.txt");      /* moves too                 */
```

Renaming via the volume — `sd.rename(old, new)` — requires the file to be
**closed** first `[G18]`.

---

## CHAPTER 6 — DIRECTORIES

### 6.1 Path operations, all on `sd`

```cpp
bool exists(const char *path);
bool mkdir (const char *path, bool pFlag = true);
bool rmdir (const char *path);                  /* must be empty */
bool remove(const char *path);                  /* files only    */
bool rename(const char *oldPath, const char *newPath);
```

`mkdir` creates missing parent directories by default, so `sd.mkdir("a/b/c")`
builds all three levels. `open()` does the opposite — it will never create a
directory `[G19]`.

```cpp
if (!sd.exists("/logs")) {
  sd.mkdir("/logs");
}
if (!file.open("/logs/today.csv", O_WRONLY | O_CREAT | O_APPEND)) {
  Serial.println("open failed");
}
```

### 6.2 Listing to the serial port

```cpp
sd.ls();                                   /* working dir, names only  */
sd.ls(LS_SIZE);                            /* with sizes               */
sd.ls(LS_A | LS_R | LS_SIZE | LS_DATE);    /* everything, recursive    */
sd.ls("/logs", LS_SIZE);                   /* a specific path          */
sd.ls(&Serial, LS_SIZE);                   /* explicit output target   */
```

Flags:

```
  LS_A      1   include hidden files      [G5]
  LS_DATE   2   show modify date and time
  LS_SIZE   4   show file size
  LS_R      8   recurse into subdirectories
```

### 6.3 Walking a directory in code

`ls()` prints. To *process* entries, open the directory as a file and step
through it:

```cpp
FsFile dir;
FsFile entry;
char   name[64];

if (!dir.open("/")) {
  Serial.println("cannot open root");
  return;
}

while (entry.openNext(&dir, O_RDONLY)) {
  entry.getName(name, sizeof(name));

  Serial.print(name);
  if (entry.isDir()) {
    Serial.println("/");
  } else {
    Serial.print("  ");
    Serial.println((uint32_t)entry.fileSize());
  }
  entry.close();                    /* close each entry */
}
dir.close();
```

`openNext` returns `false` when there are no more entries. Walking the same
directory twice needs a `dir.rewind()` in between `[G20]`.

### 6.4 The working directory

```cpp
sd.chdir("/logs");     /* relative paths now resolve under /logs */
sd.chdir();            /* back to root                           */
```

This is global state, which causes trouble in programs with more than one
subsystem touching the card `[G21]`. Absolute paths avoid the issue entirely.

---

# PART THREE — WHEN YOU NEED MORE

## CHAPTER 7 — MEMBER REFERENCE

Quick lookup. Everything below was read from the headers.

### 7.1 `SdFs sd` — card and filesystem

```
  MOUNTING
  bool     begin(SdSpiConfig config)
  bool     begin(SdCsPin_t csPin = SS)
  bool     begin(SdCsPin_t csPin, uint32_t maxSck)
  void     end(void)

  PATHS
  FsFile   open(const char *path, oflag_t oflag = O_RDONLY)
  bool     exists(const char *path)
  bool     mkdir(const char *path, bool pFlag = true)
  bool     rmdir(const char *path)
  bool     remove(const char *path)
  bool     rename(const char *oldPath, const char *newPath)
  bool     chdir(void)
  bool     chdir(const char *path)

  LISTING
  bool     ls(void)
  bool     ls(uint8_t flags)
  bool     ls(const char *path, uint8_t flags = 0)
  bool     ls(print_t *pr, uint8_t flags)
  bool     ls(print_t *pr, const char *path, uint8_t flags)

  INFO AND ERRORS
  uint8_t  fatType(void)
  FsVolume *vol(void)
  SdCard   *card(void)
  uint8_t  sdErrorCode(void)
  uint8_t  sdErrorData(void)
  void     initErrorHalt(print_t *pr)
  void     errorHalt(print_t *pr, const char *msg)
```

Every path method also has a `String` overload. Prefer the `char*` forms.

### 7.2 `sd.vol()` — the volume

```
  uint8_t   fatType(void)            16, 32, or 64
  uint32_t  bytesPerCluster(void)
  Sector_t  sectorsPerCluster(void)
  Cluster_t clusterCount(void)
  int32_t   freeClusterCount(void)   slow on first call [G9]
  uint8_t   fatCount(void)
  Sector_t  fatStartSector(void)
  Sector_t  dataStartSector(void)
  bool      isBusy(void)
  int       attrib(const char *path)
  bool      attrib(const char *path, uint8_t bits)
  void      chvol(void)              make this the current volume
```

No volume label accessor exists. See 3.2.

### 7.3 `sd.card()` — the raw card

```
  Sector_t  sectorCount(void)
  uint8_t   errorCode(void)
  uint32_t  errorData(void)
  bool      readCID(cid_t *cid)
  bool      readCSD(csd_t *csd)
  bool      isBusy(void)
```

### 7.4 `FsFile file` — an open file or directory

```
  OPEN AND CLOSE
  bool      open(const char *path, oflag_t oflag = O_RDONLY)
  bool      openNext(FsBaseFile *dir, oflag_t oflag = O_RDONLY)
  bool      close(void)
  bool      sync(void)
  bool      isOpen(void) const

  WRITE  (inherited from Print)
  size_t    print(...)     println(...)
  size_t    write(uint8_t b)
  size_t    write(const void *buf, size_t count)
  bool      getWriteError(void) const

  READ
  int       read(void)                        -1 at EOF
  int       read(void *buf, size_t count)
  int       peek(void)
  int       available(void)                   [G17]
  int       fgets(char *str, int num, char *delim = nullptr)

  POSITION
  void      rewind(void)
  bool      seekSet(uint64_t pos)
  bool      seekCur(int64_t offset)
  bool      seekEnd(int64_t offset = 0)
  uint64_t  curPosition(void) const
  uint64_t  fileSize(void) const

  IDENTITY
  size_t    getName(char *name, size_t len)   buffer >= 13 bytes [G4]
  size_t    printName(print_t *pr)
  size_t    printFileSize(print_t *pr)
  size_t    printModifyDateTime(print_t *pr)
  size_t    printAccessDate(print_t *pr)
  size_t    printCreateDateTime(print_t *pr)

  TYPE TESTS
  bool      isDir(void) const        isFile(void) const
  bool      isHidden(void) const     isReadOnly(void) const
  bool      isSystem(void) const     isFileOrSubDir(void) const
  bool      isBusy(void)

  MODIFY
  bool      rename(const char *newPath)
  bool      rename(FsBaseFile *dir, const char *newPath)
  bool      truncate(void)                    cut at current position
  bool      truncate(uint64_t length)
  bool      preAllocate(uint64_t length)      [G22]
  uint8_t   getError(void) const              [G23]
```

---

## CHAPTER 8 — ADVANCED TOPICS

### 8.1 High-rate logging

If you are sampling faster than a few hundred times a second, writing straight
from the sample loop will drop data. SD cards stall unpredictably for tens of
milliseconds while erasing flash.

### Figure 8-1: The ring buffer approach

```
  +--------+     +-------------------+     +---------+
  | sensor | --> |     RingBuf       | --> | SD card |
  +--------+     |  (RAM, N sectors) |     +---------+
    fast,        +-------------------+       slow,
    regular            ^      |              irregular
                       |      |
       sample loop always ----+
       writes here, never              writeOut(512) only when
       blocks on the card              file.isBusy() is false
```

```cpp
#include "RingBuf.h"
#include "SdFat.h"

#define LOG_FILE_SIZE     (10 * SAMPLES_PER_SECOND * 600)  /* ~10 min */
#define RING_BUF_SECTORS  ((10 * SAMPLES_PER_SECOND) / 5 + 511) / 512

SdFs   sd;
FsFile file;
RingBuf<FsFile, 512 * RING_BUF_SECTORS> rb;

/* 1. Open, truncating. preAllocate needs an EMPTY file. [G22] */
if (!file.open("log.csv", O_RDWR | O_CREAT | O_TRUNC)) { return; }

/* 2. Reserve contiguous space before writing a single byte. */
if (!file.preAllocate(LOG_FILE_SIZE)) { return; }

/* 3. Attach the buffer. */
rb.begin(&file);

/* 4. Sample loop. */
while (logging) {
  size_t n = rb.bytesUsed();

  if ((n + file.curPosition()) > (LOG_FILE_SIZE - 20)) {
    break;                                  /* file full */
  }
  if (n >= 512 && !file.isBusy()) {
    if (512 != rb.writeOut(512)) { break; } /* flush one sector */
  }

  /* ... wait until the next sample is due ... */

  rb.print(value);
  rb.write(',');
  rb.println(other);

  if (rb.getWriteError()) { break; }        /* buffer overran */
}

/* 5. Drain, trim off the unused reservation, close. */
rb.sync();
file.truncate();
file.close();
```

`RingBufLogger` refuses to compile under `SHARED_SPI` — sustained high-rate
logging needs a dedicated bus. `[G24]`

Instrument while tuning: track the peak of `rb.bytesUsed()` (if it approaches
your buffer size, enlarge the buffer) and the minimum spare microseconds before
each sample (if it goes negative, your rate is too high).

### 8.2 Faster small writes

`BufferedPrint` batches many small `print` calls into block writes.

```cpp
#include "BufferedPrint.h"

BufferedPrint<FsFile, 64> bp;

bp.begin(&file);
for (uint16_t i = 0; i < N; i++) {
  bp.printField(i, '\n');      /* value plus terminator, one call */
}
bp.sync();                     /* MUST flush before close  [G25]  */
file.close();
```

`printField` handles `char`, `const char *`, integers, `float`, and `double`.
The float and double forms take an optional precision that defaults to 2
`[G26]`.

This matters far more on an AVR than on a 240 MHz S3. On this board,
`snprintf` into a buffer followed by a single `file.write()` is simpler and
nearly as fast.

### 8.3 File timestamps

FAT stores creation and modify times, but SdFat has no clock. You supply one:

```cpp
#include "RTClib.h"
RTC_PCF8523 rtc;

void dateTime(uint16_t *date, uint16_t *time, uint8_t *ms10) {
  DateTime now = rtc.now();
  *date = FS_DATE(now.year(), now.month(), now.day());
  *time = FS_TIME(now.hour(), now.minute(), now.second());
  *ms10 = now.second() & 1 ? 100 : 0;    /* 0..199, units of 10 ms */
}

void setup(void) {
  rtc.begin();
  FsDateTime::setCallback(dateTime);     /* BEFORE any file is created */
  sd.begin(SD_CONFIG);
}
```

There are two overloads of `setCallback`: two-argument (date, time) and
three-argument (date, time, ms10). Use the three-argument form; exFAT needs it.

Install the callback before creating files or everything is dated January 1st
`[G27]`.

---

# APPENDICES

## APPENDIX A — THE GOTCHA LIST

Consult by number. Nothing here is required reading up front.

---

**[G1] On ESP32, `File` is not defined by SdFat at all.**
SdFat tests `__has_include(<FS.h>)`. The ESP32 Arduino core always provides
`FS.h`, so SdFat declines to define its `File` typedef and emits
`#warning File not defined because __has_include(FS.h)`. Your `File` is
therefore the core's unrelated type. Confusingly, the `SdFat` volume typedef
*is* still defined, so `SdFat SD;` compiles and `File f;` then quietly gives
you the wrong type, with an error message that never mentions SdFat. Use
`SdFs` and `FsFile` explicitly and silence the warning with
`#define DISABLE_FS_H_WARNING` before including.

**[G2] `SS` and `SDCARD_SS_PIN` are core-variant symbols, not SdFat symbols.**
Whether the `adafruit_metro_esp32s3` variant defines `SDCARD_SS_PIN`, and what
`SS` resolves to, is decided by the Arduino ESP32 core. Adafruit's pinout page
documents the slot as SCK 39 / MISO 21 / MOSI 42 / CS 45. Confirm the mapping on
your build rather than assuming it: print `SD_CS_PIN` once, or run the
`QuickStart` example, which dumps all four resolved pin numbers.

**[G3] `while (!Serial)` can hang forever.**
On an S3 using native USB CDC, `Serial` stays false until a host opens the
port. A sketch blocking there looks dead on a bench supply. Either build with
`-DARDUINO_USB_CDC_ON_BOOT=1` and accept the dependency, or bound the wait as
Chapter 0 does.

**[G4] `getName` buffer minimum is 13 bytes, and truncation is silent.**
Documented: *"The array must be at least 13 bytes long. The file's name will be
truncated if the file's name is too long."* Returns the string length, or 0 on
failure. For long filenames use 64 or 256 bytes. There is no way to ask how
long the name is first — size the buffer generously.

**[G5] `LS_A` is easy to miss and most examples omit it.**
Without `LS_A` (value 1), hidden files are excluded from listings. On a card
that has been in a Mac or Windows machine, that silently hides a fair amount.

**[G6] `SD_SCK_MHZ(n)` does not clamp.**
The macro is `(1000000UL * (n))` — a bare multiplication. Example comments
claim it "selects the highest speed supported by the board that is not over
n MHz," but any clamping happens downstream in the SPI driver, not here. Do not
read it as a guaranteed ceiling negotiation.

**[G7] `ENABLE_DEDICATED_SPI` defaults to 1 on every board.**
The `SD_CONFIG` ladder in most official examples tests `HAS_TEENSY_SDIO`, then
`HAS_BUILTIN_PIO_SDIO`, then `ENABLE_DEDICATED_SPI`, falling through to
`SHARED_SPI`. On your board the first two are false and the third is true by
default, so a copied example lands on `DEDICATED_SPI`. If anything else is on
the bus, that is a real fault and it will present as intermittent corruption
rather than a clean error.

**[G8] `errorHalt` and `initErrorHalt` do not return.**
They print and hang forever. Bench diagnostics, not error handlers. Anything
that must keep running after a card failure has to test `begin()`'s return
value and branch.

**[G9] `freeClusterCount()` scans the entire FAT on first call.**
Subsequent calls are cached and fast. Never call it in a loop, never from an
ISR. Note also that it returns `int32_t`, not unsigned — a negative value
signals an error.

**[G10] Test `fatType()` as a range, not a constant.**
The official `QuickStart` example uses `if (sd.fatType() <= 32)` to distinguish
FAT16/FAT32 from exFAT. Do not assume exFAT reports any particular number.

**[G11] `cid.pnm` is not null-terminated.**
It is `char pnm[5]`. Copy into a six-byte buffer and terminate it yourself
before printing, or you will run off the end of the struct.

**[G12] Open-flag numeric values differ by platform.**
`SdFatConfig.h` sets `USE_FCNTL_H` to 1 on ESP32 and ARM, 0 on AVR. When it is
1, the flags come from the system `<fcntl.h>` and the values change
substantially:

```
  Symbol       SdFat-defined (AVR)   fcntl.h (your board)
  ----------   -------------------   --------------------
  O_RDONLY     0x00                  0x0
  O_WRONLY     0x01                  0x1
  O_RDWR       0x02                  0x2
  O_APPEND     0x08                  0x8
  O_AT_END     0x04                  O_NONBLOCK (0x4000)
  O_CREAT      0x10                  0x200
  O_TRUNC      0x20                  0x400
  O_EXCL       0x40                  0x800
  O_SYNC       0x80                  0x2000
  ----------   -------------------   --------------------
  oflag_t      uint8_t               int
```

Two consequences. Never hard-code a numeric flag. And **`oflag_t` is `int` on
your board, not `uint8_t`** — any helper function of yours that passes flags
through a `uint8_t` parameter truncates `O_CREAT` (0x200) to zero, and file
creation silently stops working with no error reported.

`O_READ` and `O_WRITE` are aliases for `O_RDONLY` and `O_WRONLY` on both paths.

**[G13] `FILE_WRITE` is not write-only and does not truncate.**
It expands to `O_RDWR | O_CREAT | O_AT_END`. If you expected a fresh empty
file, you will silently append to whatever was there. Both macros are
`#ifndef`-guarded, so an earlier header can redefine them.

**[G14] `fgets` on an over-long line splits it silently.**
Your parser then sees two fragments and mis-parses both. Guard it:

```cpp
if (line[n - 1] != '\n' && n == (int)(sizeof(line) - 1)) {
  /* line was too long for the buffer */
}
```

This is the most common silent-corruption bug in CSV reading code.

**[G15] Unsynced data is lost, and so is the directory entry.**
The file's size is not recorded until a `sync()` or `close()`. Power-cycle
before either and the file may appear to be zero bytes even though data was
written. For an unattended logger, sync on an interval and accept the
write-amplification cost.

**[G16] `seekCur` with a large negative offset wraps.**
It computes `seekSet(curPosition() + offset)` where `curPosition()` is
`uint64_t`. A negative offset larger than the current position wraps to an
enormous unsigned value. `seekSet` rejects it, so you get `false` rather than
corruption, but the reason is not obvious. Bounds-check before seeking
backwards.

**[G17] `available()` returns `int`.**
Meaningless on files above `INT_MAX` bytes. For large files use
`fileSize() - curPosition()`, both `uint64_t`. For loop control, prefer testing
`read()` against -1.

**[G18] `sd.rename()` on a currently-open path is not guarded.**
Close the file first. `file.rename()` on the open object is the safe form and
keeps the file open across the rename, including across directories.

**[G19] `mkdir` creates parents; `open` never creates directories.**
`mkdir(path, pFlag = true)` builds every missing level. `O_CREAT` on `open()`
creates only the file, so `open("logs/x.csv", O_WRONLY | O_CREAT)` fails if
`logs/` does not exist. The asymmetry catches people.

**[G20] `openNext()` resumes from the directory's current position.**
Walking the same directory twice without `dir.rewind()` in between returns
nothing on the second pass, which looks exactly like a card fault.

**[G21] The working directory is global, per-volume state.**
Any library or subsystem that calls `sd.chdir()` changes path resolution for
every subsequent relative open anywhere in your program. In anything
multi-part, use absolute paths and leave the working directory at root.

**[G22] `preAllocate` has three separate traps.**
Documented: *"Allocate contiguous clusters to an empty file. The file must be
empty with no clusters allocated. The file will contain uninitialized data."*

1. The file must be genuinely empty. `O_TRUNC` gets you there. After even one
   `print()`, `preAllocate` fails.
2. The reserved space holds **garbage, not zeroes**. This is why `truncate()`
   at close is mandatory rather than tidy — without it, readers hit leftover
   data past your last record.
3. The FAT version takes `uint32_t` (4 GiB cap); the exFAT version takes
   `uint64_t`. `FsFile` presents a `uint64_t` signature but narrows on a FAT32
   volume.

**[G23] `getError()` returns `uint8_t`, not `bool`.**
It returns `0xFF` when the underlying file pointer is null. So
`if (dir.getError())` is true both for a real error and for a directory that
was never opened. Distinguish them if it matters.

**[G24] `RingBuf` and interrupts.**
The class is documented as ISR-safe on the buffer side: print into it from an
ISR, call `writeOut()` from normal code, and `bytesUsed()` is
interrupt-protected. Do **not** call `sync()`, `begin()`, or any `FsFile`
method from an ISR.

**[G25] Forgetting `bp.sync()` loses the tail.**
Up to `BufferSize` bytes sit in RAM. `file.close()` knows nothing about the
BufferedPrint and will not flush it. Order is always `bp.sync()` then
`file.close()`.

**[G26] `BufferedPrint` float precision defaults to 2.**
Silently rounding logged data to two decimals is a quiet way to ruin a dataset.
Pass precision explicitly: `printField(d, term, 6)`.

**[G27] Install the timestamp callback before creating files.**
Otherwise files carry `FS_DEFAULT_DATE`, defined as
`FS_DATE(compileYear(), 1, 1)` — the first of January of the compile year. That
is why so many SD cards show everything dated January 1st.

Related: `FS_DATE` and `FS_TIME` are commonly called macros, including in the
library's own example comments. They are actually `static inline` functions,
so they respect argument types and will not double-evaluate. `FAT_DATE(y,m,d)`
is a real macro aliasing `FS_DATE`.

**[G28] Mixing volume and file classes fails silently, with no error code.**
`FatVolume` and `FsVolume` each keep their **own separate** static
working-volume pointer. They are unrelated classes:

```
  FatVolume::m_cwv   set by SdFat32::begin()
  FsVolume::m_cwv    set by SdFs::begin()
```

`FsBaseFile::open(path, oflag)` is implemented as:

```cpp
bool open(const char* path, oflag_t oflag = O_RDONLY) {
  return FsVolume::m_cwv && open(FsVolume::m_cwv, path, oflag);
}
```

So if you declare `SdFat32 sd;` (or an `SdFat` that resolved to `SdFat32`)
alongside `FsFile dir;`, mounting sets `FatVolume::m_cwv` and leaves
`FsVolume::m_cwv` at `nullptr`. Every path-based open then short-circuits on
that null and returns false.

The failure happens before any error-reporting code is reached:

```
  sd.begin()             succeeds, returns true
  sd.fatType()           returns 32, read directly off the volume object
  sd.card()->readCID()   works
  dir.open("/")          returns false
  sd.sdErrorCode()       returns 0 - no error was recorded
  USE_DBG_MACROS 1       prints nothing - the short-circuit is before
                         the first DBG_FAIL_MACRO
```

A healthy mount, a plausible FAT type, working card access, and no diagnostics
at all. The symptoms suggest a directory or card fault; neither is involved.

Two ways to confirm it in one build:

1. Call `dir.openRoot(&sd)`. If the classes are mismatched this will not
   compile — `no matching function for call to 'FsFile::openRoot(SdFat*)'`.
   The compile error *is* the diagnosis.
2. If it does compile and `openRoot(&sd)` succeeds where `open("/")` fails,
   the working-volume static is null.

The fix is to match the classes. `sd.chvol()` also sets `m_cwv` explicitly and
is a legitimate belt-and-braces call in multi-volume code, but it cannot bridge
mismatched class hierarchies — if the types are wrong, `chvol()` sets the wrong
static.

---

## APPENDIX B — CONFIGURATION MACROS

Edit `SdFatConfig.h`, or better, override via PlatformIO `build_flags` — a
library update silently discards source edits.

```
  SDFAT_FILE_TYPE          1 on AVR under 32K flash, else 3.
  ENABLE_DEDICATED_SPI     Defaults to 1 on ALL boards. [G7]
  USE_FCNTL_H              1 on ESP32 and ARM, 0 on AVR. [G12]
  USE_SPI_ARRAY_TRANSFER   0, 1, or 2. Improves bulk SPI rate.
  USE_UTF8_LONG_NAMES      UTF-8 filenames.
  SD_CHIP_SELECT_MODE      0 = standard GPIO (default)
                           1 = user CS function optional
                           2 = user CS function required
  USE_FAT_FILE_FLAG_CONTIGUOUS   Optimised contiguous-file access.
  FS_DEFAULT_DATE          FS_DATE(compileYear(), 1, 1)
```

Note that `SDFAT_FILE_TYPE` is the library macro. `SD_FAT_TYPE`, which appears
at the top of nearly every official example, is **not** a library macro — it is
a local convention those sketches use to switch their own `#if` blocks.
Defining it in your sketch changes nothing in the library.

Custom chip select, for a port expander: set `SD_CHIP_SELECT_MODE` to 1 or 2
and supply

```cpp
void sdCsInit (SdCsPin_t pin)              { pinMode(pin, OUTPUT); }
void sdCsWrite(SdCsPin_t pin, bool level)  { digitalWrite(pin, level); }
```

---

## APPENDIX C — TROUBLESHOOTING

```
  Symptom                          Cause                            See
  -------------------------------  -------------------------------  ----
  Sketch dead, no serial output    while (!Serial) blocking on CDC  [G3]
  begin() fails, error code set    Wrong CS; card not seated;
                                   wiring; other SPI device not
                                   deselected                       2.5
  begin() ok, fatType() == 0       Mounted but no valid partition.
                                   Reformat with SdFormatter.       7.1
  Intermittent read/write errors   SPI clock too high. Try
                                   SD_SCK_MHZ(4).                   2.2
  Errors only when display active  Shared bus, DEDICATED_SPI set    [G7]
  File empty after reset           Never synced or closed           [G15]
  Last few records missing         BufferedPrint not synced         [G25]
  File grows on every run          FILE_WRITE includes O_AT_END     [G13]
  O_CREAT seems ignored            oflag passed through a uint8_t   [G12]
  preAllocate always fails         File not empty; needs O_TRUNC    [G22]
  Garbage at end of log            No truncate() after preAllocate  [G22]
  Second directory walk empty      Missing dir.rewind()             [G20]
  open() fails on a valid path     Parent directory doesn't exist   [G19]
  open("/") false, mount fine,     Volume and file classes don't
    fatType right, NO error code   match. Check every declaration.  [G28]
    and NO debug output
  Hidden files missing from ls     LS_A not set                     [G5]
  Everything dated January 1st     Timestamp callback too late      [G27]
  Compile error mentioning File    ESP32 FS.h; use SdFs / FsFile    [G1]
  Dropped samples while logging    No preAllocate, buffer too
                                   small, or SHARED_SPI             8.1
  -------------------------------  -------------------------------  ----
```

**First thing to run when nothing works:** the `QuickStart` example. It prints
the resolved MISO/MOSI/SCK/SS pin numbers, walks you through CS selection
interactively, reports card size, FAT type, and cluster size, then lists the
card. If QuickStart works and your sketch does not, the difference is in your
sketch. Its pin dump also confirms the pin mapping described in `[G2]`.

**Second thing:** `SpiLoopBackTest`, which verifies the SPI path with no card
involved. If both fail, stop debugging filesystem code — the problem is
electrical.

---

## APPENDIX D — EXAMPLE INDEX

Under `examples/`, in rough order of usefulness for this board:

```
  QuickStart             Interactive hardware verification. Start here.
  SdInfo                 Card CID/CSD, partition table, volume type.
  DirectoryFunctions     mkdir, chdir, ls, rmdir, remove.
  OpenNext               Programmatic directory iteration.
  rename                 Both forms of rename, open and closed.
  ReadCsvFile            fgets plus strtok/strtol parsing.
  RingBufLogger          High-rate logging. Section 8.1 in code form.
  BufferedPrint          Print-throughput benchmark.
  bench                  Raw read/write throughput measurement.
  SdFormatter            Correct card formatting.
  SdErrorCodes           Prints every error code with its text.
  UnicodeFilenames       UTF-8 filename handling.
  RtcTimestampTest       Timestamp callback with an RTC.
  UserChipSelectFunction Custom CS via port expander.
  UserSPIDriver          Replacing the SPI driver entirely.
  SpiLoopBackTest        Diagnose SPI wiring without a card.
  MinimumSizeSdReader    Smallest read-only configuration.
  SoftwareSpi            Bit-banged SPI.
  examplesV1/            Legacy SdFat v1 sketches. Reference only.
```

Present but **not applicable to this board**, listed so you do not waste time:
`AvrAdcLogger`, `TeensyDmaAdcLogger`, `TeensyRtcTimestamp`, `TeensySdioDemo`,
`TeensySdioLogger`, `Rp2040SdioSetup`, `UsbKey`.

Most examples open with `#include "sdios.h"` and use `cout << F("...")`, the
library's stream-style output helper. It is optional; plain `Serial.print`
works identically.

### D.1 The full class reference

There is no hosted Doxygen site for this fork. The reference ships with the
source as HTML:

```
  .pio/libdeps/<your_env>/SdFat - Adafruit Fork/doc/SdFat.html
```

Start at the Main Page, then the Classes tab, and read `SdFs`, `FsFile`, and
`FsVolume`. Failing that, the headers themselves are well commented:
`FsLib/FsFile.h` and `FsLib/FsVolume.h`.

---

```
 ============================================================================
  END OF VOLUME I
 ============================================================================
```
