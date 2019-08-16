# pbr

## 光照探头 lightprobe

## standard shader 代码结构
- standard
    1. subshder opaque lod300
       1. pass "FORWARD"
          - ForwardBase 
          - Blend [_SrcBlend] [_DstBlend]
          - zwrite
          - vertBase
          - fragBase
          - "UnityStandardCoreForward.cginc"
       2. pass "FORWARD_DELTA"
            - ForwardAdd
            - Blend [_SrcBlend] One
            - Fog { Color (0,0,0,0) }
            - ZWrite Off
            - ZTest LEqual
            - vertAdd
            - fragAdd
            - "UnityStandardCoreForward.cginc"
       3. pass "ShadowCaster"
            - ShadowCaster
            - ZWrite On
            - ZTest LEqual
            - vertShadowCaster
            - fragShadowCaster
            - "UnityStandardShadow.cginc"
       4. pass "DEFERRED"
            - Deferred
            - vertDeferred
            - fragDeferred
            - "UnityStandardCore.cginc"
       5. pass "META"
            - "Meta"
            - Cull Off
            - vert_meta
            - frag_meta
            - "UnityStandardMeta.cginc"
    2. subshader opaque load150
       1. pass "FORWARD"
            - ForwardBase
            - Blend [_SrcBlend] [_DstBlend]
            - ZWrite [_ZWrite]
            - skip_variants SHADOWS_SOFT DIRLIGHTMAP_COMBINED
            - vertBase
            - fragBase
            - "UnityStandardCoreForward.cginc"
       2. pass "FORWARD_DELTA"
            - ForwardAdd
            - Blend [_SrcBlend] One
            - Fog { Color (0,0,0,0) }
            - ZWrite Off
            - ZTest LEqual
            - skip_variants SHADOWS_SOFT
            - vertAdd
            - fragAdd
            - "UnityStandardCoreForward.cginc"
       3. pass "ShadowCaster"
            - ShadowCaster
            - ZWrite On
            - ZTest LEqual
            - vertShadowCaster
            - fragShadowCaster
            - "UnityStandardShadow.cginc"
       4. pass "META"
            - ShadowCaster
            - Meta
            - Cull Off
            - vert_meta
            - frag_meta
            - "UnityStandardMeta.cginc"
    3. fallback vertexlit

### 依赖的结构体

- UnityStandardInput.cginc
```
struct VertexInput
    {
        float4 vertex   : POSITION;
        half3 normal    : NORMAL;
        float2 uv0      : TEXCOORD0;
        float2 uv1      : TEXCOORD1;
        #if defined(DYNAMICLIGHTMAP_ON) || defined(UNITY_PASS_META)
            float2 uv2      : TEXCOORD2;
        #endif
        #ifdef _TANGENT_TO_WORLD
            half4 tangent   : TANGENT;
        #endif
        UNITY_VERTEX_INPUT_INSTANCE_ID
    };
```

- UnityStandardParticleEditor.cginc
```
struct VertexInput
{
    float4 vertex   : POSITION;
    float3 normal   : NORMAL;
    fixed4 color    : COLOR;
    #if defined(_FLIPBOOK_BLENDING) && !defined(UNITY_PARTICLE_INSTANCING_ENABLED)
        float4 texcoords : TEXCOORD0;
        float texcoordBlend : TEXCOORD1;
    #else
        float2 texcoords : TEXCOORD0;
    #endif
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

struct VertexOutput
{
    float2 texcoord : TEXCOORD0;
    #ifdef _FLIPBOOK_BLENDING
        float3 texcoord2AndBlend : TEXCOORD1;
    #endif
    fixed4 color : TEXCOORD2;
};
```

