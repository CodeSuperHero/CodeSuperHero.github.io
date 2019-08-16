# Unity PBR 笔记

## Unity PBR Shader

unity 提供的 pbs 共四个，其中两个给模型使用，另外两个给粒子使用
我们主要研究给模型使用的shader源文件，以及它关联的七个CG头文件

1. shader源文件
- Standard                      //标准版                 
- Standard (Specular Setup)     //高光版
- Particles/Standard Surface
- Particles/Standard Unlit

2. CG头文件
- UnityStandardBRDF.cginc 处理BRDF材质属性相关的函数与宏
- UnityStandardConfig.cginc 配置相关的代码（其实里面就几个宏）
- UnityStandardCore.cginc 主要代码（如顶点着色函数、片段着色函数等相关函数）
- UnityStandardInput.cginc 输入结构相关的工具函数与宏
- UnityStandardMeta.cginc meta通道中会用到的工具函数与宏
- UnityStandardShadow.cginc 阴影贴图采样相关的工具函数与宏
- UnityStandardUtils.cginc 共用的一些工具函数

3. StandardShaderGUI.cs Standard shader 材质球的 Inspector 类
    
## Standard 分析
standard shader 的基础结构

``` standard 代码
Shader "Standard"
{
    Properties
    {
        _Color("Color", Color) = (1,1,1,1)
        _MainTex("Albedo", 2D) = "white" {}
        _Cutoff("Alpha Cutoff", Range(0.0, 1.0)) = 0.5
        _Glossiness("Smoothness", Range(0.0, 1.0)) = 0.5
        _GlossMapScale("Smoothness Scale", Range(0.0, 1.0)) = 1.0
        [Enum(Metallic Alpha,0,Albedo Alpha,1)] _SmoothnessTextureChannel ("Smoothness texture channel", Float) = 0
        [Gamma] _Metallic("Metallic", Range(0.0, 1.0)) = 0.0
        _MetallicGlossMap("Metallic", 2D) = "white" {}
        [ToggleOff] _SpecularHighlights("Specular Highlights", Float) = 1.0
        [ToggleOff] _GlossyReflections("Glossy Reflections", Float) = 1.0
        _BumpScale("Scale", Float) = 1.0
        _BumpMap("Normal Map", 2D) = "bump" {}
        _Parallax ("Height Scale", Range (0.005, 0.08)) = 0.02
        _ParallaxMap ("Height Map", 2D) = "black" {}
        _OcclusionStrength("Strength", Range(0.0, 1.0)) = 1.0
        _OcclusionMap("Occlusion", 2D) = "white" {}
        _EmissionColor("Color", Color) = (0,0,0)
        _EmissionMap("Emission", 2D) = "white" {}
        _DetailMask("Detail Mask", 2D) = "white" {}
        _DetailAlbedoMap("Detail Albedo x2", 2D) = "grey" {}
        _DetailNormalMapScale("Scale", Float) = 1.0
        _DetailNormalMap("Normal Map", 2D) = "bump" {}

        [Enum(UV0,0,UV1,1)] _UVSec ("UV Set for secondary textures", Float) = 0


        // Blending state
        [HideInInspector] _Mode ("__mode", Float) = 0.0
        [HideInInspector] _SrcBlend ("__src", Float) = 1.0
        [HideInInspector] _DstBlend ("__dst", Float) = 0.0
        [HideInInspector] _ZWrite ("__zw", Float) = 1.0
    }

    CGINCLUDE
    #define UNITY_SETUP_BRDF_INPUT MetallicSetup //声明 BRDF 为金属度模型
    ENDCG

    SubShader   // shadermodel 3.0
    {
		Tags { "RenderType"="Opaque" "PerformanceChecks"="False" } // 默认 不透明渲染
		LOD 300 // lod level 300

        Pass    // forward基础pass (directional light, emission, lightmaps, ...)
        {
            Tags { "LightMode" = "ForwardBase" }    //只计算第一个平行光
            Blend [_SrcBlend] [_DstBlend]           //源乘以目标混合，StandardShaderGUI 设置
            ZWrite [_ZWrite]                        //设置深入写入模式，StandardShaderGUI 设置
            CGPROGRAM
            #pragma target 3.0
            
            // 预编译指令，StandardShaderGUI 会根据参数来设置这些指令，增加或者裁减目标shader代码

            #pragma shader_feature _NORMALMAP // 法线贴图
            #pragma shader_feature _ _ALPHATEST_ON _ALPHABLEND_ON _ALPHAPREMULTIPLY_ON // 透明测试,透明混合, 预乘Alpha
            #pragma shader_feature _EMISSION // 自发光
            #pragma shader_feature _METALLICGLOSSMAP //金属光泽度贴图
            #pragma shader_feature ___ _DETAIL_MULX2 //使用 _DetailAlbedoMap || _DetailNormalMap
            #pragma shader_feature _ _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A //smoothness参数来自于albedo贴图的alpha通道
            #pragma shader_feature _ _SPECULARHIGHLIGHTS_OFF //_SpecularHighlights 参数
            #pragma shader_feature _ _GLOSSYREFLECTIONS_OFF //_GlossyReflections 参数
            #pragma shader_feature _PARALLAXMAP //视差贴图，可以配合法线贴图表现更好的凹凸效果

            #pragma multi_compile_fwdbase //编译不同变体来处理不同光照贴图类型，阴影开关等
            #pragma multi_compile_fog // 编译变种来处理不同的雾效
            #pragma multi_compile_instancing

            #pragma vertex vertBase
            #pragma fragment fragBase
            #include "UnityStandardCoreForward.cginc"

            ENDCG
        }
        Pass    // Additive forward pass (one light per pass)
        {

        }
        Pass    // Shadow rendering pass
        {

        }
        Pass    // Deferred pass
        {

        }
        Pass    // Extracts information for lightmapping, GI (emission, albedo, ...)
        {       // This pass it not used during regular rendering.
                // 这个 pass 主要是用来计算 albedo,emission 给enlighten使用
        }     
    }

    SubShader   // shadermodel 2.0
    {

    }
}
```