<script lang="ts">
	import { onMount } from 'svelte';
	import { T } from '@threlte/core';
	import { AudioListener, PositionalAudio } from '@threlte/extras';
	import { OrbitControls } from '@threlte/extras';
	import { TextureLoader, Vector3, BufferGeometry } from 'three';

	import track from '$lib/audio/track.mp3';
	import { Canvas } from '@threlte/core';

	import { Align, Environment, Float, Text3DGeometry, Stars } from '@threlte/extras';
	import { Audio } from '@threlte/extras';
	import { Volume } from 'three/examples/jsm/Addons.js';

	let audio: PositionalAudio; // Reference to the audio object
	let isPlaying = false; // Track the playback state
	let coverOpen = false; // Simulate a cover open/close state for interaction

	let audioElement; // Reference to the audio element
	let analyser, frequencyData;
	let low = 0,
		mid = 0,
		high = 0,
		volume = 0;

	let minInput = 0;
	let maxInput = 252;
	let minOutput = 0.05;
	let maxOutput = 0.01;

	// Variables for updating points
	let vxFactor1 = 0.8;
	let vxFactor2 = 0.9;
	let vxMultiplier = 10;
	let vxRandomFactor = 0.1;
	let vyFactor1 = 0.8;
	let vyFactor2 = 0.3;
	let vyMultiplier = 10;
	let vyRandomFactor = 0.1;
	let vzRandomFactor = 0.01;

	const size = 22;
	const count = size ** 3;

	const vectorPositions: Vector3[] = [];
	for (let i = 0; i < count; i++) {
		let x = i / (size * size);
		let y = (i / size) % size;
		let z = i % size;

		const vx =
			Math.sin(Math.abs(size - x) * vxFactor1) *
				Math.sin(Math.abs(size - y) * vxFactor2) *
				vxMultiplier +
			Math.random() * vxRandomFactor;

		const vy =
			Math.sin(Math.abs(size - x) * vyFactor1) *
				Math.sin(Math.abs(size - y) * vyFactor2) *
				vyMultiplier +
			Math.random() * vyRandomFactor;

		const vz = y + Math.random() * vzRandomFactor * z;

		vectorPositions.push(new Vector3(vx, vy, vz));
	}
	const pointsBufferGeometry = new BufferGeometry().setFromPoints(vectorPositions);

	// Update the points dynamically
	function updatePointsGeometry() {
		const newVectorPositions: Vector3[] = [];

		const normalizedLow = 1 + ((low - minInput) / (maxInput - minInput)) * 9;
		const normalizedMid = 1 + ((mid - minInput) / (maxInput - minInput)) * 9;
		const normalizeHigh = 1 + ((high - minInput) / (maxInput - minInput)) * 9;
		const normalizeVolume = 1 + ((volume - minInput) / (maxInput - minInput)) * 9;

		const Low = Math.log10(normalizedLow);
		const Mid = Math.log10(normalizedMid);
		const High = Math.log10(normalizeHigh);
		const Volume = Math.log10(normalizeVolume);

		for (let i = 0; i < count; i++) {
			let x = i / (size * size);
			let y = (i / size) % size;
			let z = i % size;

			const vx =
				Math.sin(Math.abs(size - x + (Volume / 1) * Low * 3) * vxFactor1) *
					Math.sin(Math.abs(size - y) * vxFactor2) *
					vxMultiplier +
				Math.random() * vxRandomFactor;

			const vy =
				Math.sin(Math.abs(size - x + Volume * High) * vyFactor1) *
					Math.sin(Math.abs(size - y) * vyFactor2) *
					vyMultiplier +
				Math.random() * vyRandomFactor +
				High;

			const vz = y + Math.random() * vzRandomFactor * z;
			newVectorPositions.push(new Vector3(vx, vy, vz));
		}
		pointsBufferGeometry.setFromPoints(newVectorPositions);
	}

	// Update properties of points over time
	function updatePointsProperties() {
		vxFactor2 += 0.001;
	}

	// Utility to calculate average intensity in a frequency range
	function getFrequencyRange(minFreq, maxFreq) {
		const nyquist = 22050; // Half the standard sampling rate (44100 Hz)
		const startBin = Math.floor((minFreq / nyquist) * analyser.frequencyBinCount);
		const endBin = Math.floor((maxFreq / nyquist) * analyser.frequencyBinCount);

		let sum = 0;
		let count = 0;
		for (let i = startBin; i <= endBin; i++) {
			sum += frequencyData[i];
			count++;
		}
		return sum / count || 0; // Return average intensity
	}

	// Utility to calculate overall volume
	function getVolume() {
		let sum = 0;
		for (let i = 0; i < frequencyData.length; i++) {
			sum += frequencyData[i] * frequencyData[i]; // Square the values for RMS
		}
		return Math.sqrt(sum / frequencyData.length); // Return RMS volume
	}

	function startAnalysis() {
		function analyze() {
			if (analyser) {
				analyser.getByteFrequencyData(frequencyData);

				// Calculate frequency ranges
				low = getFrequencyRange(20, 250);
				mid = getFrequencyRange(250, 2000);
				high = getFrequencyRange(2000, 20000);
				volume = getVolume();

				console.log(low, mid, high, volume);

				requestAnimationFrame(analyze);
			}
		}
	}

	function analyze() {
		if (analyser) {
			analyser.getByteFrequencyData(frequencyData);

			// Calculate frequency ranges
			low = getFrequencyRange(20, 250);
			mid = getFrequencyRange(250, 2000);
			high = getFrequencyRange(2000, 20000);
			volume = getVolume();

			console.log(low, mid, high, volume);

			requestAnimationFrame(analyze);
		}
	}
	// Loop to change variables over time
	onMount(() => {
		if (audioElement) {
			const audioContext = new AudioContext();
			const audioSource = audioContext.createMediaElementSource(audioElement);

			// Create an AnalyserNode
			analyser = audioContext.createAnalyser();
			analyser.fftSize = 512; // Determines frequency resolution
			frequencyData = new Uint8Array(analyser.frequencyBinCount);

			// Connect the audio source to the analyser and the audio destination
			audioSource.connect(analyser);
			analyser.connect(audioContext.destination);

			// Start analyzing the audio
			startAnalysis();
		}
		const interval1 = setInterval(analyze, 32);
		const interval2 = setInterval(updatePointsGeometry, 32);
		return () => {
			clearInterval(interval1);
			clearInterval(interval2);
		};
	});
