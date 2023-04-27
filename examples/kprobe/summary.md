# kprobe

Attaching an eBPF program to a kernel symbol.

- Attached to the start of `sys_execve`
- Print out the number of times it has been called per second.

## Tricks

### Initialization Template

```go
// This is no-op from Linux 5.11
// Allow the current process to lock memory for eBPF resources.
if err := rlimit.RemoveMemlock(); err != nil {
    log.Fatal(err)
}

// Load pre-compiled programs and maps into the kernel.
objs := bpfObjects{}
if err := loadBpfObjects(&objs, nil); err != nil {
    log.Fatalf("loading objects: %v", err)
}
defer objs.Close()
```

### kprobe

```go
// Open a Kprobe at the entry point of the kernel function and attach the
// pre-compiled program. Each time the kernel function enters, the program
// will increment the execution counter by 1. The read loop below polls this
// map value once per second.
kp, err := link.Kprobe(fn, objs.KprobeExecve, nil)
if err != nil {
    log.Fatalf("opening kprobe: %s", err)
}
defer kp.Close()
```

### Timer

```go
// Read loop reporting the total amount of times the kernel
// function was entered, once per second.
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()

log.Println("Waiting for events..")

for range ticker.C {
    var value uint64
    if err := objs.KprobeMap.Lookup(mapKey, &value); err != nil {
        log.Fatalf("reading map: %v", err)
    }
    log.Printf("%s called %d times\n", fn, value)
}
```

### Compile bpf to go

To genearte the go file for eBPF C code, we need to include 

```go
// $BPF_CLANG and $BPF_CFLAGS are set by the Makefile.
//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cc $BPF_CLANG -cflags $BPF_CFLAGS bpf kprobe.c -- -I../headers
```

Then, we run `make container-shell` and inside the container, we run `make generate`. This will recursively run `go generate` for all sub-directories and generate the corresponding go codes.
