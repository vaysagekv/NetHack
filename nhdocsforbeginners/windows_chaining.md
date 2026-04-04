# NetHack Window Chaining System (WINCHAIN)

## Overview

The **Window Chaining System** (WINCHAIN) is an internal NetHack infrastructure that allows window function calls to be intercepted and processed through a chain of "processors" between the NetHack core and the actual window port (TTY, Curses, Qt, X11, etc.).

```
┌─────────────────────────────────────────────────────────────────┐
│                     NetHack Core                                │
│                   (game engine)                                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              -chainin (wc_chainin.c)                            │
│         Converts window_procs → chain_procs                     │
│              (adds context parameter)                           │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              Window Chain Processors                            │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐                   │
│    │ +trace  │ →  │ +custom │ →  │  ...    │                   │
│    │(logging)│    │processor│    │         │                   │
│    └─────────┘    └─────────┘    └─────────┘                   │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              -chainout (wc_chainout.c)                          │
│         Converts chain_procs → window_procs                     │
│              (removes context parameter)                        │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              Actual Window Port                                 │
│         (tty, curses, Qt, X11, win32, etc.)                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Purpose

The window chaining system enables:

1. **Debugging and Tracing** - Log all window calls for analysis
2. **Interception** - Modify or monitor window operations without changing window ports
3. **Layering** - Add functionality (profiling, filtering, etc.) transparently
4. **Testing** - Inject test harnesses between core and display

---

## Architecture

### Two API Types

The chaining system bridges two different function signature styles:

#### 1. `window_procs` API (Standard)
Used by the NetHack core and actual window ports.

```c
struct window_procs {
    void (*win_init_nhwindows)(int *, char **);
    void (*win_putstr)(winid, int, const char *);
    int (*win_nhgetch)(void);
    // ... etc
};
```

**Characteristics:**
- No context parameter
- Direct function calls
- Used by: TTY, Curses, Qt, X11, Win32, etc.

#### 2. `chain_procs` API (Chained)
Used by intermediate processors in the chain.

```c
struct chain_procs {
    void (*win_init_nhwindows)(void *, int *, char **);
    void (*win_putstr)(void *, winid, int, const char *);
    int (*win_nhgetch)(void *);
    // ... etc
};
```

**Characteristics:**
- First parameter is `void *` context (processor-specific data)
- Allows processors to maintain state
- Used by: trace, custom processors

---

## Chain Components

### 1. Chain Input (`-chainin`) - wc_chainin.c

**Location:** `win/chain/wc_chainin.c`

**Purpose:** Converts from `window_procs` API to `chain_procs` API.

**How it works:**
- Receives calls from NetHack core using standard `window_procs` signatures
- Adds a context parameter (`cibase` - pointer to `struct chainin_data`)
- Forwards to the next link in the chain using `chain_procs` signatures

```c
void chainin_putstr(winid window, int attr, const char *str)
{
    // cibase contains pointer to next processor's data and procedures
    (*cibase->nprocs->win_putstr)(cibase->ndata, window, attr, str);
}
```

**Key structure:**
```c
struct chainin_data {
    struct chain_procs *nprocs;  // Next processor's procedures
    void *ndata;                  // Next processor's data
    int linknum;                  // Position in chain
};
```

### 2. Chain Output (`-chainout`) - wc_chainout.c

**Location:** `win/chain/wc_chainout.c`

**Purpose:** Converts from `chain_procs` API back to `window_procs` API.

**How it works:**
- Receives calls from the last chain processor using `chain_procs` signatures
- Removes the context parameter
- Forwards to the actual window port using `window_procs` signatures

```c
void chainout_putstr(void *vp, winid window, int attr, const char *str)
{
    struct chainout_data *tdp = vp;
    (*tdp->nprocs->win_putstr)(window, attr, str);
}
```

### 3. Trace Processor (`+trace`) - wc_trace.c

**Location:** `win/chain/wc_trace.c`

**Purpose:** Logs all window calls to a file for debugging.

**Features:**
- Creates log file `tlog.<pid>` in trouble directory
- Logs function entry with parameters
- Logs return values
- Supports nested calls (indentation tracking)
- Logs: window creation, text output, input, menus, glyphs, etc.

**Example log output:**
```
init_nhwindows(1,*)
  create_nhwindow(STATUS)
  => 1
  create_nhwindow(MAP)
  => 2
  putstr(1, 0, 'Welcome to NetHack!'(19))
  nhgetch()
  => 'q'(113)
