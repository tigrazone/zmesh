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
   }
   ```

3. **Integrate Draco Decoding in `appendMeshPrimitive`**
   Modify the `zcgltf.appendMeshPrimitive` function to check for Draco compression and call `decodeDracoBuffer` if needed.

---

#### **Step 4: Update Documentation**

Update the `README.md` file to include instructions for enabling Draco support, e.g., by adding a build option for Draco.

---

#### **Step 5: Test the Integration**

1. Add test cases for Draco-compressed glTF models.
2. Run the tests using the `zmesh` test infrastructure.

---

Let me know if you need more detailed code examples or further assistance!
