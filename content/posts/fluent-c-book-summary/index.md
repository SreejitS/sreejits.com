---
title: "Fluent C: Book Summary & Review"
date: 2025-03-04
tags:
- c
- embedded
- book-review
- firmware
- best-practices
draft: false
---

##### *Applying the principles, practices and patterns from the book*

#### Intro : Purpose of this writeup

I've been reading the excellent book, [Fluent C](https://oreil.ly/EUzfG) from [Christopher Preschern](https://preschern.azurewebsites.net/). I recommend this book to embedded/firmware devs looking to level up their C programming skills to a professional grade. In this post I'll start with C code, that looks like something that I would have written when I graduated from college(many years ago ;) ), and will systematically apply the principles suggested in the book one by one, making it more robust and maintainable. We will do it in a fashion such that it builds upon the previous knowledge.

This post serves the purpose of being summary of the book-bookmark now, return when your next embedded project demands production grade C rigor.

All credits go to Christopher Preschern for writing the book.

Chapter 1 : Error Handling

Let's say we have the task of processing bytes stored in a file that is stored in SD card. After thinking for a while, we can come up with a basic version like the following

```c
void init_sd_and_process_file(const char *filename) {
  SDHandle *sd = sd_init();  
  if (sd) {  
    FILE* fp = sd_open(filename);
    if (fp) {  
      uint8_t buffer[256];  
      size_t bytes_read;

      while ((bytes_read = fread(buffer, 1, sizeof(buffer), fp)) > 0) {
        for (int i = 0; i < bytes_read; i++) {
          if (buffer[i] == 0xFF && SD_CARD_TYPE == CARD_SDHC) {
            processSpecialByte(buffer[i]);
          } else {
            processNormalByte(buffer[i]);
          }
        }
      }
      fclose(fp);
    } else {
      printf("File open failed");
    }
    sd_free(sd);
  } else {
    printf("SD init failed");  
  }
}

```

Now, let's identify the issues at hand and try to resolve them. On close observation you can probably tell that this function would not scale well, if, let's say, in future the file is present in remote FTP server or if the file is changed to JSON. The key reason is that this function *has several responsibilities and hence will be tied down to only these responsibilities* and hence not flexible. Let's look into the first principle and apply to the above code.

##### 1.a. Function Split

> Split the functions into many parts, each part being useful on its own.

```c
static void process_file(FILE *fp) {
  uint8_t buffer[256];  
  size_t bytes_read;

  while ((bytes_read = fread(buffer, 1, sizeof(buffer), fp)) > 0) {
    for (int i = 0; i < bytes_read; i++) {
      if (buffer[i] == 0xFF && SD_CARD_TYPE == CARD_SDHC) {
        processSpecialByte(buffer[i]);
      } else {
        processNormalByte(buffer[i]);
      }
    }
  }
}

void init_sd_and_process_file(const char *filename) {
  SDHandle *sd = sd_init();

  if (sd) {  
    FILE* fp = sd_open(filename);
    
    if (fp) {
      process_file(fp);
      fclose(fp);
    } else {
      printf("File open failed");
    }

    sd_free(sd);
  } else {
    printf("SD init failed");  
  }
}

```

Now the depth of if-else has reduced and the file specific processing is separated into its own function, making it independent and modular. Our main code is much smaller than before.

Still, the nested if-else structure makes the code harder to maintain, and the error handling doesn't ensure proper resource cleanup. This could lead to resource leaks if initialization fails partway. Let's see how to tackle this

##### 1.b. Guard Clause

> Check for all the mandatory pre-conditions and return immediately if it is not met

```c
static void process_file(FILE *fp) {
  uint8_t buffer[256];  
  size_t bytes_read;

  while ((bytes_read = fread(buffer, 1, sizeof(buffer), fp)) > 0) {
    for (int i = 0; i < bytes_read; i++) {
      if (buffer[i] == 0xFF && SD_CARD_TYPE == CARD_SDHC) {
        processSpecialByte(buffer[i]);
      } else {
        processNormalByte(buffer[i]);
      }
    }
  }
}

void init_sd_and_process_file(const char *filename) {
  SDHandle *sd = sd_init();
  if (!sd) {
    return;
  }

  FILE* fp = sd_open(filename);
  if (!fp) {
    sd_free(sd);
    return;
  }

  process_file(fp);
  fclose(fp);
  sd_free(sd);
}

```

