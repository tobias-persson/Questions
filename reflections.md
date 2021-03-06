#Reprojecting reflections

Screen space reflections are such a pain. When combined with taa they are even harder to manage. Raytracing against jittered depth/normal g-buffers can easily cause reflection rays to have widely different intersection points from frame to frame. When using neighborhood clamping, it becomes difficult to handle the flickering caused by too much clipping caused by the high variance in the ssr signal. On top of this, reflections are very hard to reproject. Since they are view dependent simply fetching the motion vector from the current pixel tends to make the reprojection "smudge" under camera motion.

![](https://github.com/greje656/Questions/blob/master/images/foreground-original.png)

Last year I spent some time trying to understand this problem a little bit more. I first drew a ray diagram desicrbing how a reflection could be reprojected in theory:

1) Retrieve the surface motion vector (ms) corresponding to the reflection incidence point (v0)
2) Reproject the incidence point using (ms)
3) Using the previous depth buffer, reconstruct the reflection incidence point (v1)
4) Retrieve the motion vector (mr) corresponding to the reflected point (p0)
5) Reproject the reflection point using (mr)
6) Using the depth buffer history, reconstruct the previous reflection point (p1)
7) Using the previous view matrix transform, reconstruct the previous surface normal of the incidence point (n1)
8) Project the camera position (deye ) and the reconstructed reflection point (dp1) onto the previous plane (defined by surface normal = n1, and surface point = v1)
9) Solve for the position of the previous reflection point (r) knowing (deye ) and (dp1) 
10) Finally, using the previous view-projection matrix, evaluate (r) in the previous reflection buffer. 

