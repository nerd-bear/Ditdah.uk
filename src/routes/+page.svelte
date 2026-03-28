<script lang="ts">
	import { onMount } from 'svelte';

	type Mode = 'paddle' | 'straight';
	type ElementType = '.' | '-';
	type Side = 'dit' | 'dah';

	const MORSE_TO_TEXT: Record<string, string> = {
		'.-': 'A',
		'-...': 'B',
		'-.-.': 'C',
		'-..': 'D',
		'.': 'E',
		'..-.': 'F',
		'--.': 'G',
		'....': 'H',
		'..': 'I',
		'.---': 'J',
		'-.-': 'K',
		'.-..': 'L',
		'--': 'M',
		'-.': 'N',
		'---': 'O',
		'.--.': 'P',
		'--.-': 'Q',
		'.-.': 'R',
		'...': 'S',
		'-': 'T',
		'..-': 'U',
		'...-': 'V',
		'.--': 'W',
		'-..-': 'X',
		'-.--': 'Y',
		'--..': 'Z',
		'.----': '1',
		'..---': '2',
		'...--': '3',
		'....-': '4',
		'.....': '5',
		'-....': '6',
		'--...': '7',
		'---..': '8',
		'----.': '9',
		'-----': '0',
		'.-.-.-': '.',
		'--..--': ',',
		'..--..': '?',
		'.----.': "'",
		'-.-.--': '!',
		'-..-.': '/',
		'-.--.': '(',
		'-.--.-': ')',
		'.-...': '&',
		'---...': ':',
		'-.-.-.': ';',
		'-...-': '=',
		'.-.-.': '+',
		'-....-': '-',
		'..--.-': '_',
		'.-..-.': '"',
		'...-..-': '$',
		'.--.-.': '@'
	};

	let mode = $state<Mode>('paddle');
	let reversePaddles = $state(false);
	let wpm = $state(15);
	let toneHz = $state(650);
	let decodedText = $state('');
	let currentMorse = $state('');
	let listening = $state(true);
	let hasAudioStarted = $state(false);
	let status = $state('Ready');

	let pressedDit = $state(false);
	let pressedDah = $state(false);
	let activePhysicalKeys = new Set<string>();

	let audioContext: AudioContext | null = null;
	let gainNode: GainNode | null = null;
	let oscillator: OscillatorNode | null = null;

	let sending = false;
	let paddleLoopActive = false;
	let straightStartedAt = 0;
	let straightSide: Side | null = null;
	let toneTimeout: number | null = null;
	let elementBoundaryTimeout: number | null = null;
	let letterTimer: number | null = null;
	let wordTimer: number | null = null;

	const unitMs = $derived(Math.max(10, 1200 / Number(wpm || 18)));
	const ditLabel = $derived(reversePaddles ? 'Dah' : 'Dit');
	const dahLabel = $derived(reversePaddles ? 'Dit' : 'Dah');

	function keySignature(event: KeyboardEvent) {
		return `${event.code}:${event.location}:${event.key}`;
	}

	function matchesLeftCtrl(event: KeyboardEvent) {
		return (
			event.code === 'ControlLeft' ||
			(event.key === 'Control' && event.location === KeyboardEvent.DOM_KEY_LOCATION_LEFT)
		);
	}

	function matchesRightCtrl(event: KeyboardEvent) {
		return (
			event.code === 'ControlRight' ||
			(event.key === 'Control' && event.location === KeyboardEvent.DOM_KEY_LOCATION_RIGHT)
		);
	}

	function matchesLeftShift(event: KeyboardEvent) {
		return (
			event.code === 'ShiftLeft' ||
			(event.key === 'Shift' && event.location === KeyboardEvent.DOM_KEY_LOCATION_LEFT)
		);
	}

	function matchesRightShift(event: KeyboardEvent) {
		return (
			event.code === 'ShiftRight' ||
			(event.key === 'Shift' && event.location === KeyboardEvent.DOM_KEY_LOCATION_RIGHT)
		);
	}

	function matchesLeftBracketSide(event: KeyboardEvent) {
		return (
			event.code === 'BracketLeft' ||
			event.key === '[' ||
			event.key === '{' ||
			event.key === '<' ||
			event.code === 'Comma'
		);
	}

	function matchesRightBracketSide(event: KeyboardEvent) {
		return (
			event.code === 'BracketRight' ||
			event.key === ']' ||
			event.key === '}' ||
			event.key === '>' ||
			event.code === 'Period'
		);
	}

	function getSideFromEvent(event: KeyboardEvent): Side | null {
		const ditPhysical =
			matchesLeftCtrl(event) || matchesLeftShift(event) || matchesLeftBracketSide(event);
		const dahPhysical =
			matchesRightCtrl(event) || matchesRightShift(event) || matchesRightBracketSide(event);

		if (ditPhysical) return reversePaddles ? 'dah' : 'dit';
		if (dahPhysical) return reversePaddles ? 'dit' : 'dah';
		return null;
	}

	async function ensureAudio() {
		if (!audioContext) {
			audioContext = new AudioContext();
			gainNode = audioContext.createGain();
			oscillator = audioContext.createOscillator();

			oscillator.type = 'sine';
			oscillator.frequency.value = toneHz;
			gainNode.gain.value = 0.0001;

			oscillator.connect(gainNode);
			gainNode.connect(audioContext.destination);
			oscillator.start();
		}

		if (audioContext.state === 'suspended') {
			await audioContext.resume();
		}

		oscillator?.frequency.setTargetAtTime(toneHz, audioContext.currentTime, 0.003);
		hasAudioStarted = true;
	}

	function toneOn() {
		if (!audioContext || !gainNode || !oscillator) return;
		const now = audioContext.currentTime;
		oscillator.frequency.cancelScheduledValues(now);
		oscillator.frequency.setTargetAtTime(toneHz, now, 0.003);
		gainNode.gain.cancelScheduledValues(now);
		gainNode.gain.setValueAtTime(Math.max(0.0001, gainNode.gain.value), now);
		gainNode.gain.exponentialRampToValueAtTime(0.14, now + 0.008);
	}

	function toneOff() {
		if (!audioContext || !gainNode) return;
		const now = audioContext.currentTime;
		gainNode.gain.cancelScheduledValues(now);
		gainNode.gain.setValueAtTime(Math.max(0.0001, gainNode.gain.value), now);
		gainNode.gain.exponentialRampToValueAtTime(0.0001, now + 0.014);
	}

	function clearGapTimers() {
		if (letterTimer) {
			window.clearTimeout(letterTimer);
			letterTimer = null;
		}
		if (wordTimer) {
			window.clearTimeout(wordTimer);
			wordTimer = null;
		}
	}

	function appendDecodedCharacter() {
		if (!currentMorse) return;
		decodedText += MORSE_TO_TEXT[currentMorse] ?? '*';
		currentMorse = '';
	}

	function scheduleDecodingGaps() {
		clearGapTimers();

		letterTimer = window.setTimeout(() => {
			appendDecodedCharacter();
			status = 'Letter complete';
		}, unitMs * 3);

		wordTimer = window.setTimeout(() => {
			appendDecodedCharacter();
			if (decodedText && !decodedText.endsWith(' ')) decodedText += ' ';
			status = 'Word gap';
		}, unitMs * 7);
	}

	function pushElement(element: ElementType) {
		currentMorse += element;
		scheduleDecodingGaps();
		status = element === '.' ? 'Dit' : 'Dah';
	}

	function pickPaddleElement(): ElementType | null {
		if (pressedDit && pressedDah) {
			return reversePaddles ? '-' : '.';
		}
		if (pressedDit) return '.';
		if (pressedDah) return '-';
		return null;
	}

	async function sendElement(element: ElementType) {
		await ensureAudio();
		sending = true;
		const duration = element === '.' ? unitMs : unitMs * 3;

		clearGapTimers();
		toneOn();

		if (toneTimeout) window.clearTimeout(toneTimeout);
		if (elementBoundaryTimeout) window.clearTimeout(elementBoundaryTimeout);

		toneTimeout = window.setTimeout(() => {
			toneOff();
			pushElement(element);
			scheduleDecodingGaps();
		}, duration);

		elementBoundaryTimeout = window.setTimeout(() => {
			sending = false;
		}, duration + unitMs);

		await new Promise<void>((resolve) => {
			window.setTimeout(resolve, duration + unitMs);
		});
	}

	async function runPaddleLoop() {
		if (paddleLoopActive) return;
		paddleLoopActive = true;

		while (listening && (pressedDit || pressedDah)) {
			const element = pickPaddleElement();
			if (!element) break;
			await sendElement(element);
		}

		paddleLoopActive = false;
		status = 'Idle';
	}

	async function startStraight(side: Side) {
		await ensureAudio();
		straightSide = side;
		straightStartedAt = performance.now();
		toneOn();
		status = side === 'dit' ? 'Straight key down (dit side)' : 'Straight key down (dah side)';
	}

	function stopStraight() {
		if (!straightStartedAt || !straightSide) return;

		const heldMs = performance.now() - straightStartedAt;
		toneOff();

		let element: ElementType;
		if (heldMs <= unitMs * 1.8) {
			element = '.';
		} else if (heldMs >= unitMs * 2.2) {
			element = '-';
		} else {
			element = straightSide === 'dit' ? '.' : '-';
		}

		pushElement(element);
		straightStartedAt = 0;
		straightSide = null;
	}

	async function handlePress(side: Side) {
		if (!listening) return;

		if (mode === 'paddle') {
			if (side === 'dit') pressedDit = true;
			if (side === 'dah') pressedDah = true;
			await runPaddleLoop();
			return;
		}

		if (!straightSide) {
			await startStraight(side);
		}
	}

	function handleRelease(side: Side) {
		if (mode === 'paddle') {
			if (side === 'dit') pressedDit = false;
			if (side === 'dah') pressedDah = false;
			return;
		}

		if (straightSide === side) {
			stopStraight();
		}
	}

	async function onKeyDown(event: KeyboardEvent) {
		if (event.repeat) return;
		if (!listening) return;

		const target = event.target as HTMLElement | null;
		if (target && ['INPUT', 'TEXTAREA', 'SELECT'].includes(target.tagName)) return;

		const side = getSideFromEvent(event);
		if (!side) return;

		event.preventDefault();
		const signature = keySignature(event);
		if (activePhysicalKeys.has(signature)) return;
		activePhysicalKeys.add(signature);

		await handlePress(side);
	}

	function onKeyUp(event: KeyboardEvent) {
		const side = getSideFromEvent(event);
		if (!side) return;

		event.preventDefault();
		activePhysicalKeys.delete(keySignature(event));
		handleRelease(side);
	}

	function clearAll() {
		decodedText = '';
		currentMorse = '';
		clearGapTimers();
		status = 'Cleared';
	}

	function backspaceLast() {
		if (mode !== 'paddle') return;
		decodedText = decodedText.trimEnd().slice(0, -1);
		status = 'Removed last character';
	}

	function stopEverything() {
		pressedDit = false;
		pressedDah = false;
		activePhysicalKeys.clear();
		straightStartedAt = 0;
		straightSide = null;
		sending = false;
		paddleLoopActive = false;

		if (toneTimeout) {
			window.clearTimeout(toneTimeout);
			toneTimeout = null;
		}
		if (elementBoundaryTimeout) {
			window.clearTimeout(elementBoundaryTimeout);
			elementBoundaryTimeout = null;
		}

		toneOff();
	}

	onMount(() => {
		const down = (event: KeyboardEvent) => void onKeyDown(event);
		const up = (event: KeyboardEvent) => onKeyUp(event);
		const blur = () => stopEverything();

		window.addEventListener('keydown', down, { capture: true });
		window.addEventListener('keyup', up, { capture: true });
		window.addEventListener('blur', blur);

		return () => {
			window.removeEventListener('keydown', down, { capture: true });
			window.removeEventListener('keyup', up, { capture: true });
			window.removeEventListener('blur', blur);
			stopEverything();
			clearGapTimers();
			oscillator?.stop();
			oscillator?.disconnect();
			gainNode?.disconnect();
			audioContext?.close();
		};
	});
