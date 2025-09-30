# C Exam Mega‑Guide
_Generated: 2025-09-30_

This is a **comprehensive, exam-ready** guide consolidating everything we covered: **Makefile, GDB,
stack/heap + Valgrind, pointers, strings, files (fgets, fscanf, fgetc, fprintf, fread, fwrite),
structs/packing/pointer arithmetic, qsort comparator, ASCII, tracing strategies**, and **worked
exam‑style examples**. Treat this as your “open-book” reference.

---

## 0) “Program + Input → Exact Output” (60‑second tracing method)
1) **Mark passes/loops** and file resets: circle `fgetc/fscanf/fgets` passes, underline `fseek(fp,0,SEEK_SET)`.
2) **Write initial state**: show variables & pointer targets; draw file cursor at **0** (start of file).
3) **Execute in order**: each assignment, each `*p = …`, each read/write (file cursor moves by bytes).
4) At every **`printf`**, compute the value **right then** (don’t postpone).
5) For file questions, pre‑build 3 lists from the input quickly:
   - **Chars** for `fgetc` (remember ASCII: `'0'..'9'`=48..57, space=32, `\n`=10, `'-'`=45)
   - **Ints** for `fscanf("%d")` (skips whitespace)
   - **Lines** for `fgets` (keeps `\n` if it fits; NUL terminates)

**Scratch template**
```
Vars: a=?, b=?, i=?, j=?
Pointers: p->?, q->?
File cursor: 0 (mark jumps at each fread/fseek/fgetc)

PASS A (fgetc): 1:'1'(49) 2:'0'(48) 3:'8'(56) ...
PASS B (%d)   : 1:1084 2:-23 3:456 ...
PASS C (fgets): 1:"1084 -23 ECE\n" 2:"456\n" ...
At printf(...): value = __
```
---

## 1) Pointers (glue topic used everywhere)
### 1.1 Basics
- Declaration: `int *p;` → **p holds an address** of an `int`.
- Address-of: `p = &x;` • Dereference: `*p = 5;` changes `x`.
- Arithmetic: `p + k` moves by `k*sizeof(*p)` bytes (int*=4, double*=8, char*=1).
- Arrays decay to pointers: in expressions, `arr` ≈ `&arr[0]` (but `sizeof(arr)` is total bytes when `arr` is in scope as an array).
- Struct access: `ps->x` == `(*ps).x`.

### 1.2 Pass‑by‑value vs pass‑by‑address
```c
void set_bad(int x){ x = 5; }              // caller unchanged
void set_ok (int *px){ *px = 5; }          // caller's int changed
```
**Swap**:
```c
void swap_ok(int *a,int *b){ int t=*a; *a=*b; *b=t; }   // ✅
void swap_bad(int *a,int *b){ int *t=a; a=b; b=t; }     // ❌ only reassigns locals
```

### 1.3 Const‑correctness (quick)
- `const int *p` → can’t change `*p`; may move `p`.
- `int *const p` → `p` fixed; can change `*p`.
- `const int *const p` → neither moves nor mutates.

---

## 2) Strings (C strings are NUL‑terminated char arrays)
### 2.1 Core functions
- `strlen(s)` → count chars before `\0` (doesn’t remove newline).
- `strcpy(dst, src)` → copies until `\0`; **dst must be big enough**.
- `strcmp(a,b)` → 0 if equal; <0 if `a<b`; >0 if `a>b` (lexicographic).
- `strchr(s,'c')` → first occurrence or `NULL`.
- `strstr(hay, needle)` → first substring match or `NULL`.
- `strdup(s)` → `malloc`’d copy (POSIX).

**Danger:** `strncpy` may not NUL‑terminate; prefer `snprintf` if you need capping.

### 2.2 Array vs pointer in structs
```c
typedef struct { int id; char *name; } S1;      // pointer (needs allocation)
typedef struct { int id; char  name[64]; } S2;  // inline buffer (copied with struct)
```
- **Shallow copy** (with `char *`): struct assignment copies the **pointer value** ⇒ two structs can point to the **same buffer**.
- **Deep copy**: allocate a separate buffer then copy characters.

**Your exam‑style example**
```c
typedef struct { int ID; char *name; } Student;
Student *sp = malloc(2*sizeof *sp);
sp[0].name = malloc(64);
strcpy(sp[0].name, "Purdue");
sp[1] = sp[0];                   // shallow: now sp[1].name == sp[0].name (same address)
strcpy(sp[1].name, "ECE264");   // overwrites shared buffer ⇒ both read "ECE264"
// Final: sp[0].ID=264; sp[1].ID=2017; both names "ECE264"
```
**Fix deep copy (independent names):**
```c
sp[1].name = malloc(64);
strcpy(sp[1].name, sp[0].name);
```

