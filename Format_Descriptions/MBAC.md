# MBAC File Format Specification (Revised)

## Abstract

The MBAC format contains geometry data, referred to in M3D as Figures. No official documentation for the format is available. It is a binary serialization of the documented BAC format which is used in the content pipeline. A MBAC file is often accompanied by a MTRA file which contains animation data.

MBAC files don't store any texture names, but they have a notion of texture indices.

## Known implementations

All known implementations are in native code.

- MascotCapsule SDK (M3DConverter, PVMicro)
- Sony Ericsson WTK emulator (in zayitlib.dll)
- MEXA emulator (v3, v4)

- Official implementation used in cell phones (Micro3D library)

- Unnoficial implementations used in emulators.
- Kemulator nnmod (original native implementation and lwjgl modified woesss implementation)
- J2ME-Loader (woesss implementation)



## Overall structure

- Header(s)
- Vertices
- Normals (if present)
- Polygons (types: T3, T4, F3, F4)
- Bones/Segments
- 20-byte encrypted trailer

## Format versions

Known versions of the format, along with allowed data encodings, are listed below.

| Version | vertexformats | normalformats | polygonformats | boneformats | Seen in |
|---------|---------------|---------------|----------------|-------------|---------|
| 3 | 1 | 0 (= N/A) | 1 | 1 | Legacy games |
| 4 | 1, 2 | 0, 1, 2 | 1, 2 | 1 | |
| 5 | 1, 2 | 0, 1, 2 | 1, 2, 3 | 1 | Burning Tires, GoF, Deep, Blades & Magic, GoF 2 |

## Format description

All fields are in little-endian.

### Main Header

```c
char magic[2] = {'M', 'B'};     // 0x4D42 little-endian
uint16_t formatversion;

if (formatversion > 3) {
    uint8_t vertexformat;       // 1 = uncompressed, 2 = compressed
    uint8_t normalformat;       // 0 = none, 1 = uncompressed, 2 = compressed
    uint8_t polygonformat;      // 1, 2, or 3
    uint8_t boneformat;         // Always 1 in known files
}

uint16_t num_vertices;
uint16_t num_polyT3;            // Textured triangles
uint16_t num_polyT4;            // Textured quads
uint16_t num_bones;
```

### Extended Header (polygonformat >= 3)

For polygonformat >= 3, additional header data follows:

```c
uint16_t num_polyF3;            // Flat-shaded triangles
uint16_t num_polyF4;            // Flat-shaded quads
uint16_t material_count;        // Number of materials
uint16_t max_attribute;         // Related to polygon attributes
uint16_t num_colors;            // Number of vertex colors

// Material/attribute table (meaning partially unknown)
repeat(max_attribute) {
    uint16_t attribute_id;
    uint16_t attribute_flags;

    repeat(material_count) {
        uint16_t material_param1;
        uint16_t material_param2;
    }
}
```

---

## Vertices

### Vertex Format 1 (Uncompressed)

```c
repeat(num_vertices) {
    int16_t x;
    int16_t y;
    int16_t z;
}
```

Coordinates are in fixed-point format. Scale factor varies by content; typical scale is 1/256 for world units.

### Vertex Format 2 (Compressed)

Vertices are packed in blocks of up to 64 each. These blocks form a bitstream and are not necessarily byte-aligned. The entire bitstream does start on byte boundary and, if necessary, is padded to also end on one.

```c
align uint8_t;

bitstream {
    have_vertices := 0;
    
    while (have_vertices < num_vertices) {
        uint(6) count_minus_one;
        uint(2) range;

        num_vertices_in_block := count_minus_one + 1;
        
        // Bits per component based on range
        bits_per_component := [8, 10, 13, 16][range];

        repeat(num_vertices_in_block) {
            sint(bits_per_component) x;
            sint(bits_per_component) y;
            sint(bits_per_component) z;
        }

        have_vertices := have_vertices + num_vertices_in_block;
    }
}

align uint8_t;  // Pad to byte boundary
```

---

## Normals

### Normal Format 0

No normal data present. Normals should be calculated from face geometry.

### Normal Format 1 (Uncompressed)

```c
repeat(num_vertices) {
    int16_t nx;     // Fixed-point 4.12 (-4096 to 4096 = -1.0 to 1.0)
    int16_t ny;
    int16_t nz;
}
```

### Normal Format 2 (Compressed)

Normals are compressed using a unit sphere encoding. Each normal is either stored as a direction index or as X/Y components with Z reconstructed.

