---
title: "Procedural Grass in Ghost of Tsushima"
date: 2022-07-22 00:00:00 +0900
categories: [Game Development, Graphics]
tags: [Ghost of Tsushima, Procedural Generation, Grass, Shader, HLSL, Vulkan]
---

![Thumbnail](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/thumbnail.jpg)

게임 **Ghost of Tsushima**에서는 바람에 흔들리는 풀과 같은 식생의 움직임이 매우 중요했습니다.
바람에 흔들리는 풀을 통해 플레이어에게 길을 안내하는 시스템과 아름다운 배경을 구현하기 위해 방대한 양의 식생이 필요했으며, 이 모든 식생이 바람과 실시간으로 상호작용해야 했습니다.
이러한 요구 사항을 충족하기 위해 개발사 **Sucker Punch**는 Grass 시스템과 Wind 시스템 개발에 많은 공을 들였습니다.

![Grass Animation 1](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/01.gif)
![Grass Animation 2](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/02.gif)

## Generating a Grass Blade

Grass Blade를 모델링하기 위해 **베지어 곡선(Bezier Curve)** 방정식을 사용합니다.
여기서는 3개의 컨트롤 포인트를 사용하는 **Quadratic Bezier Curve**를 채택했습니다.
3개의 컨트롤 포인트로 곡선을 만들고, 이 곡선을 따라 두께를 부여한 뒤 15개의 Vertex를 배치하여 풀 한 줄기를 생성합니다.

![Generating Grass](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/03.png)

*   **High LOD**: 15개의 Vertex 사용
*   **Low LOD**: 7개의 Vertex 사용

![LOD](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/04.png)

*   Grass의 길이가 짧은 경우, 반으로 접어서 2개의 Blade로 만들어 사용합니다.

![Folded Blade](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/05.png)

### Quadratic Bezier Curve 제어

곡선을 형성하는 3개의 컨트롤 포인트를 임의로 **Start**, **Mid**, **End** 포인트라고 명명합니다.

![Bezier Points](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/06.png)
![Bezier Control](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/07.png)

이 컨트롤 포인트들의 위치를 변경하여 Grass의 형태를 제어할 수 있습니다.
*   **Mid Point**를 오프셋(Offset)시켜 **Bend(휘어짐)** 정도를 조절합니다.
*   **End Point**를 위아래로 이동시켜 **Tilt(기울기/높이)**를 조절합니다.

이는 마치 포토샵이나 파워포인트에서 핸들 포인트를 움직여 커브를 제어하는 것과 유사합니다. 컨트롤 포인트가 커브를 움직이는 핸들 역할을 하여 Grass Blade의 형태를 결정합니다.

### Grass Blade Parameters

*   **Position**: 식생의 위치
*   **Facing Direction**: 풀이 바라보는 방향 (바람의 방향)
*   **Tilt**: End Point의 높이
*   **Bend**: Mid Point의 Offset
*   **Length**: Blade의 길이
*   **Width**: Blade의 두께

이 파라미터들에 **Variation(변형)**을 적용하여 다양한 형태의 Grass들을 인스턴싱할 수 있습니다.

## Wind System

![Wind System](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/08.png)

바람은 방향성을 가진 움직임이므로 Curl 노이즈를 사용할 것이라 예상했으나, 실제로는 단순하게 **Perlin 노이즈**를 사용했습니다.
바람의 전체적인 방향은 글로벌하게 결정되거나 다른 제어 요소가 있을 것으로 추정되며, 바람의 강도에 따라 **Sine Wave**를 적용하여 Blade들이 **Bobbing(까딱거리는 움직임)**하도록 구현했습니다.

## Moving a Grass Blade

Grass Blade의 모션은 단순하게 **Sine Wave**를 사용합니다.
이동하는 Sine Wave에 맞춰 End Point를 위아래로 이동시킵니다. 그러면 Blade의 머리 부분이 까딱까딱하는 움직임을 연출할 수 있습니다.

![Grass Motion](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/09.png)

*   단순한 Sine Wave 움직임 적용
*   Sine Wave의 움직임에 맞춰 Tilt 값 조절
*   Blade의 고개를 까딱하게 하는 Bobbing 움직임 구현
*   각각의 Blade가 가진 Hash 값에 따른 독립적인 움직임 부여

## Data Pipeline

파이프라인에서는 주로 2개의 **Compute Shader**를 사용합니다.

1.  **Compute Shader 1**: 총 생성할 Grass 개수를 받아 인스턴싱을 위한 버퍼를 생성합니다. (Variation 적용)
2.  **Compute Shader 2**: 인스턴스 Geometry 데이터를 생성합니다.

