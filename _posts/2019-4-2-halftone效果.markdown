---
layout:     post
title:      "halftone in unity"
subtitle:   ""
date:       2019-4-2
author:     "CodeSuperHero"
header-img: "img/2b.png"
tags:
    - Unity
    - shader
---



shader代码

最近看到了halftone效果的文章，觉得还不错，尝试搬到Unity里面来实现下，halftone原理就不说了，有兴趣了解请Google一下。这里直接贴shader代码和效果图吧。

```
Shader "Unlit/HalfTone"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Angle ("angle", float)  = 1.57                         //取滤镜点的旋转值
        _Scale ("scale", float)  = 1                            //滤镜点坐标的缩放值
        _Center ("center", vector) = (0.5, 0.5 , 0, 0)          //uv中心坐标值
        _UVSize ("uv_size", vector) = (10000, 5630 , 0, 0)      //uv坐标缩放值
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" }
        LOD 100
        Blend SrcAlpha OneMinusSrcAlpha

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float _Angle;
            float _Scale ;
            float4 _Center;
            float4 _UVSize;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            float3 blendColor(float3 base, float3 blend) {
                return float3(max(base.x, blend.x), max(base.y, blend.y), max(base.z, blend.z));
            }

            float3 blendColor(float3 base, float3 blend, float opacity) {
                return (blendColor(base, blend) * opacity + base * (1.0 - opacity));
            }

            //滤镜采样
            float pattern(float2 uv) {
                float s = sin(_Angle);
                float c = cos(_Angle);
                float2 tex = uv * _UVSize - _Center.xy;
                float2 p = float2( c * tex.x - s * tex.y, s * tex.x + c * tex.y ) * _Scale;
                return ( sin( p.x ) * sin( p.y ) ) * 4.0;
            }

            float4 frag (v2f i) : SV_Target
            {
                float4 col = tex2D(_MainTex, i.uv);
                float average = (col.r + col.g + col.b) * 0.333;                    //用灰度图做滤镜图
                float t = average * 10.0 - 5.0;                                     //加大色差
                t = t + pattern(i.uv);                                              //混合滤镜图
                float3 halftone = float3(t, t, t);
                float3 rgbCol = col.xyz + float3(0.02, 0.02, 0.02) * halftone.xyz;  //滤镜图和原图增亮叠加
                col = fixed4(blendColor(rgbCol, halftone, 0.05), col.a);
                return col;
            }
            ENDCG
        }
    }
}
```

效果展示
原图
<img src="https://raw.githubusercontent.com/CodeSuperHero/CodeSuperHero.github.io/master/img/shader/halftone.png" width = "400" height = "200" alt="图片名称"/>

滤镜图
<img src="https://raw.githubusercontent.com/CodeSuperHero/CodeSuperHero.github.io/master/img/shader/halftone_filter.png" width = "400" height = "200" alt="图片名称"/>

最终效果图
<img src="https://raw.githubusercontent.com/CodeSuperHero/CodeSuperHero.github.io/master/img/shader/halftone.png" width = "400" height = "200" alt="图片名称"/>