![](https://github.com/greje656/Questions/blob/master/images/foreground-original.png)

By adding to Stingray a copy of the depth buffer of the previous frame and using the previous view projection matrix I was able to confirm this approach could successfully reproject reflections:

![](https://github.com/greje656/Questions/blob/master/images/foreground-original.png)

Ghosting was definitly minimised under camera motion (note that here neighborhood clamping is disabled to visualize the success of the reprojection better):

![](https://github.com/greje656/Questions/blob/master/images/foreground-original.png)

Unfortunately keeping a copy of the depthbuffer is currently not really a feasible/appealing solution in a lot of scenarios. But it was a good exercise to understand the problem.

So instead I tried a different approach. The new idea was to pick a few reprojection vectors that are likely to be meaningful in the context of a reflection. Looking at the case of camera rotation, translation, and object motion we can build three "interesting" vectors to consider:

We then declare the vector with the smallest magnitude as the most likely succesful reprojection vector. This simple idea alone has improved the reprojection of the ssr buffer quite significantly:

![Imgur](http://i.imgur.com/QURbYF0.gifv)

Note that if casting multiple rays per pixel then averaging the sum of all reprojection vectors still gave us a better results than what we had previously.

Screen space reflections is one of the most difficult screen space effect I've had to deal with. They are plaged with artifacts which can often be difficult to explain or understand.In the last couple of years I've seen people propose really creative ways to minimize some of the artifacts that are inherent to ssr. 

















![](https://github.com/greje656/Questions/blob/master/images/foreground-original.png)

The results I seem to get for the foreground layer is more like (note this is using a "background range" of 2.5m so ~100in).

foreground.rgb/foreground.a:
![](https://github.com/greje656/Questions/blob/master/images/foreground.jpg)
Notice how I don't get any samples pixels whose weight is low? I'm wondering if I miss-understood something crucial here (i.e. no black pixels like you're result). 

The weight of the samples looks like this:
![](https://github.com/greje656/Questions/blob/master/images/foreground-weights.jpg)

Similarly the background looks like this:

background.rgb/background.a
![](https://github.com/greje656/Questions/blob/master/images/background.jpg)

And the background weights:
![](https://github.com/greje656/Questions/blob/master/images/background-weights.jpg)

The normalized alpha value I get looks like:
![](https://github.com/greje656/Questions/blob/master/images/alpha.jpg)

And finally "lerp(background/background.a, foreground/foreground.a, alpha)":
![](https://github.com/greje656/Questions/blob/master/images/results.jpg)

I'm wondering maybe the foreground/background layers should be weighted like this? (... nah, that doesn't make sense! I'm so confused)

foreground.rgb/SAMPLE_COUNT:
![](https://github.com/greje656/Questions/blob/master/images/foreground2.jpg)

background.rgb/SAMPLE_COUNT:
![](https://github.com/greje656/Questions/blob/master/images/background2.jpg)

####Question 2)
Should the maxCoCMinDepth tiles be sampled using linear interpolation? Or point sampling? Intuitively it feels like it should be point sampling but I see artifacts at the edges of the tiles if I do. Note that with tiles of 20x20 pixels I'm making the assumption that the maximum kernel size allowed should be 10x10px. Maybe that is a false assumption?

Linear (currently what I'm using):
![](https://github.com/greje656/Questions/blob/master/images/tile-min-depth-linear.jpg)

Point:
![](https://github.com/greje656/Questions/blob/master/images/tile-min-depth-point.jpg)

Point sampling artifacts:
![](https://github.com/greje656/Questions/blob/master/images/artifacts.jpg)

####Questions 3)
Finally (this one might be really dumb!). Does this need to differentiate samples that are behind\infront of the camera focus point? i.e. does this need signed CoCs? I think it does but that's not something that's 100% clear to me. Currently I've implemented a solution that uses abs(coc), but this doesn't work for focused objects on top of unfocused:
![](https://github.com/greje656/Questions/blob/master/images/results-bad.jpg)
It's probably that we DO need to use a signed circle of confusions but I just wanted to confirm that I'm not missing something obvious here (I'm sure I am)

####Code used to generate images
	#define NUM_SAMPLES 5
	#define COC_SIZE_IN_PIXELS 10
	#define BACKGROUND_RANGE 2.5

	struct PresortParams {
		float coc;
		float backgroundWeight;
		float foregroundWeight;
	};
	
	float2 DepthCmp2(float depth, float closestTileDepth){
		float d = saturate((depth - closestTileDepth)/BACKGROUND_RANGE);
		float2 depthCmp;
		depthCmp.x = smoothstep( 0.0, 1.0, d ); // Background
		depthCmp.y = 1.0 - depthCmp.x; // Foreground
		return depthCmp;
	}
	
	float SampleAlpha(float sampleCoc) {
		const float DOF_SINGLE_PIXEL_RADIUS = length(float2(0.5, 0.5));
		return min(
			rcp(PI * sampleCoc * sampleCoc),
			rcp(PI * DOF_SINGLE_PIXEL_RADIUS * DOF_SINGLE_PIXEL_RADIUS)
		);
	}
	
	PresortParams GetPresortParams(float sample_coc, float sample_depth, float closestTileDepth){
		PresortParams presort_params;
	
		presort_params.coc = sample_coc;
		presort_params.backgroundWeight = SampleAlpha(sample_coc) * DepthCmp2(sample_depth, closestTileDepth).x;
		presort_params.foregroundWeight = SampleAlpha(sample_coc) * DepthCmp2(sample_depth, closestTileDepth).y;
	
		return presort_params;
	}
	
	float4 ps_main(PS_INPUT input) : SV_TARGET0 {
	
		float tileMaxCoc = TEX2DLOD(input_texture3, input.uv, 0).g * COC_SIZE_IN_PIXELS;
	
		float SAMPLE_COUNT = 0.0;
		float4 background = 0;
		float4 foreground = 0;
	
		for (int x = -NUM_SAMPLES; x <= NUM_SAMPLES; ++x) {
			for (int y = -NUM_SAMPLES; y <= NUM_SAMPLES; ++y) {
	
				float2 samplePos = get_sampling_pos(x, y);
				float2 sampleUV = input.uv + samplePos/back_buffer_size * tileMaxCoc;
	
				float3 sampleColor = TEX2DLOD(input_texture0, sampleUV, 0).rgb;
				float sampleCoc = decode_coc(TEX2DLOD(input_texture1, sampleUV, 0).r) * COC_SIZE_IN_PIXELS;
				float sampleDepth = linearize_depth(TEX2DLOD(input_texture2, sampleUV, 0).r);
				float closestTileDepth = TEX2DLOD(input_texture3, input.uv, 0).r;
	
				PresortParams samplePresortParams = GetPresortParams(sampleCoc, sampleDepth, closestTileDepth);
	
				background += samplePresortParams.backgroundWeight * float4(sampleColor, 1.f);
				foreground += samplePresortParams.foregroundWeight * float4(sampleColor, 1.f);
	
				SAMPLE_COUNT += 1;
			}
		}
		
		float alpha = saturate( 2.0 * ( 1.0 / SAMPLE_COUNT ) * ( 1.0 / SampleAlpha(tileMaxCoc)) * foreground.a);
		float3 finalColor = lerp(background/background.a, foreground/foreground.a, alpha);
	
		return float4(finalColor, 1);
	}