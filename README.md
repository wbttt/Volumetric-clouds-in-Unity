# Volumetric Clouds in Unity<br/>



## Texture

Use three different textures to render the volumetric clouds,


1. Weather texture :

   <img src="https://i.imgur.com/AbQXfYe.png" width="200" height="200" />



2. Perlin-Worley texture ( base cloud shape ) : 128^3 resolution

   <img src="https://i.imgur.com/rSJVdLP.png" width="200" height="200" />

   

3. Worley noise texture ( detail base cloud shape ) : 32^3 resolution

   <img src="https://i.imgur.com/1bE5c57.png" width="200" height="200" /><br/><br/>

   
## Cloud density model

**Weather system:**

This little map represents the weather settings that drive the clouds over our section of world map. The Red is coverage, Green is precipitation and blue is cloud type.

<img src="https://i.imgur.com/tZCo0zx.png" width="200" height="200" /><br/>

```
float3 WeatherData(float3 pos)
{
	float2 unit = pos.xz * _WeatherTexUVScale / 30000.0;

	float2 uv = unit * 0.5 + 0.5;

	return tex2Dlod(_WeatherTex, float4(uv, 0.0, 0.0));
}
```
<br/>



**Cloud type :** 

Based on weather system map , we could know the weatherData.b = cloud type
<br/>


**Height gradient:**

height gradient  change the noise signal over altitude.

```
float4 GetHeightGradient(float cloudType)
{
	const float4 CloudGradient1 = float4(0.0, 0.07, 0.08, 0.15);
	const float4 CloudGradient2 = float4(0.0, 0.2, 0.42, 0.6);
	const float4 CloudGradient3 = float4(0.0, 0.08, 0.75, 0.98);

	float a = 1.0 - saturate(cloudType * 2.0);
	float b = 1.0 - abs(cloudType - 0.5) * 2.0;
	float c = saturate(cloudType - 0.5) * 2.0;

	return CloudGradient1 * a + CloudGradient2 * b + CloudGradient3 * c;
}
```
<br/>


If we were to describe the probability of change in density over height for a cloud we might use a remapping function.

```
float Remap(float org_val, float org_min, float org_max, float new_min, float new_max)
{
	return new_min + saturate(((org_val - org_min) / (org_max - org_min))*(new_max - new_min));
}
```
<br/>


**Basic cloud shape:**

We build a basic cloud shape by sampling our Perlin-Worley texture and multiplying it by our height signal and then multiply the result by the coverage.
<br/>


**Detail cloud shape:**

Erode the base cloud shape by subtracting the Worley noise texture at the edges of the cloud. If you invert the Worley noise at the base of the clouds you get some nice whispy shapes.
<br/>


**Wind effect :**

```
pos += wind_direction * time_offset;
pos += height_fraction * wind_direction * 500.0;
```
<br/>




## Cloud Lighting model



**Ambient light:**

```
float3 AmbientLighting(float heightFrac)
{
	return lerp(_CloudBaseColor.rgb, _CloudTopColor.rgb, heightFrac);
}
```
<br/>



**Beer's Law :**



<img src="https://i.imgur.com/PBokUuO.png" width="500" height="350" /><br/>

Beer’s law states that we can determine the amount of light reaching a point based on the optical thickness of the medium that it travels through.

```
float BeerLaw(float opticalDepth)
{
	return exp(- Parameter * opticalDepth);
}
```
<br/>



**Henyey-Greenstein model :** 

In clouds, there is a higher probability of light scattering forward. This is called Anisotropic scattering. This model is used to reproduce Anisotropy in cloud lighting.

```
// scatterAngle = norm_rayDireciton dot light_direction
float HenyeyGreenStein(float ScatterAngle, float g)
{
	return ((1.0 - g * g) / pow((1.0 + g * g - 2.0 * g * ScatterAngle), 3.0 / 2.0)) / (4.0 * 3.14159);
}
```
<br/>


Each time we sample light energy, we multiply it by The Henyey-Greenstein phase function plus Beer's Law.
$$
Light Energy = e^{-d}*HG
$$

If you want to have a better volumetric clouds, You could add the powder sugar effect which produces the dark edges facing the light.
$$
E = 1 - e^{-2d}
$$
<br/>


## Rendering

**Rendering sun light:**

