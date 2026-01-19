# Ruby Shader Graph

## Shader Overview
This shader creates a ruby gem effect using:
- **Base Material:** Deep red color with high smoothness
- **Emission:** Animated glow effect
- **Fresnel:** Edge highlighting based on view angle
- **Procedural Noise:** Internal pattern for realism

---

## Shader Pipeline Structure

### Input → Vertex Shader → Fragment Shader → Output

```hlsl
// VERTEX SHADER: Transforms mesh data
struct Attributes {
    float3 positionOS : POSITION;    // Object Space Position
    float3 normalOS : NORMAL;        // Object Space Normal
    float4 uv0 : TEXCOORD0;          // UV Coordinates
};

struct Varyings {
    float4 positionCS : SV_POSITION; // Clip Space (screen position)
    float3 positionWS;               // World Space Position
    float3 normalWS;                 // World Space Normal
    float4 texCoord0;                // UV for fragment shader
};

// FRAGMENT SHADER: Calculates final pixel color
struct SurfaceDescription {
    float3 BaseColor;    // Albedo (diffuse color)
    float3 Emission;     // Self-illumination
    float Smoothness;    // Surface glossiness (0-1)
    float Metallic;      // Metallic property (0-1)
};
```

---

### Base Color & Surface Properties

```hlsl
// Base Color: Deep Red (sRGB → Linear conversion for correct lighting)
surface.BaseColor = SRGBToLinear(float3(0.7, 0.05, 0.05));

// Surface Properties
surface.Smoothness = 0.9;   // High smoothness = mirror-like reflections
surface.Metallic = 0.0;     // Non-metallic (dielectric material)
surface.Occlusion = 1.0;    // No ambient occlusion
```

**Technical Note:** High smoothness (0.9) creates sharp specular highlights. The PBR (Physically Based Rendering) lighting model uses this to calculate realistic reflections.

---

### Emission Lighting

```hlsl
// Emission Color: Same red as base
float4 emissionColor = SRGBToLinear(float3(0.7, 0.05, 0.05));

// Final emission output (bypasses lighting calculations)
surface.Emission = emissionColor.xyz * intensity;
```

**Technical Note:** Emission is added **after** lighting calculations, making it independent of scene lights. It creates the "inner glow" effect.

---

### Time-Based Animation

```hlsl
// Nodes: Time → Sine → Normalize to 0-1 range

// Step 1: Get time value
float time = IN.TimeParameters.x;

// Step 2: Sine wave (-1 to 1)
float sineWave = sin(time);

// Step 3: Normalize to 0-1 range
float pulsate = (sineWave + 1.0) * 0.5;

// Step 4: Apply to emission
surface.Emission *= pulsate;
```

**Technical Note:** `sin(time)` oscillates between -1 and 1. We normalize it to 0-1 to create smooth pulsing without negative values.

**Math:**
```
time = 0s  → sin(0) = 0    → pulsate = 0.5
time = π/2 → sin(π/2) = 1  → pulsate = 1.0 (max brightness)
time = π   → sin(π) = 0    → pulsate = 0.5
```

---

### View-Dependent Edge Glow

```hlsl
// Fresnel function: Highlights edges based on viewing angle
void Unity_FresnelEffect_float(float3 Normal, float3 ViewDir, float Power, out float Out)
{
    // N·V dot product (1 = face-on, 0 = edge-on)
    float NdotV = saturate(dot(normalize(Normal), ViewDir));
    
    // Invert and apply power for edge emphasis
    Out = pow((1.0 - NdotV), Power);
}

// Usage in shader:
float fresnel = FresnelEffect(worldNormal, viewDirection, 3.0);
surface.Emission += fresnel * emissionColor;
```

**Technical Note:** 
- **N·V = 1:** Viewing perpendicular → Fresnel = 0 (no glow)
- **N·V = 0:** Viewing along edge → Fresnel = 1 (max glow)
- **Power = 3:** Controls falloff sharpness (higher = sharper edges)

**Visual:**
```
     Camera
       ↓
    ●━━━━━●  ← Edge (N·V ≈ 0) → Bright glow
    │     │
    │ Ruby│  ← Center (N·V ≈ 1) → No glow
    │     │
    ●━━━━━●  ← Edge (N·V ≈ 0) → Bright glow
```

---

### Gradient Noise Generation

```hlsl
// Step 1: Animate UVs over time
float2 scrollSpeed = float2(0.1, 0);
float2 animatedUV = IN.uv0.xy * 3.0 + (time * scrollSpeed);

// Step 2: Generate Perlin-like noise
float noise = GradientNoise(animatedUV, scale: 6.0);

// Step 3: Multiply with base emission
surface.Emission = baseColor * noise + fresnelGlow;
```