Now our main code is readable. We could return more descriptive error information instead of just returning(We dive deep into this topic in Chapter 2(Todo:add link)).

In case of complex code logic, we may still end up with nested if-else and/or duplicated error handling and cleanup code, making code unmaintainable again. Then in this case, use the next principle.

##### 1.c Samurai Principle \[skipped\]

> Return from a function victorious or do not return at all. Abort the function when error is encountered.
>
> *I personally found this principle not really useful for embedded systems where high availability, safety is required.*

##### 1.d. Goto Error Handling

> Put all the resource cleanup and error handling at the end of the function. If some functions errors out, use *goto* to jump to the error handling/cleanup code

```c
static void process_file(FILE *fp) {
  uint8_t buffer[256];  
  size_t bytes_read;

  while ((bytes_read = fread(buffer, 1, sizeof(buffer), fp)) > 0) {
    for (int i = 0; i < bytes_read; i++) {
      if (buffer[i] == 0xFF && SD_CARD_TYPE == CARD_SDHC) {
        processSpecialByte(buffer[i]);
      } else {
        processNormalByte(buffer[i]);
      }
    }
  }
}

void init_sd_and_process_file(const char *filename) {
  SDHandle *sd = NULL;
  FILE *fp = NULL;

  sd = sd_init();
  if (!sd) {
    return;
  }

  fp = sd_open(filename);
  if (!fp) {
    goto cleanup_sd;
  }

  process_file(fp);

cleanup:
  if (fp) {
    fclose(fp);
  }

cleanup_sd:
  if (sd) {
    sd_free(sd);
  }
}

```

The function may look longer but ensures all allocated resources are freed before exiting. It makes it more maintainable by avoiding duplicated cleanup logic.

The if-else cascade still feels like it’s begging for further refinement—it’s not quite as clean or elegant as it could be. There’s room to simplify and make it even more streamlined. Let's see how we can achieve that.

##### 1.d. Object Based Error Handling

> Instead of putting multiple responsibilities in one function, put initialization and cleanup in separate functions, similar to constructors/destructors in OOP.

```c
/* Initialization with Built-in Validation */
static SDHandle* init_sd_card(void) {
    // Guard against initialization failure
    SDHandle* sd = sd_init();
    if (!sd) {
        log_error("SD card initialization failed");
        return NULL;
    }
    return sd;
}

static FILE* open_sd_file(SDHandle* sd, const char* filename) {
    // Guard against invalid parameters
    if (!sd || !filename) {
        log_error("Invalid parameters for file open");
        return NULL;
    }
    
    // Guard against open failure
    FILE* fp = sd_open(filename);
    if (!fp) {
        log_error("Failed to open file: %s", filename);
        return NULL;
    }
    return fp;
}

/* Processing with Built-in Safeguards */
static void process_file(FILE* fp) {
    // Guard against invalid file pointer
    if (!fp) {
        log_error("Invalid file handle for processing");
        return;
    }

    uint8_t buffer[256];
    size_t bytes_read;

    while ((bytes_read = fread(buffer, 1, sizeof(buffer), fp)) > 0) {
        for (int i = 0; i < bytes_read; i++) {
            if (buffer[i] == 0xFF && SD_CARD_TYPE == CARD_SDHC) {
                processSpecialByte(buffer[i]);
            } else {
                processNormalByte(buffer[i]);
            }
        }
    }
}

/* Cleanup Functions (NULL-safe) */
static void cleanup_file(FILE* fp) {
    if (fp) fclose(fp);
}

static void cleanup_sd_card(SDHandle* sd) {
    if (sd) sd_free(sd);
}

/* Simplified Main Flow */
void init_sd_and_process_file(const char* filename) {
    SDHandle* sd = init_sd_card();
    FILE* fp = open_sd_file(sd, filename);
    
    process_file(fp);
    
    cleanup_file(fp);
    cleanup_sd_card(sd);
}
```

Just look at the final function, the intent is clear and will be clear even after you see it after many months. The code is modular, split into clean and working individual units, and is maintainable.

Looking at the first version of the code and the final one, it’s amazing how much better it got just by taking things step by step and applying good practices. The difference is huge—it’s like going from a messy draft to a clean, polished final version. Night and day, really!