```
float3 SunLight(float3 position, float3 PlanetCenter, float3 sunLightDirection, float3 sunLightColor, float scatterAngle)
{
	const float3 RandomUnitSphere[6] = 
	{
		{0.3f, -0.8f, -0.5f},
		{0.9f, -0.3f, -0.2f},
		{-0.9f, -0.3f, -0.1f},
		{-0.5f, 0.5f, 0.7f},
		{-1.0f, 0.3f, 0.0f},
		{-0.3f, 0.9f, 0.4f}
	};


	const int steps = 6;
	float sunRayStepLength = (Cloud Thickness * _sunLightRayStepScale) / ((float)steps);
	float3 sunRayStep = LightDirection * sunRayStepLength;
	pos += 0.5 * sunRayStep;

	float opticalDepth = 0.0;

	for (int i = 0; i < steps; i++)
	{
		float3 randomOffset = RandomUnitSphere[i] * sunRayStepLength * 0.25 * ((float)(i + 1));

		float3 samplePosition = position + randomOffset;

		float cloudPercentageHeight = calcCloudPercentageHeight(samplePosition, PlanetCenter);
		float3 weather = WeatherData(samplePosition);

		float cloudDensity = SampleCloud(samplePos, cloudPercentageHeight, weather);

		opticalDepth += cloudDensity * sunRayStepLength;

		pos += sunRayStep;

	}

	float HG = HenyeyGreenStein(scatteringAngle, g);

	return sunLightColor * BeerLambert(opticalDepth) * HG;
}
```
<br/>


**Rendering cloud:**

<img src="https://i.imgur.com/4PB5Inp.png" style="zoom:50%" /><br/>

By ray marching through spherical atmosphere we can ensure that clouds properly descend into the horizon.
<br/>



<img src="https://i.imgur.com/viXa1iw.png" style="zoom:47%" /><br/><br/>



**Step1.** Determine eye position under the clouds, within the clouds or above the clouds.

If you had some questions about" INTERSECTION OF A RAY WITH A SPHERE ",  please refer to the references[3], p.388.

<img src="https://i.imgur.com/W9ZX1sk.png" style="zoom:47%" /><br/><br/>



**Step2.** Calculate ray step length based on the distance from eye position to cloud.

```
rayStepLength = _CloudRayStepScale * (distanceToHitTopCloud - distanceToHitBottomCloud) / steps;
```
<br/>



**Step3.** Ray marching 

```
color = 0
opticalDepth = 0
rayStep = norm_rayDirection * rayStepLength      

//ray marching
for(int i = 0; i < execute Steps; i++)
{
	cloudPercentageHeight(position, planetCenter)
	
	weatherData(position)
	
	cloudDensity(position, cloud percentage height, weatherData)
	
	if (cloudDensity > 0)
	{
	ambientLight(cloudPercentageHeight)
	sunLight(position, PlanetCenter, sunLightDirection, sunLightColor, scatterAngle)
	currentOpticalDepth = cloudDensity * rayStepLength * scatterCoefficient
	ambientLight *= _AmbientLightIntensity * currentOpticalDepth;
	sunLight *= _SunLightIntensity * currentOpticalDepth;
	opticalDepth += currentOpticalDepth;
	color.rgb += (sunLight + ambientLight) * BeerLaw(opticalDepth);
	}
	position += rayStep
	color.a = 1.0 - BeerLaw(opticalDepth)
	horizonFade = 1 - saturate(distanceToBottomCloud/HorizonMaxDistance)
    color *= horizonFade
    
	return color
}
```
<br/>



## Demo

https://www.youtube.com/watch?v=XgJ4qaCpRDE
<br/>


## References

[1] [Nubis - Authoring Realtime Volumetric Cloudscapes with the Decima Engine - Final](http://advances.realtimerendering.com/s2017/Nubis%20-%20Authoring%20Realtime%20Volumetric%20Cloudscapes%20with%20the%20Decima%20Engine%20-%20Final%20.pdf) 

[2] [Siggraph15 Schneider Real-Time Volumetric Cloudscapes of Horizon Zero Dawn](http://killzone.dl.playstation.net/killzone/horizonzerodawn/presentations/Siggraph15_Schneider_Real-Time_Volumetric_Cloudscapes_of_Horizon_Zero_Dawn.pdf)

[3] [Graphics Gems（1998）P.388](http://inis.jinr.ru/sl/vol1/CMC/Graphics_Gems_1,ed_A.Glassner.pdf)

[4] [Real-Time Volumetric Rendering](http://patapom.com/topics/Revision2013/Revision%202013%20-%20Real-time%20Volumetric%20Rendering%20Course%20Notes.pdf)

[5] [kode80 Clouds for Unity3D](http://kode80.com/blog/2016/04/11/kode80-clouds-for-unity3d/index.html)

[6] [Gamedev.net](https://www.gamedev.net/forums/topic/680832-horizonzero-dawn-cloud-system/)

[7] [Production Volume RenderingSIGGRAPH 2017 Course](https://graphics.pixar.com/library/ProductionVolumeRendering/paper.pdf)

[8] [Simple Ray Marching Scene](https://zhuanlan.zhihu.com/p/53491836)

[9] [Tutorial - Ray marching framework](https://blog.csdn.net/tjw02241035621611/article/details/80057928)

[10] [Introduction : vertex and fragment shader in Unity](https://blog.csdn.net/candycat1992/article/details/40212735)