### 2.3 Counting word occurrences in a line
```c
// non-overlapping
for (char *p=line; (p=strstr(p, word)); p += strlen(word)) count++;

// overlapping (rare)
for (char *p=line; (p=strstr(p, word)); p++) count++;
```

---

## 3) Files (text + binary) and the classic three‑pass pattern
### 3.1 APIs you’ll use
- `fopen(path, "r"/"w"/"a"/"rb"/"wb")` → `FILE*`
- `fclose(fp)`
- `fgetc(fp)` → **int** (char code) or `EOF` (usually −1)
- `fputc(c, fp)`
- `fgets(buf, n, fp)` → reads **at most n−1 chars**, keeps `\n` if it fits, then NUL
- `fputs(s, fp)`
- `fscanf(fp,"%d",&x)` → reads next int token (**skips whitespace**) → returns items matched
- `fprintf(fp,...)` → formatted write
- `fread(ptr,size,count,fp)` / `fwrite(ptr,size,count,fp)` → binary blocks
- `fseek(fp, off, SEEK_SET/CUR/END)` → returns **0** on success
- `ftell(fp)` → `long` position
- `rewind(fp)` → goto start, clear EOF/error

### 3.2 Three‑pass template (like your exams)
```c
// PASS 1: fgetc (chars)
for (int ch,i=1; (ch=fgetc(fp))!=EOF; i++)
    printf("%3d: ch=%d '%c'\n", i, ch, (char)ch);
fseek(fp,0,SEEK_SET);

// PASS 2: fscanf %d (ints)
for (int v,i=1; fscanf(fp,"%d",&v)==1; i++)
    printf("%3d: val=%d\n", i, v);
fseek(fp,0,SEEK_SET);

// PASS 3: fgets (lines)
char buf[256];
while (fgets(buf,sizeof buf,fp))
    printf("buff=%s", buf); // newline preserved if it fit
```
**EOF detail:** `fgetc` returns `EOF` at end → typically −1; printing `%d` shows −1.

### 3.3 Byte math you did
- Suppose file holds **100 ints**; exam states `sizeof(int)=4` ⇒ total **400 bytes**.
- If we read 50 ints (200), `fseek(..., sizeof(int), SEEK_CUR)` (4), then read 1 int (4), total consumed **208** ⇒ **192 bytes left**.

### 3.4 `%d` vs `%c` whitespace behavior
- `%d` and `%s` **skip** leading whitespace.
- `%c` **does not** (use `" %c"` to skip preceding whitespace).

---

## 4) Structs, packing, addresses, and qsort comparator
### 4.1 Packing
- `#pragma pack(1)` ⇒ **no padding**; struct size = sum of fields.
- Without packing, alignment may increase size (e.g., double @ 8‑byte boundary).

**Packed example**:
```c
#pragma pack(1)
typedef struct { int x,y,z; double t; } V; // size 20: x@0 y@4 z@8 t@12
```

### 4.2 Address of a field (formula to memorize)
```
&arr[i].field = base + i*sizeof(struct) + offset(field)
```
- `offset(field)` = sum of sizes of **earlier** fields (under current packing).

**Double‑based vector (packed) example**
- `typedef struct { double x,y,z,t; } DV;` (packed) size=32; offsets x@0 y@8 z@16 t@24.
- If `&vptr[0] = 1000`, then `&vptr[2] = 1000 + 2*32 = 1064`, so `&vptr[2].z = 1064+16 = **1080**`.

**Pointer from a field example**
```c
typedef struct { int x,y,z,t; } VI; // 16 bytes
VI varr[10];
int *p = &varr[2].y;    // base + 2*16 + 4
int v = p[4];           // += 4 ints => + 16 bytes => &varr[3].y → value 2*3=6
```

### 4.3 Copy vs original
```c
V v = varr[3];  // copy
v.x = -264;     // only the copy changes; varr[3].x still 3 → (3 - (-264)) = 267
```

### 4.4 qsort comparator (correct form)
```c
int cmp(const void *a, const void *b){
    int x = *(const int*)a, y = *(const int*)b;
    return (x>y) - (x<y);
}
```
**Wrong patterns**: comparing addresses themselves, or writing through `const int*` (compile error).

---

## 5) Stack vs Heap (lifetime) + typical pitfalls
### 5.1 Stack
- Each function call pushes a **stack frame** (parameters by value, locals). Pops on return.
- **Never return** a pointer/ref to a local.

```c
char *bad(){ char buf[16]; return buf; } // ❌ dangling
```

### 5.2 Heap
- Manual lifetime. One successful `malloc` → exactly **one** `free`.
- After `free(p)`, **don’t** dereference; set to `NULL` if you’ll test it.

### 5.3 Use‑after‑free / double‑free / leaks
- Use‑after‑free: accessing freed memory (Valgrind: “Invalid read/write of size…” with freed block).
- Double‑free: calling `free` twice on same pointer.
- Leaks: *definitely lost* (unreachable), *still reachable* (not freed but reachable at exit).

