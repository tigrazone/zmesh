### Steps to Add Draco to cgltf in the Project `zmesh`

Below are the steps to integrate Draco compression with cgltf in the `zmesh` project, based on the findings from the repository:

---

#### **Step 1: Understand the Current Setup**

- The project already uses `cgltf` for glTF parsing, as seen in the `libs/cgltf` directory and multiple references in the codebase.
- The build system is defined in `build.zig`, where `cgltf` is included as a dependency.
- Draco compression support is partially referenced in `libs/cgltf/cgltf_write.h` through the `KHR_draco_mesh_compression` extension.

---

#### **Step 2: Add Draco Library**

You need the Draco library to handle mesh compression and decompression. Here’s how to include it:

1. **Download Draco Source Code**

   - Clone the [Draco GitHub repository](https://github.com/google/draco):
     ```bash
     git clone https://github.com/google/draco.git
     ```
   - Place the Draco source code in the `libs/draco` directory of the project.

2. **Update `build.zig` to Include Draco**
   Add Draco's source files and include paths to the build script:

   ```zig name=build.zig
   const draco_include_path = b.path("libs/draco/src");
   const draco_sources = &.{
       "libs/draco/src/draco/compression/decode.cc",
       "libs/draco/src/draco/compression/encode.cc",
       // Add more Draco source files as needed
   };

   zmesh_lib.addIncludePath(draco_include_path);
   zmesh_lib.addCSourceFiles(.{
       .files = draco_sources,
       .flags = &.{ "-std=c++17" }, // Draco uses C++17
   });
   ```

---

#### **Step 3: Modify cgltf to Use Draco**

The `cgltf` library already has support for the `KHR_draco_mesh_compression` extension. You need to ensure the integration is functional:

1. **Enable Draco in `cgltf_write.h`**
   Enable the `CGLTF_EXTENSION_FLAG_DRACO_MESH_COMPRESSION` flag when using cgltf.

   - This flag is already defined in `libs/cgltf/cgltf_write.h`.

2. **Add Draco Decoding in `zcgltf.zig`**
   Add a function in `zcgltf.zig` to decode Draco-compressed data using the Draco library.

   ```zig name=src/zcgltf.zig
   pub fn decodeDracoBuffer(
       draco_data: []const u8,
       indices: *std.ArrayList(u32),
       positions: *std.ArrayList([3]f32),
   ) !void {
       // Call Draco's decoding functions here
       // Example: Use FFI to call Draco's C++ decoder functions
    const c = @cImport({
        @cInclude("draco/compression/decode.h");
        @cInclude("draco/core/decoder_buffer.h");
    });

    // Create a Draco decoder buffer
    var buffer = c.draco.DecoderBuffer{};
    c.draco.DecoderBuffer_Init(&buffer, @ptrCast([*]const u8, draco_data.ptr), draco_data.len);

    // Initialize the Draco decoder
    var decoder = c.draco.Decoder{};

    // Decode the mesh
    const mesh = c.draco.Mesh{};
    const status = c.draco.Decoder_DecodeMeshFromBuffer(&decoder, &buffer, &mesh);
    if (c.draco.Status_IsOk(status) == 0) {
        return error.DecodingFailed;
    }

    // Extract indices
    const num_faces = c.draco.Mesh_NumFaces(&mesh);
    try indices.ensureTotalCapacity(num_faces * 3);
    for (var i: usize = 0; i < num_faces; i += 1) {
        var face = c.draco.Face{};
        c.draco.Mesh_GetFace(&mesh, i, &face);

        indices.appendAssumeCapacity(face.index[0]);
        indices.appendAssumeCapacity(face.index[1]);
        indices.appendAssumeCapacity(face.index[2]);
    }

    // Extract positions
    const pos_att = c.draco.Mesh_GetAttributeByUniqueId(&mesh, c.draco.POSITION);
    if (pos_att == null) {
        return error.MissingPositionAttribute;
    }
    const num_vertices = c.draco.Mesh_NumPoints(&mesh);
    try positions.ensureTotalCapacity(num_vertices);
    for (var i: usize = 0; i < num_vertices; i += 1) {
        var pos: [3]f32 = undefined;
        c.draco.AttributeDecoder_DecodeAttributeFloat(&decoder, pos_att, i, &pos[0], 3);
        positions.appendAssumeCapacity(pos);
    }

    // Clean up
    c.draco.Decoder_Delete(&decoder);
    c.draco.Mesh_Delete(&mesh);
    c.draco.DecoderBuffer_Delete(&buffer);
   }
   ```

3. **Integrate Draco Decoding in `appendMeshPrimitive`**
   Modify the `zcgltf.appendMeshPrimitive` function to check for Draco compression and call `decodeDracoBuffer` if needed.

   To integrate Draco decoding into the `zcgltf.appendMeshPrimitive` function, you need to check whether the mesh uses Draco compression (via the `KHR_draco_mesh_compression` extension) and then delegate the decoding process to the `decodeDracoBuffer` function.

Here's how you can modify the `zcgltf.appendMeshPrimitive` function:

---

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

    // Check if the primitive uses Draco compression
    if (prim.has_draco_mesh_compression) {
        const draco_comp = prim.draco_mesh_compression;
        const buffer_view = draco_comp.buffer_view.?; // Draco-compressed buffer view
        const buffer = buffer_view.buffer;

        assert(buffer.data != null);

        const draco_data = buffer.data[buffer_view.offset .. buffer_view.offset + buffer_view.size];
        try decodeDracoBuffer(draco_data, indices, positions);

        return; // Exit early since Draco decoding is complete
    }

    const num_vertices: u32 = @as(u32, @intCast(prim.attributes[0].data.count));
    const num_indices: u32 = @as(u32, @intCast(prim.indices.?.count));

    // Indices
    {
        try indices.ensureTotalCapacity(indices.items.len + num_indices);

        const accessor = prim.indices.?;
        const buffer_view = accessor.buffer_view.?;

        assert(accessor.stride == buffer_view.stride or buffer_view.stride == 0);
        assert(buffer_view.buffer.data != null);

        const data_addr = @as([*]const u8, @ptrCast(buffer_view.buffer.data)) +
            accessor.offset + buffer_view.offset;

        if (accessor.stride == 1) {
            if (accessor.component_type != .r_8u) {
                return error.InvalidIndicesAccessorComponentType;
            }
            const src = @as([*]const u8, @ptrCast(data_addr));
            var i: u32 = 0;
            while (i < num_indices) : (i += 1) {
                indices.appendAssumeCapacity(src[i]);
            }
        } else if (accessor.stride == 2) {
            if (accessor.component_type != .r_16u) {
                return error.InvalidIndicesAccessorComponentType;
            }
            const src = @as([*]const u16, @ptrCast(@alignCast(data_addr)));
            var i: u32 = 0;
            while (i < num_indices) : (i += 1) {
                indices.appendAssumeCapacity(src[i]);
            }
        } else if (accessor.stride == 4) {
            if (accessor.component_type != .r_32u) {
                return error.InvalidIndicesAccessorComponentType;
            }
            const src = @as([*]const u32, @ptrCast(@alignCast(data_addr)));
            var i: u32 = 0;
            while (i < num_indices) : (i += 1) {
                indices.appendAssumeCapacity(src[i]);
            }
        } else {
            return error.UnsupportedIndicesAccessorStride;
        }
    }

    // Positions
    {
        const accessor = prim.attributes[0].data;
        const buffer_view = accessor.buffer_view.?;

        assert(buffer_view.buffer.data != null);

        const data_addr = @as([*]const u8, @ptrCast(buffer_view.buffer.data)) +
            accessor.offset + buffer_view.offset;

        try positions.ensureTotalCapacity(positions.items.len + num_vertices);

        const src = @as([*]const [3]f32, @ptrCast(@alignCast(data_addr)));
        var i: u32 = 0;
        while (i < num_vertices) : (i += 1) {
            positions.appendAssumeCapacity(src[i]);
        }
    }

    // Handle optional attributes (normals, texcoords, tangents) here if needed...
}
```

### Key Changes:

1. **Draco Compression Check**:

   - The `has_draco_mesh_compression` flag is checked for the primitive.
   - If Draco compression is present, the `draco_mesh_compression` information is retrieved.

2. **Call `decodeDracoBuffer`**:

   - The Draco buffer data is extracted and passed to the `decodeDracoBuffer` function.
   - The `indices` and `positions` lists are populated by the decoding function.

3. **Early Exit**:

   - If Draco decoding is successful, the function immediately returns without processing further attributes.

4. **Fallback for Non-Draco Data**:
   - If the primitive does not use Draco compression, the function falls back to the standard glTF decoding logic.

---

#### **Step 4: Update Documentation**

Update the `README.md` file to include instructions for enabling Draco support, e.g., by adding a build option for Draco.

---

#### **Step 5: Test the Integration**

1. Add test cases for Draco-compressed glTF models.
2. Run the tests using the `zmesh` test infrastructure.
