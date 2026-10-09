Now complete!

Shader 1: Water
Water is a shader graph that is reflective and rises and falls

Vetex Shader Part of graph
<img width="932" height="507" alt="WaterVertex" src="https://github.com/user-attachments/assets/bcbcf611-809c-40b2-9ba7-b22c3f9d604f" />
Fragment Part of graph
<img width="1052" height="492" alt="WaterFragment" src="https://github.com/user-attachments/assets/e031b8f4-c1a3-4d52-9597-aafe57366ce1" />

Shader 2: Leaves

Vertex shader that moves with time on all 3 axis

Leaf Vertex
<img width="937" height="522" alt="LeafVertex" src="https://github.com/user-attachments/assets/bdf0b7be-f937-4911-b635-9bc528d1659c" />

Leaf Fragment
<img width="1012" height="417" alt="LeafFragment" src="https://github.com/user-attachments/assets/2d51dccc-dfd9-4fde-98ba-18519db022de" />


Shader 3: Diffuse lighting with specular

I did just retype the code from the slides out by hand:

Shader "Custom/Ground"
{
    Properties
    {
        _BaseColor("Base Color", Color) = (1, 1, 1, 1)
        _MainTex("Base Texture", 2D) = "white" {}
        _SpecColor ("Specular Color", Color) = (1,1,1,1)
        _Shininess ("Shininess", Range(0.1,100)) = 16 
    }

    SubShader
    {
        Tags { "RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline" }

        Pass
        {
            HLSLPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION; //Object Space Position
                float3 normalOS : NORMAL; //Object Space Normal
                float2 uv : TEXCOORD0; // Texture UV
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION; //Honogenous clip-space position
                float3 normalWS : TEXCOORD1; //World Space Normal
                float3 viewDirWS : TEXCOORD2; //World Space View Direction
                float2 uv : TEXCOORD0; //UV for texturing
            };

            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            CBUFFER_START(UnityPerMaterial)
                half4 _BaseColor;
                float4 _SpecColor;
                float _Shininess;
            CBUFFER_END

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                //Transdorm the object space position to homogeneous clip space
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                //Transdorm the object space normal to world space
                OUT.normalWS = normalize(TransformObjectToWorldNormal(IN.normalOS));
                //Compute view direction in world space
                float3 worldPosWS = TransformObjectToWorld(IN.positionOS.xyz);
                OUT.viewDirWS = normalize(GetCameraPositionWS() - worldPosWS);
                //Pass the UV to the fragment shader
                OUT.uv = IN.uv;
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                //Sample the base texture
                half4 texColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv);

                //Fetch the main light in URP
                Light mainLight = GetMainLight();
                half3 lightDir = normalize(mainLight.direction);

                //Normalize the world space normal
                half3 normalWS = normalize(IN.normalWS);

                //Calculate Lambertian diffuse lighting (NdotL)
                half NdotL = saturate(dot(normalWS, lightDir));

                //Calculate ambient lighting using sperical harmonics (SH)
                half3 ambientSH = SampleSH(normalWS);

                //Conbine the base color and texture with the diffuse light
                half3 diffuse = texColor.rgb * _BaseColor.rgb * NdotL;

                //Calculate the reflection direction for specular
                half3 reflectDir = reflect(-lightDir,normalWS);

                //Calculate specular contribution using Blinn-Phong model
                half3 viewDir = normalize(IN.viewDirWS);
                half specFactor = pow(saturate(dot(reflectDir, viewDir)), _Shininess);
                half3 specular = _SpecColor.rgb * specFactor;

                //Combine diffuse lighting, ambient lighting, and specular highlights
                half3 finalColor = diffuse + ambientSH * texColor.rgb * _BaseColor.rgb + specular;

                //Return the final color
                return half4(finalColor,1.0);
            }
            ENDHLSL
        }
    }
}