Chapter 2 : Returning Error Information

##### 2.a. Return Status Codes

> Return all the possible errors to the caller.

Let's say the task at hand is to parse a file. The most basic way of returning error information is to return all the status codes.

```c
// Enumerate every possible error (clutters code)
typedef enum {
    PARSER_OK,
    FILE_NOT_FOUND,
    INVALID_HEADER,
    DATA_CORRUPTED,
    MEMORY_ERROR
} ParserStatus;

ParserStatus parse_file(const char* filename, int** data_out) {
    FILE* file = fopen(filename, "r");
    if (!file) return FILE_NOT_FOUND;
    
    char header[4];
    if (fread(header, 1, 4, file) != 4 || header[0] != 'M') {
        fclose(file);
        return INVALID_HEADER;
    }
    
    *data_out = malloc(sizeof(int) * 100);
    if (!*data_out) {
        fclose(file);
        return MEMORY_ERROR;
    }
    
    // ... (parse data)
    fclose(file);
    return PARSER_OK;
}
```

The caller code will look something like this

```c
int* data;
ParserStatus status = parse_file("data.bin", &data);
    
switch (status) { // Long, repetitive handling
  case FILE_NOT_FOUND: printf("File missing!\n"); break;
  case INVALID_HEADER: printf("Invalid format!\n"); break;
  case MEMORY_ERROR:   printf("Out of memory!\n"); break;
  case DATA_CORRUPTED: printf("Data broken!\n"); break;
  case PARSER_OK:      printf("Success!\n"); break;
}
```

The issue with this approach is that it bloats the callee code, since it returns all the possible status codes, as well as the caller code, as it has to check for each kind of possible statuses. A better way would be to return only those error that the caller can do something about.

##### 2.b. Return Relevant Errors

> Return only those error codes that the caller can take actionable action, else return a common error.

```c
// Only return errors the caller can fix (e.g., file issues)
typedef enum { PARSER_OK, FILE_ERROR, MEMORY_ERROR } ParserStatus;

ParserStatus parse_file(const char* filename, int** data_out) {
    FILE* file = fopen(filename, "r");
    if (!file) return FILE_ERROR;
    
    // Assume header is valid for simplicity
    *data_out = malloc(sizeof(int) * 100);
    if (!*data_out) {
        fclose(file);
        return MEMORY_ERROR;
    }
    
    fclose(file);
    return PARSER_OK;
}
```

The caller code in this case will look something like this

```c
int* data;
ParserStatus status = parse_file("data.bin", &data);
    
if (status == FILE_ERROR)  printf("Check file!\n");
if (status == MEMORY_ERROR) printf("Free memory!\n");
if (status == PARSER_OK)    printf("Success!\n");
```

The obvious benefits are smaller caller code. Any other error that happens should be considered implementation specific, and a common error code can be used to indicate that. This kind of grouping works fine during the development phase of the code.

Using standard C return values for errors complicates returning other data, often requiring passing variables by reference. A better approach is to return a special value (e.g., `NULL` or `-1`) for errors and the actual data on success. This simplifies the design and avoids unnecessary complexity.

##### 2.c. Return Special Values

> Use return value to return the data computed by function and reserve special values for errors

```c
// Return data directly; use NULL/-1 for errors
int* parse_file(const char* filename) {
    FILE* file = fopen(filename, "r");
    if (!file) return NULL;
    
    int* data = malloc(sizeof(int) * 100);
    if (!data) {
        fclose(file);
        return NULL;
    }
    
    // ... (parse data)
    fclose(file);
    return data;
}
```

```c
int* data = parse_file("data.bin");
if (!data) {
  printf("Error: File or memory issue!\n"); // Ambiguous!
  return 1;
  }
printf("Success!\n");
free(data);
```

This simplifies the caller's code but sacrifices error details. A balanced approach is to log errors and use `assert` for irrecoverable issues. This is especially useful during debugging and development, providing clarity without overcomplicating the design.

2.d. Log Errors

> Log the errors using relevant channels(`printf` over UART, saving in SD card, displaying QR code etc. )