인스턴스 생성을 위해 Grass Blade들의 위치, 바라보는 방향, 크기 등의 변수에 Variation을 적용하여 버퍼를 구성합니다.

![Pipeline](/assets/img/posts/2022-07-22-procedural-grass-in-ghost-of-tsushima/10.png)

*   **Geometry Shader**는 사용하지 않음 (Sucker Punch 방식)
*   **Compute 1**: 인스턴스 Geometry Data 생성
*   **Compute 2**: Variation을 적용하여 인스턴싱을 위한 버퍼 생성

## Results & Implementation

Sucker Punch는 Compute Shader를 사용하여 구현했지만, 저는 학습 목적으로 **Geometry Shader**를 사용하여 구현해보았습니다.
Grass Blade Geometry를 생성하고 바람에 휘날리는 움직임까지 구현했습니다.

### 구현 코드

SaschaWillems이 GitHub에서 제공하는 Vulkan 예제 중 Geometry Shader 코드를 수정하여 구현하였습니다.
[https://github.com/SaschaWillems/Vulkan](https://github.com/SaschaWillems/Vulkan)

```hlsl
struct UBO { float4x4 projection; float4x4 model; float2 viewportDim; float time; };
cbuffer ubo : register(b1) { UBO ubo; }
struct VSOutput { [[vk::location(0)]] float4 Pos : POSITION0; [[vk::location(1)]] float3 Normal : NORMAL0; };
struct GSOutput { float4 Pos : SV_POSITION; [[vk::location(0)]] float3 Color : COLOR0; };

float3 QuatraticBezierCurve(float t, float3 P0, float3 P1, float3 P2) { return (P0 * (1.0-t) * (1.0-t)) + (P1 * t * 2 * (1.0-t)) + (P2*t*t); }

void GenerateGrassBlade(inout TriangleStream<GSOutput> outStream, float3 Position, float3 FacingDir, float Length, float Tilt, uint PrimId) {
    float Width = 0.3;
    float HalfWidth = Width * 0.5;
    float3 UpDir = float3(0.0, -1.0, 0.0);
    float3 StartPoint = Position;
    float3 EndPoint = StartPoint + (FacingDir*Length) + (UpDir*Tilt);
    float3 MidPoint = (StartPoint + EndPoint) * 0.5;
    MidPoint += UpDir * 3.0;
    float Offset = (float)PrimId * 10.0;
    float Noise = sin(ubo.time * 2.0 + ((float)Offset));
    float Height = EndPoint.y;
    Height = Height * 0.7 + (Height * Noise * 0.3);
    EndPoint.y = Height;
    float3 TangentDir = normalize(cross(FacingDir, UpDir));
    float Step = 0.0;
    float Interval = 1.0 / 7.0;
    float3 Center = QuatraticBezierCurve(Step, StartPoint, MidPoint, EndPoint);
    for (int i = 0; i < 7; ++i) {
        float3 P0 = Center - (TangentDir * HalfWidth);
        float3 P1 = Center + (TangentDir * HalfWidth);
        GSOutput output = (GSOutput)0;
        output.Pos = mul(ubo.projection, mul(ubo.model, float4(P0, 1.0)));
        output.Color = float3(0.0, 1.0, 0.0);
        outStream.Append(output);
        output.Pos = mul(ubo.projection, mul(ubo.model, float4(P1, 1.0)));
        output.Color = float3(0.0, 0.0, 1.0);
        outStream.Append(output);
        Step += Interval;
        Center = QuatraticBezierCurve(Step, StartPoint, MidPoint, EndPoint);
    }
    GSOutput output = (GSOutput)0;
    output.Pos = mul(ubo.projection, mul(ubo.model, float4(Center, 1.0)));
    output.Color = float3(0.0, 1.0, 0.0);
    outStream.Append(output);
    outStream.RestartStrip();
}

float nrand(float2 uv) { return frac(sin(dot(uv, float2(12.9898, 78.233))) * 43758.5453); }

[maxvertexcount(15)]
void main(triangle VSOutput input, uint pid : SV_PrimitiveID, inout TriangleStream<GSOutput> outStream) {
    float3 center = float3(0.0, 0.0, 0.0);
    for(int i=0; i<3; i++) {
        center += input[i].Pos.xyz;
    }
    center = center / 3.0;
    float dirx = nrand(float2((float)pid, 4.0));
    float dirz = nrand(float2((float)pid, 8.0));
    float3 facingDir = float3(dirx, 0.0, dirz);
    facingDir = normalize(facingDir);
    float Length = 2.0 + nrand(float2((float)pid, 2.0)) * 2.0;
    GenerateGrassBlade(outStream, center, facingDir, Length, 2.0, pid);
}
```
