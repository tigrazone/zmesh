### Steps to Add Meshopt Decompression to `cgltf` in the `zmesh` Project

Here are **detailed implementation steps** to integrate Meshopt decompression into `cgltf` for the `zmesh` project:

---

### **1. Understand the Current Setup**

From the repository:

- `cgltf` is already integrated for parsing glTF files (`libs/cgltf`).
- `meshoptimizer` library is integrated for mesh optimization (`libs/meshoptimizer`).
- The `zcgltf` Zig module handles glTF parsing and buffer loading (`src/zcgltf.zig`).

---

### **2. Identify Meshopt Decompression Requirements**

Meshopt decompression is used to decode glTF files with the `KHR_meshopt_compression` extension.

- The `cgltf_buffer_view` struct has a field `has_meshopt_compression` to indicate compressed data.
- The `cgltf_meshopt_compression` struct contains details about the compression (buffer, offset, size, stride, etc.).

---

### **3. Modify `zcgltf` to Handle Meshopt Decompression**

Update `zcgltf` to detect and handle Meshopt-compressed data.

```zig name=src/zcgltf.zig
pub fn appendMeshPrimitive(
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

    const num_vertices: u32 = @as(u32, @intCast(prim.attributes[0].data.count));
    const num_indices: u32 = @as(u32, @intCast(prim.indices.?.count));

    // Handle Meshopt-compressed data
    if (prim.indices.?.buffer_view.has_meshopt_compression) {
        const compression = prim.indices.?.buffer_view.meshopt_compression;
        const buffer_data = compression.buffer.data[compression.offset .. compression.offset + compression.size];
        try meshoptDecodeIndices(indices, buffer_data, compression.count, compression.stride);
    } else {
        // Standard uncompressed indices
        const accessor = prim.indices.?;
        const buffer_view = accessor.buffer_view.?;
        const data = buffer_view.buffer.data[buffer_view.offset .. buffer_view.offset + buffer_view.size];
        try decodeIndices(indices, data, accessor.component_type, accessor.stride, num_indices);
    }

    // Handle Meshopt-compressed positions
    const pos_accessor = prim.attributes[0].data;
    if (pos_accessor.buffer_view.has_meshopt_compression) {
        const compression = pos_accessor.buffer_view.meshopt_compression;
        const buffer_data = compression.buffer.data[compression.offset .. compression.offset + compression.size];
        try meshoptDecodePositions(positions, buffer_data, compression.count, compression.stride);
    } else {
        // Standard uncompressed positions
        const buffer_view = pos_accessor.buffer_view.?;
        const data = buffer_view.buffer.data[buffer_view.offset .. buffer_view.offset + buffer_view.size];
        try decodePositions(positions, data, pos_accessor.stride, num_vertices);
    }

    // Handle additional attributes (normals, texcoords, tangents) similarly...
}
```

---

### **4. Implement Meshopt Decoding Functions**

Add helper functions to decode Meshopt-compressed data using the Meshopt library.

```zig name=src/meshopt.zig
const std = @import("std");

pub fn meshoptDecodeIndices(
    indices: *std.ArrayList(u32),
    data: []const u8,
    count: usize,
    stride: usize,
) !void {
    const c = @cImport({
        @cInclude("meshoptimizer.h");
    });

    try indices.ensureTotalCapacity(count);
    const result = c.meshopt_decodeIndexBuffer(
        indices.items.ptr,
        count,
        stride,
        data.ptr,
        data.len,
    );
    if (result != 0) {
        return error.MeshoptDecompressionFailed;
    }
}

pub fn meshoptDecodePositions(
    positions: *std.ArrayList([3]f32),
    data: []const u8,
    count: usize,
    stride: usize,
) !void {
    const c = @cImport({
        @cInclude("meshoptimizer.h");
    });

    try positions.ensureTotalCapacity(count);
    const result = c.meshopt_decodeVertexBuffer(
        positions.items.ptr,
        count,
        stride,
        data.ptr,
        data.len,
    );
    if (result != 0) {
        return error.MeshoptDecompressionFailed;
    }
}
```

---

### **5. Update `build.zig`**

Ensure `meshoptimizer` is compiled and linked correctly in the project.

```zig name=build.zig
zmesh_lib.addIncludePath(b.path("libs/meshoptimizer"));
zmesh_lib.addCSourceFiles(.{
    .files = &.{
        "libs/meshoptimizer/vertexcodec.cpp",
        "libs/meshoptimizer/indexcodec.cpp",
        "libs/meshoptimizer/allocator.cpp",
    },
    .flags = &.{ "-std=c++17" }, // Meshopt requires C++17
});
```

---

### **6. Test the Implementation**

Create a test glTF file with `KHR_meshopt_compression` and validate the decompression.

```zig name=src/tests/meshopt_test.zig
const std = @import("std");

test "Meshopt Decompression" {
    var indices = std.ArrayList(u32).init(std.testing.allocator);
    defer indices.deinit();

    var positions = std.ArrayList([3]f32).init(std.testing.allocator);
    defer positions.deinit();

    const compressed_data = // Load compressed buffer data here...
    try meshoptDecodeIndices(&indices, compressed_data, 1000, 4);

    const compressed_positions = // Load compressed positions here...
    try meshoptDecodePositions(&positions, compressed_positions, 1000, 12);

    // Validate results
    std.testing.expect(indices.len == 1000);
    std.testing.expect(positions.len == 1000);
}
```

---

### **7. Update Documentation**

Add instructions for enabling `KHR_meshopt_compression` support in the README.

```markdown name=README.md
### Meshopt Decompression

The `zmesh` library supports glTF files with the `KHR_meshopt_compression` extension. To enable this:

1. Ensure the `meshoptimizer` library is included in the build.
2. Use `zmesh.io.zcgltf.appendMeshPrimitive` to load compressed meshes.
```

---

### Next Steps

1. Test with various glTF files containing `KHR_meshopt_compression`.
2. Optimize performance if decompression is slow.
3. Report any issues with the integration.
