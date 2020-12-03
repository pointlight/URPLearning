```c++
Shader "URPArchitecture/URP"
{
    Properties
    {
        _Int("Int", Int) = 1
        _Float("Float", Float) = 1.1
        _Range("Range", Range(0.0, 5.0)) = 1.0
        _Color("Color", Color) = (1, 1, 1, 1)
        _Vector("Vector", Vector) = (1, 1, 1, 1)
        _2D("2D", 2D) = "" {}
        _3D("3D", 3D) = "black" {}
        _Cube("Cube", Cube) = "white" {}
        
        /** Attributes */
        [PerRendererData]
        [HideInspector]
        [MainTexture]
        [MainColor]
        [ToggleOff/On] use with "Shader_Feature"
    }
    
    SubShader
    {
        Tags
        {
            /**
             * Universal Pipeline tag is required. If Universal render pipeline is not set in the graphics settings
             * this Subshader will fail. One can add a subshader below or fallback to Standard built-in to make this
             * material work with both Universal Render Pipeline and Builtin Unity Pipeline.
             * Corresponding to "Shader.globalRenderPipeline" in "UniversalRenderPipeline.cs" file.
             */
            "RenderPipeline" = "UniversalPipeline"
            
            "Queue" = "Transparent"
            "RenderType" = "Opaque"
            "DisableBatching" = "True"
            "ForceNoShadowCasting" = "True"
            "IgnoreProjector" = "True"
            "CanUseSpriteAtlas" = "False"
            "PreviewType" = "Plane"
        }
        
        Pass
        {
            Name "Pass"
            
            /** Same witch SubShader's Tags，no need "RenderPipeline" but have some special tag. */
            Tags
            {
                "LightMode"="ForwardBase"           // 定义该 pass 在流水线中的角色
                "RequireOptions"="SoftVegetation"   // 指定满足某些条件时才渲染该 pass
            }
            
            /* State Settings */
            Cull Back | Front | Off
            ZTest Less | LEqual | GEqual | Equal |NotEqual | Always
            Zwrite On | Off
            Blend SrcFactor DstFactor
            
            /** HLSL logic */
            HLSLPROGRAM
            
            // Precompiles
            #pragma vertex vert
            #pragma fragment frag
            #pragma prefer_hlslcc gles
            #pragma exclude_renderers d3d11_9x
            #pragma target 2.0
            #pragma multi_compile_instancing
            
            // Includes
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/ShaderGraphFunctions.hlsl"
            #include "Packages/com.unity.shadergraph/ShaderGraphLibrary/ShaderVariablesFunctions.hlsl"
            
            /**
             * 常量缓冲区的定义，GPU 里的一块区域，可以非常快速的和 GPU 传输数据。
             * 因为要占GPU显存，所以其容量不是很大。
             * UnityPerMaterial = 材质球的 Properties（Texture除外）共用这个 Cbuffer。
             * 需要在常量缓冲区进行声明，可以享受到 SRP 的合批功能。
             */
            CBUFFER_START(UnityPerMaterial)
            // 常量缓冲区所填充的内容
            half4 _Color;
            ...
            CBUFFER_END
            
            /**
             * 纹理与采样器的分离定义:
             * 内置管线是和纹理设置绑定的，修改不灵活。
             * 分离后可以在 Shader 内部自由组合而不受外部设置的限制。
             */
            // 纹理的定义，如果是编译到GLES2.0平台，相当于samler2D _MainTex;否则TEXTURE2D _MainTex
            TEXTURE2D(_MainTex);
            float4 _MainTex_ST;
            
            /**
             * 采样器的定义(纹理与采样器分离定义),采样器是指纹理的过滤模式与重复模式,此功能在OpenGL ES2.0上不支持，相当于没写.
             * 1.SAMPLER(sampler_textureName):sampler+纹理名称，这种定义形式是表示采用textureName这个纹理Inspector面板中的采样方式.
             * 2.SAMPLER(_filter_wrap):比如SAMPLER(point_clamp),使用自定义的采样器设置，自定义的采样器一定要同时包含过滤模式<filter>与重复模式<wrap>的设置.
             * 3.SAMPLER(_filter_wrapU_wrapV):比如SAMPLER(linear_clampU_mirrorV),可同时设置重复模式的U与V的不同值.
             * 4.filter:point/linear/triLinear
             * 5.wrap:clamp/repeat/mirror/mirrorOnce
             */
            // 为了方便操作 定义预定义
            #define smp SamplerState_Point_Repeat
            // SAMPLER(sampler_MainTex); 默认采样器
            SAMPLER(smp);
            
            /** 顶点着色器的输入 */
            struct vertex
            {
                float3 positionOS : POSITION;
                float2 uv :TEXCOORD0;
            };
            
            /** 顶点着色器的输出 */
            struct v2f
            {
                float4 positionCS : SV_POSITION;
                float2 uv :TEXCOORD0;
            };
            
            /** 顶点着色器 */
            v2f vert(vertex v)
            {
                v2f o = (Varyings)0;
                o.uv = TRANSFORM_TEX(v.uv,_MainTex);
                o.positionCS = TransformObjectToHClip(v.positionOS);
                return o;
            }

            /** 像素着色器 */
            half4 frag(v2f i) : SV_TARGET 
            {    
                // 进行纹理采样 SAMPLE_TEXTURE2D(纹理名，采样器名，uv)
                half4 mainTex = SAMPLE_TEXTURE2D(_MainTex,smp,i.uv);
                half4 c = _Color * mainTex;
                return c;
            }
            ENDHLSL
        }
        
        /** 复用其他 Unity Shader 的 Pass，Unity 内部会把所有 Pass 的名称转换为大写 */
        UsePass "MyShader/MYPASSNAME"
        
        /** 负责抓取屏幕并将结果存储在一张纹理中 */
        GrabPass {}
    }
    
    /** 当其他 SubShader 都不可用时的默认处理 */
    FallBack "name" | Off
}
```