```

**Usage:** Add `+trace` to the windowchain option to enable logging.

---

## Chain Setup and Initialization

### Configuration

The window chain is configured via the `windowchain` option:

```
OPTIONS=windowchain:+trace
```

Or on command line:
```
nethack -windowchain:+trace
```

### Chain Assembly Process

Located in `src/windows.c`:

```c
void commit_windowchain(void)
{
    // 1. Add -chainin at the head
    // 2. Add user-specified processors (+trace, etc.)
    // 3. Add -chainout at the tail
    
    // 4. Allocate each processor's private data
    for (n = 1, p = chain; p; n++, p = p->nextlink) {
        p->linkdata = (*p->wincp->chain_routine)(WINCHAIN_ALLOC, n, 0, 0, 0);
    }
    
    // 5. Initialize each processor with next link's info
    for (n = 1, p = chain; p; n++, p = p->nextlink) {
        if (p->nextlink) {
            (*p->wincp->chain_routine)(WINCHAIN_INIT, n, p->linkdata,
                                       p->nextlink->wincp->procs,
                                       p->nextlink->linkdata);
        } else {
            // Last link points to actual window system
            (*p->wincp->chain_routine)(WINCHAIN_INIT, n, p->linkdata,
                                       gl.last_winchoice->procs, 0);
        }
    }
    
    // 6. Install chain as windowprocs
    windowprocs = *chain->wincp->procs;
}
```

### Chain Link Structure

```c
struct winlink {
    struct winlink *nextlink;     // Next processor in chain
    struct win_choices *wincp;    // Window choices entry
    void *linkdata;               // Processor's private data
};
```

---

## Processor Lifecycle

### 1. Allocation Phase (WINCHAIN_ALLOC)

Each processor's `chain_routine` is called with `WINCHAIN_ALLOC`:

```c
void *my_procs_chain(int cmd, int n, void *me, void *nextprocs, void *nextdata)
{
    switch (cmd) {
    case WINCHAIN_ALLOC:
        // Allocate and initialize processor's private data
        tdp = (struct my_data *) alloc(sizeof *tdp);
        tdp->nprocs = 0;      // Will be set in INIT
        tdp->ndata = 0;       // Will be set in INIT
        tdp->linknum = n;     // Position in chain (1, 2, 3...)
        return tdp;
    }
}
```

### 2. Initialization Phase (WINCHAIN_INIT)

Each processor is linked to the next:

```c
case WINCHAIN_INIT:
    tdp = me;
    tdp->nprocs = nextprocs;  // Procedures of next link
    tdp->ndata = nextdata;    // Data of next link
    break;
```

### 3. Runtime Operation

During the game, each processor:
1. Receives call from previous link
2. Performs its function (logging, filtering, etc.)
3. Forwards to next link via `nprocs` function pointers
4. Returns result back up the chain

---

## Creating a Custom Processor

To create a new window chain processor:

### 1. Define the Processor Structure

```c
struct myprocessor_data {
    struct chain_procs *nprocs;   // Next processor's procedures
    void *ndata;                   // Next processor's data
    int linknum;                   // Position in chain
    // Add your custom data here
    FILE *logfile;
    int call_count;
};
```

### 2. Implement Chain Routine

```c
void *myprocessor_procs_chain(int cmd, int n, void *me, 
                               void *nextprocs, void *nextdata)
{
    struct myprocessor_data *tdp = 0;
    
    switch (cmd) {
    case WINCHAIN_ALLOC:
        tdp = (struct myprocessor_data *) alloc(sizeof *tdp);
        tdp->nprocs = 0;
        tdp->ndata = 0;
        tdp->linknum = n;
        tdp->call_count = 0;
        return tdp;
        
    case WINCHAIN_INIT:
        tdp = me;
        tdp->nprocs = nextprocs;
        tdp->ndata = nextdata;
        return tdp;
        
    default:
        panic("myprocessor_procs_chain: bad cmd");
    }
    return tdp;
}
```

### 3. Implement Window Functions

```c
void myprocessor_putstr(void *vp, winid window, int attr, const char *str)
{
    struct myprocessor_data *tdp = vp;
    
    // Do something before the call
    tdp->call_count++;
    
    // Forward to next processor
    (*tdp->nprocs->win_putstr)(tdp->ndata, window, attr, str);
    
    // Do something after the call (if needed)
}