```c
// Use asserts for unrecoverable errors during development
int* parse_file(const char* filename) {
    FILE* file = fopen(filename, "r");
    if (file == NULL) {
        fprintf(stderr, "Error (%s:%d): File not found\n", __FILE__, __LINE__);
        assert(file != NULL && "File must exist");
    }
    
    int* data = malloc(sizeof(int) * 100);
    if (data == NULL) {
        fprintf(stderr, "Error (%s:%d): Out of memory\n", __FILE__, __LINE__);
        assert(data != NULL && "Out of memory");
    }
    
    // ... (parse data)
    fclose(file);
    return data;
}
```

```c
int* data = parse_file("data.bin");
if (data == NULL) {
fprintf(stderr, "Error (%s:%d): Parse failed\n", __FILE__,__LINE__);
  return 1;
}
printf("Success!\n");
free(data);
```

Now asserts tell you exactly what happened and you can always choose to disable them in release binary, and replace them with a reboot in case of embedded target, after making sure that such reboot does not cause the target to be stuck in boot loop.

This summarizes few of the ways one can return useful error information to the user.

Chapter 3 : Memory Management \[skipped\]

Chapter 4 : Returning Data from C Functions

##### 4.a. Return Value

When writing functions in C, returning data efficiently and safely is a common challenge. While the `return` statement works for simple cases, real-world scenarios often demand more flexibility. In this post, we’ll explore six techniques to return data from C functions, their use cases, and trade-offs.

> Return single value from function, ideal for atomic operations.

```c
int add(int a, int b) {  
    return a + b;  
}  

int result = add(3, 5); // result = 8  
```

This is the most basic way of returning data from functions. As the callers gets it's own copy of the data, the function is re-entrant and this suitable for multithreaded environment.

However, C only supports returning only single type of object via this method. Let's see how we can return many values from function.

##### 4.b. Out-parameters

> Use pointers to "return" multiple related values through function parameters.

```c
void calculate_ops(int a, int b, int *sum, int *product) {  
    *sum = a + b;  
    *product = a * b;  
}  

int s, p;  
calculate_ops(4, 5, &s, &p); // s=9, p=20  
```

Using pointers as function arguments, we can emulate the by-reference arguments. Now all the related values can be copied to the function arguments. In a multi-threaded environment, use synchronization primitives to make sure the data is not changed during the copying.

This issues with this approach is, after a point returning many values like this makes the function signature long, and is not clear in first look that they are out-parameters. A better and clean way to returning related data would be packing them into a structure and returning them.

##### 4.c. Aggregate Instance

> Bundle all the related data into a single structure and return it

```c
typedef struct {  
    int sum;  
    int product;  
} MathResult;  

MathResult calculate_result(int a, int b) {  
    return (MathResult){a + b, a * b};  
}  

MathResult result = calculate_result(2, 3);  
```

As C supports returning object of single type, you can create a custom type using struct and bundle all the related data in to this struct and return it.

The structs live in stack if passed like this, so will consume large amounts of stack or if passed onto nested functions. We still can use pointer to struct and treat it as out-parameter, but then we would have to be clear in the functions API as to who is responsible for allocating and cleaning up the memory pointed by the pointers. Let's say we have lot of data that needs to returned, but the data does not change in between invocation, then we can use the immutable instance to save the copying overhead.

##### 4.d. Immutable Instance

> Keep all the immutable data or the data that you want to share but want to make sure the caller does not modify into static memory and pass `const` pointer to the caller

```c
typedef struct module_info {
    const char *module_name;  // Immutable string pointer
    const char *author;       // Immutable string pointer
} module_info;

// Returns read-only module info from static memory
const module_info *get_module_info() {
    static const module_info info = {  // Static const storage
        .module_name = "SecurityModule",
        .author = "SecureDevTeam"
    };
    return &info;
}

void print_module_info() {
    const module_info *info = get_module_info();
    printf("Module Name: %s\nAuthor: %s\n", 
           info->module_name, 
           info->author);
}
```

This approach of using static memory with `const` pointers offers robust data protection through compile-time enforcement of immutability, eliminating memory leaks and allocation overhead while ensuring thread-safe initialization. It’s ideal for fixed configurations, constants, or shared metadata (like module names or author info) that must remain unmodified.