```c
// Predefined direction table (6 axis-aligned normals)
static const int8_t NORM_TABLE[6][3] = {
    { 64,   0,   0},    // +X
    {  0,  64,   0},    // +Y
    {  0,   0,  64},    // +Z
    {-64,   0,   0},    // -X
    {  0, -64,   0},    // -Y
    {  0,   0, -64}     // -Z
};

repeat(num_vertices) {
    int8_t nx;      // Signed, range -64 to 63 (maps to -1.0 to ~0.984)
    int8_t ny;
    int8_t nz_or_sign;
}
```

Decoding logic:
```c
if (nx == 64) {
    // Special case: use predefined direction
    direction_index := (ny & 0x07);  // 0-5
    normal := NORM_TABLE[direction_index];
} else {
    // Calculate Z from unit sphere constraint
    nx_float := nx / 64.0;
    ny_float := ny / 64.0;
    nz_squared := 1.0 - nx_float² - ny_float²;
    
    if (nz_squared > 0) {
        nz_float := sqrt(nz_squared);
    } else {
        nz_float := 0;  // Clamp for numerical stability
    }
    
    if (nz_or_sign < 0) {
        nz_float := -nz_float;
    }
}
```

**Note:** Some files have been observed with `nx² + ny² > 1`, which may indicate malformed data or a different interpretation.

---

## Polygons

### Polygon Format 1 (Uncompressed)

#### Textured Triangles (T3)

```c
repeat(num_polyT3) {
    uint16_t flags;
    uint16_t v0_index;      // Pre-multiplied by 3 (divide by 3 for actual index)
    uint16_t v1_index;
    uint16_t v2_index;
    
    uint8_t u0;             // UV coordinates (0-127 for 128x128 texture)
    uint8_t v0;
    uint8_t u1;
    uint8_t v1;
    uint8_t u2;
    uint8_t v2;
}
// Total: 14 bytes per triangle
```

#### Textured Quads (T4)

```c
repeat(num_polyT4) {
    uint16_t flags;
    uint16_t v0_index;
    uint16_t v1_index;
    uint16_t v2_index;
    uint16_t v3_index;
    
    uint8_t u0, v0;
    uint8_t u1, v1;
    uint8_t u2, v2;
    uint8_t u3, v3;
}
// Total: 18 bytes per quad
```

#### Polygon Flags

```c
#define POLY_FLAG_QUAD          0x8000  // Set for quads (T4/F4)
#define POLY_FLAG_TRANSPARENT   0x0001  // Enable alpha transparency
#define POLY_FLAG_ADDITIVE      0x0010  // Additive blending mode
// Other flags: TBD
```

### Polygon Format 2 (Compressed)

Bit-packed format with configurable field widths.

```c
bitstream {
    uint(8) flag_bits;          // Bits per flags field
    uint(8) vertex_index_bits;  // Bits per vertex index
    
    repeat(num_polyT3) {
        uint(flag_bits) flags;
        uint(vertex_index_bits) v0;
        uint(vertex_index_bits) v1;
        uint(vertex_index_bits) v2;
        uint(7) u0;
        uint(7) v0_uv;
        uint(7) u1;
        uint(7) v1_uv;
        uint(7) u2;
        uint(7) v2_uv;
    }
    
    repeat(num_polyT4) {
        uint(flag_bits) flags;  // OR with 0x8000 for quad flag
        uint(vertex_index_bits) v0;
        uint(vertex_index_bits) v1;
        uint(vertex_index_bits) v2;
        uint(vertex_index_bits) v3;
        uint(7) u0, v0_uv;
        uint(7) u1, v1_uv;
        uint(7) u2, v2_uv;
        uint(7) u3, v3_uv;
    }
}

align uint8_t;
```

### Polygon Format 3 (Extended Compressed)

Similar to format 2 but with additional UV precision control.