</script>

<div class="absolute h-full w-full object-cover">
	<Canvas>
		<T.DirectionalLight position={[0, 0, 0]} intensity={1} />

		<T.PerspectiveCamera
			makeDefault
			position.y={1}
			position.z={8}
			fov={90}
			on:create={({ ref }) => ref.lookAt(0, 0, 0)}
		>
			<OrbitControls enableDamping enablePan={false} enableZoom={false} autoRotate />
			<AudioListener />
		</T.PerspectiveCamera>

		<T.DirectionalLight position.y={10} position.z={10} />

		<Align>
			<T.Points>
				<T is={pointsBufferGeometry} />
				<T.PointsMaterial size={0.25} />
			</T.Points>
		</Align>
	</Canvas>
</div>

<div class="flex justify-between w-full">
	<div class="audio-container z-40 mx-auto bg-[#0d1320] p-6">
		<div class="w-full flex flex-col gap-2">
			<div class="font-bold w-full text-white">
				<!-- svelte-ignore a11y-missing-content -->
				<a
					href="https://soundcloud.com/l70ka5/physical-form?si=51e1a15f21bd409bb8facf382af472c2&utm_source=clipboard&utm_medium=text&utm_campaign=social_sharing"
					target="_blank"
				>
					L70KA5 - Physical Form
				</a>
			</div>
			<audio bind:this={audioElement} controls autoplay loop>
				<source src={track} type="audio/mp3" class="w-72" />
				Your browser does not support the audio element.
			</audio>
		</div>
	</div>
	<div class="z-50 audio-stats grid grid-cols-1 gap-2">
		<p><strong>Low:</strong> {low.toFixed(2)}</p>
		<p><strong>Mid:</strong> {mid.toFixed(2)}</p>
		<p><strong>High:</strong> {high.toFixed(2)}</p>
		<p><strong>Volume:</strong> {volume.toFixed(2)}</p>
	</div>
</div>

<style>
	.audio-container {
		display: fixed;
		flex-direction: column;
		align-items: center;
		margin: 1rem;
	}

	.audio-stats {
		margin-top: 1rem;
		font-family: Arial, sans-serif;
		text-align: center;
	}
</style>