However, it sacrifices runtime flexibility—data cannot be updated dynamically, occupies permanent memory, and relies on compile-time initialization. While `const` prevents accidental changes, determined misuse via unsafe casts can bypass protections. Choose this for simple, stable data; avoid it for dynamic or large datasets needing frequent updates.

Large and changing datasets are best dealt with caller allocated buffers.

##### 4.e. Caller Owned Buffer

> The caller passes the pointer to the buffer and the size, the callee fills in the buffer after checking for overflows.

```c
void fill_buffer(int a, int b, int *buffer) {  
    buffer[0] = a + b;  
    buffer[1] = a * b;  
}  

int buffer[2];  
fill_buffer(4, 5, buffer); // buffer = [9, 20]  
```

The callee can return large data that is changing at runtime. The caller can access the data in safe and re-entrant manner as it is the sole owner of the data.

The caller has to know the size of the buffer beforehand. In some cases, this may not be possible, so the callee handles the responsibility of allocating the buffer whose size is known at runtime.

##### 4.f. Callee Allocated Buffer

> Allocate the buffer in the callee code and copy the data and return pointer to the caller.

```c
int* create_dynamic_result(int a, int b) {  
    int *result = malloc(2 * sizeof(int));  
    if (result) {  
        result[0] = a + b;  
        result[1] = a * b;  
    }  
    return result;  
}  

int *dynamic = create_dynamic_result(2, 3);  
free(dynamic); // Responsibility lies with the caller
```

This approach is suitable for dynamic and variable-sized data. The pointer and the size could be returned as aggregate instance also.

However, the caller is responsible for freeing of resource, forgetting to do so will result in memory leak. One way of dealing with this is to document in function APIs and/or to have a dedicated function for cleanup, making it evident that the pointer needs to be freed.

Chapter 5 : Data Lifetimes and Ownership

This chapter is about structuring the C program around OOP-like objects, which are basically instances of data structures. In C such instances are nothing more than named region of storage. Hence the focus will be on who will be responsible for creating and destroying the instance.

##### 5.a. Stateless Software Module

> Keep the functions simple and do not build state info, so that the functions can be called and the result does not depend on the previous function calls

Let's start with the most basic example of adding two numbers and see how we can build on top of this. A simple implementation will not build up state information. The caller and callee will share info using return values.

```c
// 1. Stateless Module - No retained state
void MathUtil_add(int a, int b) {
    printf("Stateless: %d + %d = %d\n", a, b, a + b);
}
```

It is not easy to provide all the required functionality using such simple interface. You would have to branch to other patterns in order to share some sort of state information.

##### 5.b. Software Module with Global State

> If there is no caller-dependent state information, then have a file global static instance, which is common for the callee module to operate on.

In such implementation, the global state is hidden from the caller and is managed transparently. File-global instances are protected using synchronization primitives for multithreading in the callee module.

```c
// 2. Global State Module
static int sum= 0;
void MathUtil_addThis(int val) {
    printf("Sum: %d -> %d\n", sum, sum+val);
}
```

This is a form of anti-pattern called Singleton and should be generally avoided. But in some cases like global unique resources(e.g.. SysTick timer), this can be used with precaution. There may be some initialization for the global instance that needs to be handled during boot/first call. Also race the static variables are prone to race conditions if used without mutual exclusion.

Again this may not be sufficient for complex situations where you would have caller specific state information that needs to be passed along. In this case we can use the next pattern.

##### 5.c. Caller Owned Instance

> Have the caller pass an instance, that will build up the required state information.

By doing so, multiple callers/threads can call the required function as each caller will have it's own instance to be operated on. The callee will not have any information about the lifetime of the instance, so the allocation/cleanup has to be done by the caller. Applied to our toy example, will lead to the following

```c
// 3. Caller-owned Instance
typedef struct {
    int value;
} Adder;
void Adder_init(Adder* a, int init_val) { a->value = init_val; }
void Adder_add(Adder* a, int x) { 
    a->value += x;
    printf("Caller-Owned: New value: %d\n", a->value);
}
```

A more practical example for embedded systems would be using multiple UARTs for getting GNSS data and sending network data. In this case we can have different caller handles for the UART.

This deals with multiple instances for multiple callers. There could be a case when the same instance has to be used by multiple callers, each caller might add/remove state information. In this case we use the next pattern.

##### 5.d. Shared Instance