</script>

<svelte:head>
	<title>DitDah.uk | VBand Compatible Morse Code Practice Site</title>
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<meta
		name="description"
		content="VBand compatible, Morse code practice site with a responsive browser-based keyer supporting paddle and straight key modes, adjustable speed and tone, and real-time text decoding!"
	/>
	<meta
		name="keywords"
		content="Morse code practice, VBand compatible, browser keyer, CW trainer, paddle keyer, straight key, Morse code decoder, ham radio practice"
	/>
	<meta name="robots" content="index, follow" />
	<meta name="author" content="DitDah.uk" />
	<meta property="og:type" content="website" />
	<meta property="og:site_name" content="DitDah.uk" />
	<meta property="og:title" content="DitDah.uk | VBand Compatible Morse Code Practice Site" />
	<meta
		property="og:description"
		content="VBand compatible, Morse code practice site with a responsive browser-based keyer supporting paddle and straight key modes, adjustable speed and tone, and real-time text decoding!"
	/>
	<meta property="og:image" content="/og-image.jpg" />
	<meta property="og:image:alt" content="DitDah.uk Morse code practice interface" />
	<meta property="og:url" content="https://ditdah.uk/" />
	<meta name="twitter:card" content="summary_large_image" />
	<meta name="twitter:title" content="DitDah.uk | VBand Compatible Morse Code Practice Site" />
	<meta
		name="twitter:description"
		content="VBand compatible, Morse code practice site with a responsive browser-based keyer supporting paddle and straight key modes, adjustable speed and tone, and real-time text decoding!"
	/>
	<meta name="twitter:image" content="/og-image.jpg" />

	<link rel="icon" type="image/png" href="/favicon/favicon-96x96.png" sizes="96x96" />
	<link rel="icon" type="image/svg+xml" href="/favicon/favicon.svg" />
	<link rel="shortcut icon" href="/favicon/favicon.ico" />
	<link rel="apple-touch-icon" sizes="180x180" href="/favicon/apple-touch-icon.png" />
	<link rel="manifest" href="/favicon/site.webmanifest" />