```c
bitstream {
    uint(8) flag_bits;          // Bits per flags field
    uint(8) vertex_index_bits;  // Bits per vertex index
    uint(8) uv_bits;            // Bits per UV component (typically 7)
    uint(8) reserved;           // Unknown purpose
    
    repeat(num_polyT3) {
        uint(flag_bits) flags;
        uint(vertex_index_bits) v0;
        uint(vertex_index_bits) v1;
        uint(vertex_index_bits) v2;
        uint(uv_bits) u0;
        uint(uv_bits) v0_uv;
        uint(uv_bits) u1;
        uint(uv_bits) v1_uv;
        uint(uv_bits) u2;
        uint(uv_bits) v2_uv;
    }
    
    repeat(num_polyT4) {
        uint(flag_bits) flags;
        uint(vertex_index_bits) v0;
        uint(vertex_index_bits) v1;
        uint(vertex_index_bits) v2;
        uint(vertex_index_bits) v3;
        repeat(4) {
            uint(uv_bits) u;
            uint(uv_bits) v;
        }
    }
    
    // F-polygons (flat-shaded) follow if num_polyF3/F4 > 0
    // Structure similar but without UV coordinates
}

align uint8_t;
```

UV coordinates are in pixels relative to texture size. For 7-bit UVs with 128x128 textures, divide by 127.0 for normalized coordinates.

---

## Bones/Segments

### Bone Format 1

```c
repeat(num_bones) {
    uint16_t num_vertices;      // Vertices belonging to this bone
    int16_t parent_index;       // -1 for root, otherwise parent bone index
    
    // 3x4 transformation matrix in fixed-point 4.12 format
    int32_t matrix[12];         // Row-major: [m00,m01,m02,m03, m10,m11,m12,m13, m20,m21,m22,m23]
}
// Total: 52 bytes per bone
```

Matrix layout:
```
| m[0]  m[1]  m[2]  m[3]  |   | Xx  Yx  Zx  Tx |
| m[4]  m[5]  m[6]  m[7]  | = | Xy  Yy  Zy  Ty |
| m[8]  m[9]  m[10] m[11] |   | Xz  Yz  Zz  Tz |
```

- Columns 0-2: Rotation/scale (typically unit scale)
- Column 3: Translation
- Values are fixed-point: divide by 4096.0 to get float

Vertex assignment is sequential: bone 0 owns vertices 0 to (num_vertices[0]-1), bone 1 owns the next num_vertices[1] vertices, etc.

---

## 20-byte Encrypted Trailer

Used for content protection/verification.

```c
struct Trailer {
    uint8_t key1[2];        // XOR key for data1
    uint8_t data1[8];       // Encrypted identifier
    uint8_t key2[2];        // XOR key for data2  
    uint8_t data2[8];       // Encrypted identifier (should match data1 after decryption)
};
```

Decryption:
```c
for (int i = 0; i < 8; i++) {
    decrypted1[i] = data1[i] ^ key1[i % 2];
    decrypted2[i] = data2[i] ^ key2[i % 2];
}
// decrypted1 should equal decrypted2
```

Known identifiers:
- `HI000000` - HI Corporation (stock M3DConverter)
- `SE000000` - Sony Ericsson (Abyss engine games)
- `BN000000` - Bandai Namco
- `MX000000` - Unknown
- `VF000000` - Vodafone/Unknown

The identifier may be specified in the source BAC file during conversion.

---

## Runtime Structure (from decompiled code)

The Figure object in memory has the following layout:

```c
struct Figure {
    void* allocator;            // +0x00: Memory allocator
    void* buffer;               // +0x04: Main data buffer
    uint32_t num_vertices;      // +0x08
    uint32_t vertices_offset;   // +0x0C: Offset to vertex data
    uint32_t normals_offset;    // +0x10: Offset to normals (0 if none)
    uint32_t num_polygons;      // +0x14: Total polygons (T3+T4+F3+F4)
    uint32_t num_triangles;     // +0x18: Just triangles (T3+F3)
    uint32_t polygons_offset;   // +0x1C: Offset to polygon data
    uint32_t num_segments;      // +0x20: Number of bones
    uint32_t segments_offset;   // +0x24: Offset to segment data
    uint32_t temp_buffer;       // +0x28: Working buffer for transforms
};
```

Each Segment in memory:

```c
struct Segment {
    uint16_t num_vertices;      // +0x00
    uint16_t start_vertex;      // +0x02
    Segment* parent;            // +0x04: Pointer to parent segment
    int32_t bind_matrix[12];    // +0x08: Bind pose matrix (4.12 fixed)
    uint32_t dirty_flag;        // +0x38: Transform needs update
    int32_t local_matrix[12];   // +0x3C: Current local transform
    int32_t world_matrix[12];   // +0x6C: Computed world transform
};
// Total: 156 bytes per segment
```

---

## References

- MascotCapsule Micro3D SDK documentation
- Decompiled library code analysis by AI (2025)
- Community reverse engineering efforts (minexew, woesss, nikita36078 etc?)