> Let the software module have the ownership of instances, and it needs to handled as different callers operate on same instance.

In this paradigm, the *software module* retains ownership of instances while allowing multiple callers to operate on shared resources. Unlike caller-owned patterns, initialization and cleanup logic leverages the module's internal state to reuse existing resources.

```c
// 4. Shared Instance
typedef struct {
    int id;
} Resource;
static Resource* shared = NULL;
Resource* Resource_get() {
    if (!shared) {
        shared = malloc(sizeof(Resource));
        shared->id = 42;
        printf("Shared: Instance created\n");
    }
    return shared;
}
void Resource_free() {
    free(shared);
    shared = NULL;
}
```

Chapter 6 : Flexible APIs

Aim of this chapter is to create interfaces, that stand the test of time, makes it easy for future modifications, yet keeping it simple enough to not break the existing code. Note: No need to apply all the patterns, just apply the one that seems apt for the scenario, as implementing adds a complexity, that is only justifiable if the functionality achieved saves you from future headache.

##### 6.a. Header Files

This is the most basic way for providing interface to your module and I assume this is understood, honestly I don't know why the author added this as a pattern.

Let's say you want to write a driver for an IMU. For a small project, without much future maintenance, one can come up with something like the following

```c
// imu_basic.h
typedef struct { int x, y, z; } ImuData;

void imu_init(void);
ImuData imu_read(void);
void imu_write(uint8_t reg, uint8_t val);
void imu_config(uint8_t settings);
```

This would be enough for some projects but has the limitations of only supporting one IMU, is hardcoded to support only one implementation, let's say I2C or SPI. And this cannot handle any runtime configurations. Let's say for a safety critical application, you have redundant IMUs. Then it can be dealt better by using handles.

##### 6.b. Handle

If you have to share state and operate on shared resources, then expose a function that create the context on which the caller will operate, and the caller will have all the necessary state info. In our case of multiple IMUs, it will look something like this

```c
// Forward declaration of opaque handle
typedef struct ImuHandle ImuHandle;

// Interface functions
ImuHandle* imu_create(uint8_t addr, uint32_t speed);
void imu_destroy(ImuHandle* h);
ImuData imu_read(ImuHandle* h);
void imu_write(ImuHandle* h, uint8_t reg, uint8_t val);
```

```c
//Usage
ImuHandle* imu1 = imu_create(0x68, 400000);
ImuData data = imu_read(imu1);
imu_write(imu1, 0x1B, 0x00);  // Example register write
imu_destroy(imu1);
```

Now it can handle different IMU's and maintain state in between calls, but still will work with the hardcoded communication protocol. Let's say you have one IMU talking over I2C and another speaking SPI, then the above pattern can be extended to support this use case too by using dynamic interface.

##### 6.c. Dynamic Interface

Have a common interface for the different types of functionalities and the caller has to provide the specific function for that functionality. All this is achieved using function pointers.

```c
// imu_dynamic.h
typedef struct ImuHandle ImuHandle;

// Function prototypes
typedef void (*ReadFn)(ImuHandle* h, ImuData* data);
typedef void (*WriteFn)(ImuHandle* h, uint8_t reg, uint8_t val);

struct ImuDriverFunction {
    ReadFn read;
    WriteFn write;
};

// Interface functions
ImuHandle* imu_create(uint8_t addr, uint32_t speed, struct ImuDriverFunction f);
void imu_destroy(ImuHandle* h);
ImuData imu_read(ImuHandle* h);
void imu_write(ImuHandle* h, uint8_t reg, uint8_t val);
```

Now you can handle different IMU's from different vendors, though now the code has become little complex. So be careful to only apply the pattern if there is actual need for it.

##### 6.d. Function Control

Great! Now let's say we are thinking of our task in a more generic way and we know that the product we are working on now has IMU and will contain various other sensors, like the pressure sensors, environmental sensors etc. as it evolves. Let's try to have a common interface for reading/writing/configuring to such "similar" class of sensors, even though the exact code logic for interacting with the sensors are different, the overall logic is same.

We can do by introducing an argument to the function, that adds the information about the which logic to execute.

