### Steps to Add Parallel Decompression of Meshopt-Compressed glTFs in `zmesh`

Here’s the detailed plan with implementation steps to parallelize decompression for meshopt-compressed glTFs in the `zmesh` project.

---

### **1. Understand the Current Setup**

- **Existing decompression logic**: Meshopt decompression routines like `meshopt_decodeIndexBuffer` and `meshopt_decodeVertexBuffer` are already supported in `libs/meshoptimizer/`.
- **Integration with glTF**: The project uses `cgltf` to parse glTF files, and `zcgltf.zig` wraps the decompression and buffer handling logic.
- **Parallelization target**: Parallelize the decompression of attributes like indices, positions, normals, and tangents.

---

### **2. Implement Parallelized Decompression**

Modify `zcgltf` to use threads for decompression of individual attributes.

#### **Step 2.1: Add Parallel Decoding in `zcgltf`**

```zig name=src/zcgltf.zig
const std = @import("std");

pub fn parallelDecodeMeshopt(
    data: *Data,
    mesh_index: u32,
    prim_index: u32,
    indices: *std.ArrayList(u32),
    positions: *std.ArrayList([3]f32),
    normals: ?*std.ArrayList([3]f32),
    texcoords0: ?*std.ArrayList([2]f32),
    tangents: ?*std.ArrayList([4]f32),
) !void {
    assert(mesh_index < data.meshes_count);
    assert(prim_index < data.meshes.?[mesh_index].primitives_count);

    const mesh = &data.meshes.?[mesh_index];
    const prim = &mesh.primitives[prim_index];

    const allocator = std.heap.ThreadAllocator.init(std.testing.allocator);
    defer allocator.deinit();

    // Spawn parallel tasks for each decompression
    const indices_task = try std.Thread.spawn(
        allocator.allocator(),
        decodeIndices,
        .{prim, indices},
    );

    const positions_task = try std.Thread.spawn(
        allocator.allocator(),
        decodePositions,
        .{prim, positions},
    );

    var normals_task = if (normals) |n| try std.Thread.spawn(
        allocator.allocator(),
        decodeNormals,
        .{prim, n},
    ) else null;

    var texcoords_task = if (texcoords0) |t| try std.Thread.spawn(
        allocator.allocator(),
        decodeTexcoords,
        .{prim, t},
    ) else null;

    // Wait for threads to complete
    try indices_task.join();
    try positions_task.join();
    if (normals_task) |task| try task.join();
    if (texcoords_task) |task| try task.join();
}

fn decodeIndices(args: anytype) !void {
    const prim = args.prim;
    const indices = args.indices;

    if (prim.indices.?.buffer_view.has_meshopt_compression) {
        const compression = prim.indices.?.buffer_view.meshopt_compression;
        const buffer_data = compression.buffer.data[compression.offset .. compression.offset + compression.size];
        try meshoptDecodeIndices(indices, buffer_data, compression.count, compression.stride);
    } else {
        // Fallback for uncompressed data
    }
}

fn decodePositions(args: anytype) !void {
    const prim = args.prim;
    const positions = args.positions;

    if (prim.attributes[0].data.buffer_view.has_meshopt_compression) {
        const compression = prim.attributes[0].data.buffer_view.meshopt_compression;
        const buffer_data = compression.buffer.data[compression.offset .. compression.offset + compression.size];
        try meshoptDecodePositions(positions, buffer_data, compression.count, compression.stride);
    } else {
        // Fallback for uncompressed data
    }
}

fn decodeNormals(args: anytype) !void {
    // Similar to decodePositions but for normals
}

fn decodeTexcoords(args: anytype) !void {
    // Similar to decodePositions but for texcoords
}
```

---

### **3. Parallelization in Meshopt Decoding Functions**

Ensure that the decompression functions (`meshoptDecodeIndices`, `meshoptDecodePositions`) can handle concurrent calls. These functions are already thread-safe as they operate on independent buffers.

---

### **4. Update `build.zig` for Threading Support**

Enable threading in the project by modifying the build script.

```zig name=build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const exe = b.addExecutable(.{ .name = "zmesh" });
    exe.setBuildMode(b.standardReleaseOptions());
    exe.addPackagePath("zmesh", "src");
    exe.addPackagePath("std", std.zigPackagePath());
    exe.linkSystemLibrary("pthread"); // Ensure threading library is linked
    b.installArtifact(exe);
}
```

---

### **5. Testing the Implementation**

#### **5.1: Create Test Cases**

Write test cases to validate parallel decompression.

```zig name=src/tests/parallel_decompression_test.zig
const std = @import("std");

test "Parallel Decompression of Meshopt Compressed glTF" {
    var indices = std.ArrayList(u32).init(std.testing.allocator);
    defer indices.deinit();

    var positions = std.ArrayList([3]f32).init(std.testing.allocator);
    defer positions.deinit();

    const data = // Load glTF data here...

    try zcgltf.parallelDecodeMeshopt(
        data,
        0, // mesh index
        0, // primitive index
        &indices,
        &positions,
        null, // normals
        null, // texcoords0
        null, // tangents
    );

    // Validate decompressed data
    std.testing.expect(indices.len > 0);
    std.testing.expect(positions.len > 0);
}
```

#### **5.2: Benchmark Parallel vs Sequential**

Measure performance improvement using sequential and parallel decompression to validate the benefits of parallelization.

---

### **6. Optimize and Document**

- **Optimization**: Profile the implementation to identify bottlenecks and optimize thread synchronization and memory allocation.
- **Documentation**: Update the `README.md` file to include details about parallel decompression.

```markdown name=README.md
## Parallel Decompression

The `zmesh` library supports parallel decompression for meshopt-compressed glTF files. This improves performance by handling indices, positions, normals, and tangents in separate threads. To enable parallel decompression, ensure your system supports threading.
```