- UnityStandardParticleShadow.cginc
```
struct VertexInput
{
    float4 vertex   : POSITION;
    float3 normal   : NORMAL;
    fixed4 color    : COLOR;
    #if defined(_FLIPBOOK_BLENDING) && !defined(UNITY_PARTICLE_INSTANCING_ENABLED)
        float4 texcoords : TEXCOORD0;
        float texcoordBlend : TEXCOORD1;
    #else
        float2 texcoords : TEXCOORD0;
    #endif
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

- UnityStandardShadow.cginc
```
struct VertexInput
{
    float4 vertex   : POSITION;
    float3 normal   : NORMAL;
    float2 uv0      : TEXCOORD0;
    #if defined(UNITY_STANDARD_USE_SHADOW_UVS) && defined(_PARALLAXMAP)
        half4 tangent   : TANGENT;
    #endif
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

- UnityStandardCoreForwardSimple.cginc
``` VertexOutputBaseSimple
struct VertexOutputBaseSimple
    {
        UNITY_POSITION(pos);
        float4 tex                          : TEXCOORD0;
        half4 eyeVec                        : TEXCOORD1; // w: grazingTerm
        half4 ambientOrLightmapUV           : TEXCOORD2; // SH or Lightmap UV
        SHADOW_COORDS(3)
        UNITY_FOG_COORDS_PACKED(4, half4) // x: fogCoord, yzw: reflectVec
        half4 normalWorld                   : TEXCOORD5; // w: fresnelTerm
        #ifdef _NORMALMAP
            half3 tangentSpaceLightDir      : TEXCOORD6;
            #if SPECULAR_HIGHLIGHTS
                half3 tangentSpaceEyeVec    : TEXCOORD7;
            #endif
        #endif
        #if UNITY_REQUIRE_FRAG_WORLDPOS
            float3 posWorld                 : TEXCOORD8;
        #endif
        UNITY_VERTEX_OUTPUT_STEREO
    };
```

- UnityStandardCore.cginc
``` FragmentCommonData
struct FragmentCommonData
{
    half3 diffColor, specColor;
    // Note: smoothness & oneMinusReflectivity for optimization purposes, mostly for DX9SM2.0 level.
    // Most of the math is being done on these (1-x) values, and that saves a few preciousALU slots.
    half oneMinusReflectivity, smoothness;
    float3 normalWorld;
    float3 eyeVec;
    half alpha;
    float3 posWorld;
    #if UNITY_STANDARD_SIMPLE
        half3 reflUVW;
    #endif
    #if UNITY_STANDARD_SIMPLE
        half3 tangentSpaceNormal;
    #endif
};
```

### 调用栈
- VertexOutputBaseSimple verBase(VertexInput v)  //vertexinput 使用的"UnityStandardInput.cginc"
  - VertexOutputBaseSimple vertForwardBaseSimple(VertexInput v) 
- half4 fragBase (VertexOutputBaseSimple i)
  - half4 fragForwardBaseSimpleInternal (VertexOutputBaseSimple i)
    - FragmentCommonData FragmentSetupSimple(VertexOutputBaseSimple i)

``` vertForwardBaseSimple
VertexOutputBaseSimple vertForwardBaseSimple (VertexInput v)
{
    // GPUInstance https://docs.unity3d.com/Manual/SinglePassInstancing.html
    UNITY_SETUP_INSTANCE_ID(v); 
    VertexOutputBaseSimple o;
    UNITY_INITIALIZE_OUTPUT(VertexOutputBaseSimple, o);
    UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
    float4 posWorld = mul(unity_ObjectToWorld, v.vertex);
    o.pos = UnityObjectToClipPos(v.vertex);
    o.tex = TexCoords(v);
    half3 eyeVec = normalize(posWorld.xyz - _WorldSpaceCameraPos);
    half3 normalWorld = UnityObjectToWorldNormal(v.normal);
    o.normalWorld.xyz = normalWorld;
    o.eyeVec.xyz = eyeVec;
    #ifdef _NORMALMAP
        half3 tangentSpaceEyeVec;
        TangentSpaceLightingInput(normalWorld, v.tangent, _WorldSpaceLightPos0.xyz, eyeVec,o.tangentSpaceLightDir, tangentSpaceEyeVec);
        #if SPECULAR_HIGHLIGHTS
            o.tangentSpaceEyeVec = tangentSpaceEyeVec;
        #endif
    #endif
    //We need this for shadow receiving
    TRANSFER_SHADOW(o);
    o.ambientOrLightmapUV = VertexGIForward(v, posWorld, normalWorld);
    o.fogCoord.yzw = reflect(eyeVec, normalWorld);
    o.normalWorld.w = Pow4(1 - saturate(dot(normalWorld, -eyeVec))); // fresnel term
    #if !GLOSSMAP
        o.eyeVec.w = saturate(_Glossiness + UNIFORM_REFLECTIVITY()); // grazing term
    #endif
    UNITY_TRANSFER_FOG(o, o.pos);
    return o;
    }
```

``` TexCoords
float4 TexCoords(VertexInput v)
{
    float4 texcoord;
    texcoord.xy = TRANSFORM_TEX(v.uv0, _MainTex); // Always source from uv0
    texcoord.zw = TRANSFORM_TEX(((_UVSec == 0) ? v.uv0 : v.uv1), _DetailAlbedoMap);
    return texcoord;
}
```

``` TangentSpaceLightingInput
void TangentSpaceLightingInput(half3 normalWorld, half4 vTangent, half3 lightDirWorld, half3 eyeVecWorld, out half3 tangentSpaceLightDir, out half3 tangentSpaceEyeVec)
{
    half3 tangentWorld = UnityObjectToWorldDir(vTangent.xyz);
    half sign = half(vTangent.w) * half(unity_WorldTransformParams.w);
    half3 binormalWorld = cross(normalWorld, tangentWorld) * sign;
    tangentSpaceLightDir = TransformToTangentSpace(tangentWorld, binormalWorld, normalWorldlightDirWorld);
    #if SPECULAR_HIGHLIGHTS
        tangentSpaceEyeVec = normalize(TransformToTangentSpace(tangentWorld, binormalWorldnormalWorld, eyeVecWorld));
    #else
        tangentSpaceEyeVec = 0;
    #endif
}
```

```
inline half4 VertexGIForward(VertexInput v, float3 posWorld, half3 normalWorld)
{
    half4 ambientOrLightmapUV = 0;
    // Static lightmaps
    #ifdef LIGHTMAP_ON
        ambientOrLightmapUV.xy = v.uv1.xy * unity_LightmapST.xy + unity_LightmapST.zw;
        ambientOrLightmapUV.zw = 0;
        // Sample light probe for Dynamic objects only (no static or dynamic lightmaps)
    #elif UNITY_SHOULD_SAMPLE_SH
        #ifdef VERTEXLIGHT_ON
            // Approximated illumination from non-important point lights
            ambientOrLightmapUV.rgb = Shade4PointLights (
            unity_4LightPosX0, unity_4LightPosY0, unity_4LightPosZ0,
            unity_LightColor[0].rgb, unity_LightColor[1].rgb, unity_LightColor[2].rgb,unity_LightColor[3].rgb,
            unity_4LightAtten0, posWorld, normalWorld);
        #endif
        ambientOrLightmapUV.rgb = ShadeSHPerVertex (normalWorld, ambientOrLightmapUV.rgb);
    #endif
    #ifdef DYNAMICLIGHTMAP_ON
        ambientOrLightmapUV.zw = v.uv2.xy * unity_DynamicLightmapST.xy +unity_DynamicLightmapST.zw;
    #endif
    return ambientOrLightmapUV;
}
```

```
FragmentCommonData FragmentSetupSimple(VertexOutputBaseSimple i)
{
    half alpha = Alpha(i.tex.xy);
    #if defined(_ALPHATEST_ON)
        clip (alpha - _Cutoff);
    #endif
    FragmentCommonData s = UNITY_SETUP_BRDF_INPUT (i.tex);
    // NOTE: shader relies on pre-multiply alpha-blend (_SrcBlend = One, _DstBlend =OneMinusSrcAlpha)
    s.diffColor = PreMultiplyAlpha (s.diffColor, alpha, s.oneMinusReflectivity, /*out*/ s.alpha);
    s.normalWorld = i.normalWorld.xyz;
    s.eyeVec = i.eyeVec.xyz;
    s.posWorld = IN_WORLDPOS(i);
    s.reflUVW = i.fogCoord.yzw;
    #ifdef _NORMALMAP
        s.tangentSpaceNormal =  NormalInTangentSpace(i.tex);
    #else
        s.tangentSpaceNormal =  0;
    #endif
        return s;
}
```

```
inline UnityGI FragmentGI (FragmentCommonData s, half occlusion, half4 i_ambientOrLightmapUV, half atten, UnityLight light, bool reflections)
{
    UnityGIInput d;
    d.light = light;
    d.worldPos = s.posWorld;
    d.worldViewDir = -s.eyeVec;
    d.atten = atten;
    #if defined(LIGHTMAP_ON) || defined(DYNAMICLIGHTMAP_ON)
        d.ambient = 0;
        d.lightmapUV = i_ambientOrLightmapUV;
    #else
        d.ambient = i_ambientOrLightmapUV.rgb;
        d.lightmapUV = 0;
    #endif
    d.probeHDR[0] = unity_SpecCube0_HDR;
    d.probeHDR[1] = unity_SpecCube1_HDR;
    #if defined(UNITY_SPECCUBE_BLENDING) || defined(UNITY_SPECCUBE_BOX_PROJECTION)
        d.boxMin[0] = unity_SpecCube0_BoxMin; // .w holds lerp value for blending
    #endif
    #ifdef UNITY_SPECCUBE_BOX_PROJECTION
        d.boxMax[0] = unity_SpecCube0_BoxMax;
        d.probePosition[0] = unity_SpecCube0_ProbePosition;
        d.boxMax[1] = unity_SpecCube1_BoxMax;
        d.boxMin[1] = unity_SpecCube1_BoxMin;
        d.probePosition[1] = unity_SpecCube1_ProbePosition;
    #endif
    if(reflections)
    {
        Unity_GlossyEnvironmentData g = UnityGlossyEnvironmentSetup(s.smoothness, -s.eyeVec,s.normalWorld, s.specColor);
        // Replace the reflUVW if it has been compute in Vertex shader. Note: the compiler willoptimize the calcul in UnityGlossyEnvironmentSetup itself
        #if UNITY_STANDARD_SIMPLE
            g.reflUVW = s.reflUVW;
        #endif
        return UnityGlobalIllumination (d, occlusion, s.normalWorld, g);
    }
    else
    {
        return UnityGlobalIllumination (d, occlusion, s.normalWorld);
    }
}
```