```c
typedef struct SensorHandle SensorHandle;

// Function prototypes
typedef void (*ReadFn)(SensorHandle* h, SensorData* data);
typedef void (*WriteFn)(SensorHandle* h, uint8_t reg, uint8_t val);
typedef void (*Ctl)(Sensorhandle *h, void *param)

struct SensorDriverFunction {
    ReadFn read;
    WriteFn write;
    Ctl ioCtl
};

// Common interface
SensorHandle* sensor_create(void* config, struct SensorDriverFunction fns);
void sensor_destroy(SensorHandle* h);
void sensor_read(SensorHandle* h, SensorData* data);
void sensor_write(SensorHandle* h, uint8_t reg, uint8_t val);
int sensor_ioctl(SensorHandle* h, void* param);

```

This pattern makes it possible to add new sensors without changing the core functionality, is vendor neutral. Also has runtime control using function pointers. This is powerful and flexible pattern for products requiring field upgrades, but definitely has added complexity to it.

Chapter 7 : Flexible Iterator Interfaces \[skipped\]

Chapter 8 : Organizing Files in Modular Programs

This chapter was different from all the other chapters, as this focused more on the file directory structure rather than the actual code. The core focus is more on creating a folder structure that makes it easy for many developers to work on the codebase in the future by making it more modular and uncoupled. So, each of the following chapters will walk you through the examples and mention the issues in the current structure that will set the stage for the next solution. Let's say we the following folder structure.

```c
/app
├── filereader.c
├── filereader.h
├── hash.h
├── hash.c
└── main.c
```

##### 8.a. Software Module Directories

As the number of files increases, the above basic structure will be filled by many files, so a common way is to put the relevant source and header file under single folder named after the functionality. Let's say we need many kinds of hash implementation in our app.

```c
/app
├── filereader
│   ├── filereader.c
│   └── filereader.h
├── adler
│   ├── adlerhash.c
│   └── adlerhash.h
├── bernstein
│   ├── bernstein.c
│   └── bernstein.h
└── main.c
```

This is well structured. The issue is that the other header files can be accessed by using relative path, which brings along an unnecessary dependency of changing all the source files incase any folder in the relative path is changed. The solution for this issue is using a global include directory.

##### 8.b. Global Include Directory

Add a common folder `include/` which contains all the header files used by other code. If any header file is used by internal source file, then no need to include it there.

```c
/app
├── include
│   ├── filereader.h
│   ├── adlerhash.h
│   └── bernstein.h
├── filereader
│   └── filereader.c
├── adler
│   └── adlerhash.c
├── bernstein
│   └── bernstein.c
└── main.c
```

Now it is very clear which header files are supposed to be used by other code and which are internal and there is no need for relative folder names in the include. This works well for small to medium sized codebase. As the size grows, putting all the header files under a single folder will likely make it very crowded and difficult to understand the dependencies.

##### 8.c. Self-Contained Components

As the codebase grows, we can break our codebase into "app" units that can are self contained and can be deployed together. For this, identify source and header files that have similar functionality and put them under a common directory and put the header file under a designated sub-directory. Think of this as having multiple "apps" that have loose coupling with other modules, but strong coupling within itself. This then can be developed and tested individually. An example of this is the following

```c
hashlibrary
├── include
│   ├── adlerhash.h
│   └── bernstein.h
├── adler
│   └── adlerhash.c
└── bernstein
    └── bernstein.c
fileapp
├── include
│   └── filereader.h
└── filereader
    └── filereader.c
otherapp
├── include
│   └── otherapp.h
└── app
    └── app.c
main.c
```

Now the codebase is very well structured and can be scaled as the codebase and the team size increases.

##### 8.d. API Copy

API copy makes sense only if you very large codebase and different teams are actively developing together. In this case, you can split the codebase into separate repositories and copy the API's and binaries. This will help in versioning/testing/deploying separate parts of codebase independently. Make sure to freeze the APIs at the header file level.

For our example, we can probably split the hash library into its separate repository, it would look something like the following.

```c
hashlibrary
├── include
│   ├── adlerhash.h
│   └── bernstein.h
└── hashLibrary.a # build artifact from another repo
fileapp
├── include
│   └── filereader.h
└── filereader
    └── filereader.c
otherapp
├── include
│   └── otherapp.h
└── app
    └── app.c
main.c
```