**Leak tally from your example**: Student array (2 × 12 = 24 bytes) + one name buffer (64) ⇒ **88 bytes** leaked if never freed.

---

## 6) GDB — line breakpoints, backtrace, and clarity diagrams
**Breakpoints stop *before* the line executes.**  
- Example (break at line 16 in a recursive `f1`):  
  1st hit: `#0 f1(5)` (before calling `f1(4)`) → frames: `f1(5), f2(5), main`.  
  2nd hit: `#0 f1(4)` (before calling `f1(3)`) → frames add one `f1`.  
  3rd hit: `#0 f1(3)` (before calling `f1(2)`) → frames: `f1(3), f1(4), f1(5), f2(5), main` (total **5**).

**Useful commands (extended)**
- Break: `b file.c:LINE`, `b func`, `b func if i==5`
- Control: `r`, `c`, `n`, `s`, `finish`, `until file.c:LINE`
- Inspect: `bt`, `frame N`, `up`, `down`, `info locals`, `info args`, `p expr`
- Memory: `x/10xb ptr` (bytes), `x/10xd ptr` (ints), `x/s pstr` (C string)
- TUI: `layout src`, `tui enable`

---

## 7) Valgrind — how to read the dump fast
Run with symbols: compile `-g` then
```
valgrind --leak-check=full --track-origins=yes ./prog args
```
- **Invalid read/write of size N at ... (file:line)** → where you used memory incorrectly.
- Follow lines below to see **where the block was alloc’d** and **its size** (often the wrong-sized `malloc`).

**Classic pointer‑size bug**
```c
array2d = malloc(5 * sizeof(int));      // ❌ 5 pointers on 64-bit need 5*8
array2d = malloc(5 * sizeof *array2d);  // ✅ correct
```

---

## 8) Makefile — beyond the basics
**Variables**: `=`, `:=`, `+=`, `?=`  
**Auto‑vars**: `$@`, `$<`, `$^`, `$*`, `$(@D)`, `$(@F)`  
**Recipe prefixes**: `@` silent, `-` ignore errors, `+` allow recursive make under `-n`  
**Phony**: `.PHONY: clean testall …`
**Deps from headers**:
```make
CFLAGS += -MMD -MP
DEPS := $(OBJS:.o=.d)
-include $(DEPS)
```
**Testing with diff (stops on mismatch)**
```make
testadd: main
	./main 4 5 A > add1.out
	diff add1.out add1.correct   # non-zero -> recipe aborts
```

---

## 9) ASCII cheat (handy for fgetc)
- `'0'..'9'` → 48..57
- space `' '` → 32
- newline `\n` → 10
- minus `'-'` → 45

---

## 10) Worked mini‑examples
### 10.1 File “skip and read”
- File contains ints `0..99` (4 bytes each).
- Read 50 ints → next unread is 50; `fseek(...,4,SEEK_CUR)` skips 50; next `fread` returns **51**.
- Bytes remaining: 400 − (200+4+4) = **192**; final `fgetc` at EOF → **−1**.

### 10.2 Struct offset seek
```c
#pragma pack(1)
typedef struct { int x,y,z; double t; } V; // size 20, z@8
// fseek(fp, 48, SEEK_SET) => 20*2 + 8 => &varr[2].z; if z=3*i, read 6.
```

### 10.3 Pointer from field
```c
typedef struct { int x,y,z,t; } VI; // 16 bytes
int *p = &varr[2].y;   // base + 2*16 + 4
printf("%d\n", p[4]);  // +4 ints => varr[3].y => 2*3 = 6
```

### 10.4 Shallow‑copy behavior
```c
Student *sp = malloc(2*sizeof *sp);
sp[0].name = malloc(64);
strcpy(sp[0].name, "Purdue");
sp[1] = sp[0];
strcpy(sp[1].name, "ECE264"); // both show ECE264
```

---

## 11) One‑page mental checklist
**Files**: fgetc int/EOF; `%d` skips ws; `fgets` keeps `\n` + NUL; `fseek` success=0; EOF prints −1.  
**Pointers**: `&` address; `*` deref; `p+1` jumps `sizeof(*p)`; `ps->x==(*ps).x`; beware NULL.  
**Structs/pack**: size=sum with pack(1); `&arr[i].field = base + i*sizeof(T) + offset(field)`.  
**Strings**: NUL‑terminated; `strcpy` overwrites dest; shallow copy shares buffer; deep copy allocates.  
**GDB**: breakpoints stop **before** line; `bt` = frames; count from `#0`.  
**Makefile**: `$(SRCS:%.c=%.o)`, `.c.o:`, `-c`, `$@/$</$^`, `diff` stops on mismatch.

Good luck — you’ve got this!