int myprocessor_nhgetch(void *vp)
{
    struct myprocessor_data *tdp = vp;
    int rv;
    
    // Forward to next processor
    rv = (*tdp->nprocs->win_nhgetch)(tdp->ndata);
    
    // Do something with the result
    return rv;
}
```

### 4. Define the Procedures Structure

```c
struct chain_procs myprocessor_procs = {
    "+myprocessor",              // Name (must start with '+')
    wp_myprocessor,              // wp_id from enum
    0,                           // wincap
    0,                           // wincap2
    {1, 1, 1, 1, 1, 1, 1, 1,     // has_color
     1, 1, 1, 1, 1, 1, 1, 1},
    myprocessor_init_nhwindows,
    myprocessor_player_selection,
    // ... all other functions
};
```

### 5. Register in windows.c

Add to `winchoices[]` array in `src/windows.c`:

```c
#ifdef WINCHAIN
    { &chainin_procs, chainin_procs_init, chainin_procs_chain },
    { (struct window_procs *) &chainout_procs, chainout_procs_init, 
      chainout_procs_chain },
    { (struct window_procs *) &trace_procs, trace_procs_init, 
      trace_procs_chain },
    { (struct window_procs *) &myprocessor_procs, myprocessor_procs_init, 
      myprocessor_procs_chain },  // Add your processor here
#endif
```

### 6. Add wp_id to enum

In `include/winprocs.h`:

```c
enum wp_ids { 
    wp_tty = 1, 
    wp_X11, 
    wp_Qt, 
    wp_mswin, 
    wp_curses,
    wp_chainin, 
    wp_chainout, 
    // ...
    wp_trace,
    wp_myprocessor,  // Add your processor's ID
};
```

---

## Key Files

| File | Purpose |
|------|---------|
| `win/chain/wc_chainin.c` | Converts window_procs → chain_procs |
| `win/chain/wc_chainout.c` | Converts chain_procs → window_procs |
| `win/chain/wc_trace.c` | Trace/logging processor |
| `include/winprocs.h` | Defines window_procs and chain_procs structures |
| `src/windows.c` | Chain assembly and management |
| `src/options.c` | windowchain option handling |

---

## Important Notes

### Naming Convention

- Processors starting with `'+'` (e.g., `+trace`) are user-selectable chain processors
- Processors starting with `'-'` (e.g., `-chainin`, `-chainout`) are system-internal
- Regular names (e.g., `tty`, `qt`) are actual window systems

### Chain Order

The chain is always assembled as:
```
-chainin → +user_processors... → -chainout → actual_window_system
```

### Window Capabilities

The `wincap` and `wincap2` fields from the actual window system are preserved and propagated to the chain's entry point so that capability checks work correctly.

### No Re-initialization

The actual window system's `ini_routine` is NOT called again during chain commit - it's already been initialized when the window type was first selected.

---

## Use Cases

### 1. Debugging Window Issues

```
OPTIONS=windowtype:qt,windowchain:+trace
```

This logs all window calls when using the Qt interface, helping identify where problems occur.

### 2. Performance Profiling

A custom processor could time each window call:

```c
void profile_putstr(void *vp, winid w, int attr, const char *str)
{
    clock_t start = clock();
    (*tdp->nprocs->win_putstr)(tdp->ndata, w, attr, str);
    clock_t end = clock();
    log_timing("putstr", end - start);
}
```

### 3. Input Recording/Replay

A processor could record all user input for later replay:

```c
int record_nhgetch(void *vp)
{
    int ch = (*tdp->nprocs->win_nhgetch)(tdp->ndata);
    fprintf(record_file, "%d\n", ch);
    return ch;
}
```

### 4. Remote Display

A processor could forward window calls over a network connection to a remote display.

---

## Summary

The window chaining system is a powerful infrastructure that:
- **Decouples** the NetHack core from window system details
- **Enables** debugging through the trace processor
- **Allows** transparent addition of functionality
- **Maintains** compatibility with all existing window ports
- **Provides** a clean API for extending window processing

It's not a windowing system itself, but rather a **middleware layer** that sits between the game and the display, enabling powerful introspection and extension capabilities.