```c
Repository B:
hashlibrary
├── include
│   ├── adlerhash.h
│   └── bernstein.h
├── adler
│   └── adlerhash.c
└── bernstein
    └── bernstein.c
```

Remember that it is not necessary to apply all the above techniques, it really depends on the codebase and the team size. You may very well have a good structure after applying few of the steps, you can stop after that. But if you face the issues described then applying the next step will help in scaling.

Chapter 9 : Escaping \#ifdef Hell

As a product offering matures, the codebase matures alongside, eventually supporting various hardware variants and various features for those variants. In C, generally this involves sprinkling \#ifdef all over, although pretty harmful for the first few times, the codebase starts diverging for each variant. Understanding the program logic becomes riddled with finding the matching \#endif for each \#ifdef and testing all the kinds of possible combinations for the variants. Before you know you are in \#ifdef hell. We have few patterns to avoid this hell or help refactor incase you are already in this hell. Let's say we have the following legacy code to refactor

```c
#include <stdio.h>

void save_data(const char* text) {
    const char* filename;
    
    /* Platform-specific paths */
    #ifdef WINDOWS
    filename = "C:\\data\\output.txt";
    #else
    filename = "/var/data/output.txt";
    #endif

    /* File handling */
    #ifdef WINDOWS
    FILE* f = fopen(filename, "w+b");
    #else
    FILE* f = fopen(filename, "w+");
    #endif

    if (!f) {
        #ifdef LOGGING
        #ifdef WINDOWS
        printf("Windows open failed\n");
        #else
        perror("Linux open failed");
        #endif
        #endif
        return;
    }

    fputs(text, f);
    fclose(f);
}
```

##### 9.a. Avoid Variants

> Use standard functions available in all platforms, for example POSIX functions.

For the given example we can use common functions available across platforms

```c
// Before: Platform-specific modes
#ifdef WINDOWS
"w+b" vs "w+"

// After: Use standard binary mode
fopen(filename, "wb+");  // Valid on all platforms
```

##### 9.b. Isolated Primitives

> Isolate the main programming logic and the logic for handling the variants into separate functions such that the main logic contains only the platform-independent code.

Separate the path generation logic from the main code flow

```c
const char* get_output_path() {
    #ifdef WINDOWS
    return "C:\\data\\output.txt";
    #else
    return "/var/data/output.txt";
    #endif
}
```

##### 9.c. Atomic Primitives

> Handle single variant in a function, generally, let the feature variant in turn call the platform specific functions needed.

Split error handling from the main program logic

```c
void log_error(const char* context) {
    #ifdef LOGGING
    #ifdef WINDOWS
    printf("%s failed\n", context);
    #else
    perror(context);
    #endif
    #endif
}
```

##### 9.d. Abstraction Layer

> Provide platform-independent API in the header file and put all the platform specific code in the source file. This abstracts away all the platform independent logic away from the main program flow.

Create platform-agnostic header (`file_io.h`):

```c
// Unified interface
const char* get_output_path();
void log_error(const char* context);
```

##### 9.e. Split Variant Implementation

> Create separate .c file for each variant and choose which to compile using MACROS in build logic i.e. Makefiles/IDE settings.

Separate platform implementations  
`windows_io.c`:

```c
#include "file_io.h"

const char* get_output_path() { 
    return "C:\\data\\output.txt"; 
}

void log_error(const char* context) {
    #ifdef LOGGING
    printf("%s failed\n", context);
    #endif
}
```

`linux_io.c`:

```c
#include "file_io.h"

const char* get_output_path() { 
    return "/var/data/output.txt"; 
}

void log_error(const char* context) {
    #ifdef LOGGING
    perror(context);
    #endif
}
```

Final refactored code

```c
#include <stdio.h>
#include "file_io.h"  // Abstraction layer

void save_data(const char* text) {
    const char* filename = get_output_path();
    FILE* f = fopen(filename, "wb+");
    
    if (!f) {
        log_error("File open");
        return;
    }
    
    fputs(text, f);
    fclose(f);
}
```

Now we have the following improvements

- Zero `#ifdef`s in main business logic
- Platform-specific details contained in:
  - Separate implementation files (`windows_io.c`/`linux_io.c`)
  - Isolated primitive functions (`get_output_path()`)
- Each function has single responsibility
- Build system control using Makefiles