**Gradient Noise Algorithm:**
```hlsl
void Unity_GradientNoise_Deterministic_float(float2 UV, float Scale, out float Out)
{
    float2 p = UV * Scale;
    float2 gridCell = floor(p);       // Integer part
    float2 cellPos = frac(p);         // Fractional part
    
    // Get gradient at 4 grid corners
    float d00 = dot(randomGradient(gridCell), cellPos);
    float d01 = dot(randomGradient(gridCell + float2(0,1)), cellPos - float2(0,1));
    float d10 = dot(randomGradient(gridCell + float2(1,0)), cellPos - float2(1,0));
    float d11 = dot(randomGradient(gridCell + float2(1,1)), cellPos - float2(1,1));
    
    // Smooth interpolation (smoothstep)
    float2 smooth = cellPos * cellPos * (3.0 - 2.0 * cellPos);
    
    // Bilinear interpolation
    Out = lerp(lerp(d00, d01, smooth.y), 
               lerp(d10, d11, smooth.y), 
               smooth.x);
}
```

**Technical Note:** Gradient noise creates continuous, organic patterns without texture files. The `scale: 6.0` parameter controls pattern frequency (higher = more detail).

---

## 📊 Complete Shader Graph Node Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                          SHADER GRAPH STRUCTURE                      │
└─────────────────────────────────────────────────────────────────────┘

TIME ──┬──→ [SINE] ──→ [ADD 1.0] ──→ [MULTIPLY 0.5] ──→ PULSATE (0-1)
       │                                                        │
       └──→ [MULTIPLY SPEED] ──┐                              │
                                │                               │
UV ──→ [TILING 3×3] ──────────→ [MULTIPLY] ──→ [GRADIENT      │
                                                  NOISE] ──┐   │
                                                           │   │
BASE COLOR (0.7, 0.05, 0.05) ─────────────────────────────┤   │
                                                           │   │
                                                      [MULTIPLY]│
                                                           │   │
                                                           ├───┼──→ [ADD] ──→ EMISSION
                                                           │   │
WORLD NORMAL ─────┐                                       │   │
                  │                                        │   │
VIEW DIRECTION ───┼──→ [FRESNEL] ──→ [MULTIPLY] ──────────┘   │
                  │      Power=3       Color                   │
                  │                                             │
                  └──→ N·V DOT PRODUCT                         │
                                                                │
                                            [MULTIPLY] ←────────┘
                                                ↓
                                            PULSATE
```

---

## Fragment Shader Summary

```hlsl
SurfaceDescription SurfaceDescriptionFunction(SurfaceDescriptionInputs IN)
{
    SurfaceDescription surface = (SurfaceDescription)0;
    
    // === Base Material ===
    surface.BaseColor = SRGBToLinear(float3(0.7, 0.05, 0.05));
    surface.Smoothness = 0.9;
    surface.Metallic = 0.0;
    
    // === Time Animation ===
    float pulsate = (sin(IN.TimeParameters.x) + 1.0) * 0.5;
    
    // === Procedural Noise ===
    float2 animUV = IN.uv0.xy * 3.0 + IN.TimeParameters.x * float2(0.1, 0);
    float noise = GradientNoise(animUV, 6.0);
    float4 noiseEmission = baseColor * noise;
    
    // === Fresnel Effect ===
    float fresnel = FresnelEffect(IN.WorldSpaceNormal, IN.WorldSpaceViewDirection, 3.0);
    float4 fresnelEmission = fresnel * baseColor;
    
    // === Final Emission ===
    surface.Emission = (noiseEmission + fresnelEmission * pulsate).xyz;
    
    return surface;
}
```

---

## Key Technical Concepts

### 1. **Coordinate Spaces**
- **Object Space:** Mesh vertices as modeled
- **World Space:** Scene coordinates
- **Clip Space:** Screen projection coordinates

### 2. **Dot Product (N·V)**
```
N·V = |N| × |V| × cos(θ)
    = cos(angle between normal and view)
    = 1 when face-on
    = 0 when edge-on
```

### 3. **Perlin Noise Properties**
- Continuous (smooth transitions)
- Deterministic (same input = same output)
- Band-limited (no sharp edges)
- Tileable (when using periodic functions)

---

## Quick Reference

| Property | Value | Effect |
|----------|-------|--------|
| Base Color | (0.7, 0.05, 0.05) | Deep red |
| Smoothness | 0.9 | Mirror-like reflections |
| Fresnel Power | 3.0 | Sharp edge glow |
| Noise Scale | 6.0 | Medium detail pattern |
| UV Tiling | 3×3 | Pattern repeats 3 times |
| Animation Speed | 0.1 | Slow scroll |
| Pulse Frequency | 1 cycle/2π seconds | ~1 cycle per 6.28s |

---