</svelte:head>

<div class="min-h-screen bg-[#07080b] text-white antialiased">
	<main class="mx-auto flex min-h-screen max-w-[90%] flex-col px-6 py-12 sm:px-10 lg:px-14">
		<header class="mb-16">
			<div class="text-[2.8rem] font-semibold tracking-tight text-white/95">DITDAH.UK</div>
		</header>

		<section class="w-full">
			<div class="grid grid-cols-1 gap-8 sm:grid-cols-3">
				<label class="space-y-3">
					<span class="block text-center text-[1.05rem] text-white/90">Mode</span>
					<select
						bind:value={mode}
						class="w-full rounded-md border border-white/15 bg-white/90 px-3 py-2 text-lg text-black transition outline-none focus:border-white/40"
					>
						<option value="paddle">Paddle</option>
						<option value="straight">Straight</option>
					</select>
				</label>

				<label class="space-y-3">
					<span class="block text-center text-[1.05rem] text-white/90">WPM</span>
					<input bind:value={wpm} min="5" max="50" type="range" class="w-full accent-[#efd462]" />
					<div class="text-center text-sm text-white/55">{wpm} wpm</div>
				</label>

				<label class="space-y-3">
					<span class="block text-center text-[1.05rem] text-white/90">Tone</span>
					<input
						bind:value={toneHz}
						min="300"
						max="1200"
						step="10"
						type="range"
						class="w-full accent-[#79c6f2]"
					/>
					<div class="text-center text-sm text-white/55">{toneHz} hz</div>
				</label>
			</div>

			<div
				class="mt-14 rounded-[2rem] border border-white/6 bg-[linear-gradient(90deg,rgba(46,58,49,0.72),rgba(34,44,42,0.88))] p-7 shadow-[0_0_0_1px_rgba(255,255,255,0.02)] sm:p-8"
			>
				<div class="mb-5 flex flex-wrap items-start justify-between gap-4">
					<div>
						<div class="text-[1.05rem] text-white/95">Decoded Text</div>
						<div class="mt-2 text-sm text-white/45">
							{ditLabel}: Left Ctrl / Left Shift / [ / &lt; &nbsp;·&nbsp; {dahLabel}: Right Ctrl /
							Right Shift / ] / &gt;
						</div>
					</div>

					<div class="flex flex-wrap gap-2">
						<button
							class="rounded-full border border-white/10 bg-white/5 px-3 py-1.5 text-sm text-white/85 transition hover:bg-white/10"
							onclick={clearAll}
						>
							Clear
						</button>
						<button
							class="rounded-full border border-white/10 bg-white/5 px-3 py-1.5 text-sm text-white/85 transition hover:bg-white/10 disabled:opacity-35"
							onclick={() => (reversePaddles = !reversePaddles)}
							disabled={mode !== 'paddle'}
						>
							Reverse
						</button>
						<button
							class="rounded-full border border-white/10 bg-white/5 px-3 py-1.5 text-sm text-white/85 transition hover:bg-white/10"
							onclick={() => (listening = !listening)}
						>
							{listening ? 'Listening' : 'Paused'}
						</button>
					</div>
				</div>

				<textarea
					readonly
					value={decodedText}
					class="min-h-[16rem] w-full resize-none bg-transparent font-mono text-lg tracking-wide text-white/90 outline-none placeholder:text-white/20"
					placeholder="Start keying..."
				/>

				<div
					class="mt-6 flex flex-wrap items-center justify-between gap-4 border-t border-white/8 pt-4 text-sm text-white/50"
				>
					<div>Status: {status}</div>
					<div>Current: <span class="font-mono text-white/70">{currentMorse || '—'}</span></div>
					<div>{hasAudioStarted ? 'Audio ready' : 'Audio arms on first key press'}</div>
				</div>
			</div>

			<div class="mt-5 flex flex-wrap gap-3 text-sm text-white/35">
				<div class="rounded-full border border-white/8 px-3 py-1.5">Minimal trainer</div>
				<div class="rounded-full border border-white/8 px-3 py-1.5">Browser keyer input</div>
				<div class="rounded-full border border-white/8 px-3 py-1.5">No pop tone envelope</div>
			</div>
		</section>

		<div class="mt-2 text-center text-white/35">Made by Iona a SWL (M6PXO in a few days)</div>
	</main>
</div>
